# hello-world

Hi Humans!

a redi gou

#!/usr/bin/perl

#
# Markdown -- A text-to-HTML conversion tool for web writers
#
# Copyright (c) 2004 John Gruber
# <http://daringfireball.net/projects/markdown/>
#


package Markdown;
require 5.006_000;
use strict;
use warnings;

use Digest::MD5 qw(md5_hex);
use vars qw($VERSION);
$VERSION = '1.0.1';
# Tue 14 Dec 2004

## Disabled; causes problems under Perl 5.6.1:
# use utf8;
# binmode( STDOUT, ":utf8" );  # c.f.: http://acis.openlib.org/dev/perl-unicode-struggle.html


#
# Global default settings:
#
my $g_empty_element_suffix = " />";     # Change to ">" for HTML output
my $g_tab_width = 4;


#
# Globals:
#

# Regex to match balanced [brackets]. See Friedl's
# "Mastering Regular Expressions", 2nd Ed., pp. 328-331.
my $g_nested_brackets;
$g_nested_brackets = qr{
	(?> 								# Atomic matching
	   [^\[\]]+							# Anything other than brackets
	 | 
	   \[
		 (??{ $g_nested_brackets })		# Recursive set of nested brackets
	   \]
	)*
}x;


# Table of hash values for escaped characters:
my %g_escape_table;
foreach my $char (split //, '\\`*_{}[]()>#+-.!') {
	$g_escape_table{$char} = md5_hex($char);
}


# Global hashes, used by various utility routines
my %g_urls;
my %g_titles;
my %g_html_blocks;

# Used to track when we're inside an ordered or unordered list
# (see _ProcessListItems() for details):
my $g_list_level = 0;


#### Blosxom plug-in interface ##########################################

# Set $g_blosxom_use_meta to 1 to use Blosxom's meta plug-in to determine
# which posts Markdown should process, using a "meta-markup: markdown"
# header. If it's set to 0 (the default), Markdown will process all
# entries.
my $g_blosxom_use_meta = 0;

sub start { 1; }
sub story {
	my($pkg, $path, $filename, $story_ref, $title_ref, $body_ref) = @_;

	if ( (! $g_blosxom_use_meta) or
	     (defined($meta::markup) and ($meta::markup =~ /^\s*markdown\s*$/i))
	     ){
			$$body_ref  = Markdown($$body_ref);
     }
     1;
}


#### Movable Type plug-in interface #####################################
eval {require MT};  # Test to see if we're running in MT.
unless ($@) {
    require MT;
    import  MT;
    require MT::Template::Context;
    import  MT::Template::Context;

	eval {require MT::Plugin};  # Test to see if we're running >= MT 3.0.
	unless ($@) {
		require MT::Plugin;
		import  MT::Plugin;
		my $plugin = new MT::Plugin({
			name => "Markdown",
			description => "A plain-text-to-HTML formatting plugin. (Version: $VERSION)",
			doc_link => 'http://daringfireball.net/projects/markdown/'
		});
		MT->add_plugin( $plugin );
	}

	MT::Template::Context->add_container_tag(MarkdownOptions => sub {
		my $ctx	 = shift;
		my $args = shift;
		my $builder = $ctx->stash('builder');
		my $tokens = $ctx->stash('tokens');

		if (defined ($args->{'output'}) ) {
			$ctx->stash('markdown_output', lc $args->{'output'});
		}

		defined (my $str = $builder->build($ctx, $tokens) )
			or return $ctx->error($builder->errstr);
		$str;		# return value
	});

	MT->add_text_filter('markdown' => {
		label     => 'Markdown',
		docs      => 'http://daringfireball.net/projects/markdown/',
		on_format => sub {
			my $text = shift;
			my $ctx  = shift;
			my $raw  = 0;
		    if (defined $ctx) {
		    	my $output = $ctx->stash('markdown_output'); 
				if (defined $output  &&  $output =~ m/^html/i) {
					$g_empty_element_suffix = ">";
					$ctx->stash('markdown_output', '');
				}
				elsif (defined $output  &&  $output eq 'raw') {
					$raw = 1;
					$ctx->stash('markdown_output', '');
				}
				else {
					$raw = 0;
					$g_empty_element_suffix = " />";
				}
			}
			$text = $raw ? $text : Markdown($text);
			$text;
		},
	});

	# If SmartyPants is loaded, add a combo Markdown/SmartyPants text filter:
	my $smartypants;

	{
		no warnings "once";
		$smartypants = $MT::Template::Context::Global_filters{'smarty_pants'};
	}

	if ($smartypants) {
		MT->add_text_filter('markdown_with_smartypants' => {
			label     => 'Markdown With SmartyPants',
			docs      => 'http://daringfireball.net/projects/markdown/',
			on_format => sub {
				my $text = shift;
				my $ctx  = shift;
				if (defined $ctx) {
					my $output = $ctx->stash('markdown_output'); 
					if (defined $output  &&  $output eq 'html') {
						$g_empty_element_suffix = ">";
					}
					else {
						$g_empty_element_suffix = " />";
					}
				}
				$text = Markdown($text);
				$text = $smartypants->($text, '1');
			},
		});
	}
}
else {
#### BBEdit/command-line text filter interface ##########################
# Needs to be hidden from MT (and Blosxom when running in static mode).

    # We're only using $blosxom::version once; tell Perl not to warn us:
	no warnings 'once';
    unless ( defined($blosxom::version) ) {
		use warnings;

		#### Check for command-line switches: #################
		my %cli_opts;
		use Getopt::Long;
		Getopt::Long::Configure('pass_through');
		GetOptions(\%cli_opts,
			'version',
			'shortversion',
			'html4tags',
		);
		if ($cli_opts{'version'}) {		# Version info
			print "\nThis is Markdown, version $VERSION.\n";
			print "Copyright 2004 John Gruber\n";
			print "http://daringfireball.net/projects/markdown/\n\n";
			exit 0;
		}
		if ($cli_opts{'shortversion'}) {		# Just the version number string.
			print $VERSION;
			exit 0;
		}
		if ($cli_opts{'html4tags'}) {			# Use HTML tag style instead of XHTML
			$g_empty_element_suffix = ">";
		}


		#### Process incoming text: ###########################
		my $text;
		{
			local $/;               # Slurp the whole file
			$text = <>;
		}
        print Markdown($text);
    }
}



sub Markdown {
#
# Main function. The order in which other subs are called here is
# essential. Link and image substitutions need to happen before
# _EscapeSpecialChars(), so that any *'s or _'s in the <a>
# and <img> tags get encoded.
#
	my $text = shift;

	# Clear the global hashes. If we don't clear these, you get conflicts
	# from other articles when generating a page which contains more than
	# one article (e.g. an index page that shows the N most recent
	# articles):
	%g_urls = ();
	%g_titles = ();
	%g_html_blocks = ();


	# Standardize line endings:
	$text =~ s{\r\n}{\n}g; 	# DOS to Unix
	$text =~ s{\r}{\n}g; 	# Mac to Unix

	# Make sure $text ends with a couple of newlines:
	$text .= "\n\n";

	# Convert all tabs to spaces.
	$text = _Detab($text);

	# Strip any lines consisting only of spaces and tabs.
	# This makes subsequent regexen easier to write, because we can
	# match consecutive blank lines with /\n+/ instead of something
	# contorted like /[ \t]*\n+/ .
	$text =~ s/^[ \t]+$//mg;

	# Turn block-level HTML blocks into hash entries
	$text = _HashHTMLBlocks($text);

	# Strip link definitions, store in hashes.
	$text = _StripLinkDefinitions($text);

	$text = _RunBlockGamut($text);

	$text = _UnescapeSpecialChars($text);

	return $text . "\n";
}


sub _StripLinkDefinitions {
#
# Strips link definitions from text, stores the URLs and titles in
# hash references.
#
	my $text = shift;
	my $less_than_tab = $g_tab_width - 1;

	# Link defs are in the form: ^[id]: url "optional title"
	while ($text =~ s{
						^[ ]{0,$less_than_tab}\[(.+)\]:	# id = $1
						  [ \t]*
						  \n?				# maybe *one* newline
						  [ \t]*
						<?(\S+?)>?			# url = $2
						  [ \t]*
						  \n?				# maybe one newline
						  [ \t]*
						(?:
							(?<=\s)			# lookbehind for whitespace
							["(]
							(.+?)			# title = $3
							[")]
							[ \t]*
						)?	# title is optional
						(?:\n+|\Z)
					}
					{}mx) {
		$g_urls{lc $1} = _EncodeAmpsAndAngles( $2 );	# Link IDs are case-insensitive
		if ($3) {
			$g_titles{lc $1} = $3;
			$g_titles{lc $1} =~ s/"/&quot;/g;
		}
	}

	return $text;
}


sub _HashHTMLBlocks {
	my $text = shift;
	my $less_than_tab = $g_tab_width - 1;

	# Hashify HTML blocks:
	# We only want to do this for block-level HTML tags, such as headers,
	# lists, and tables. That's because we still want to wrap <p>s around
	# "paragraphs" that are wrapped in non-block-level tags, such as anchors,
	# phrase emphasis, and spans. The list of tags we're looking for is
	# hard-coded:
	my $block_tags_a = qr/p|div|h[1-6]|blockquote|pre|table|dl|ol|ul|script|noscript|form|fieldset|iframe|math|ins|del/;
	my $block_tags_b = qr/p|div|h[1-6]|blockquote|pre|table|dl|ol|ul|script|noscript|form|fieldset|iframe|math/;

	# First, look for nested blocks, e.g.:
	# 	<div>
	# 		<div>
	# 		tags for inner block must be indented.
	# 		</div>
	# 	</div>
	#
	# The outermost tags must start at the left margin for this to match, and
	# the inner nested divs must be indented.
	# We need to do this before the next, more liberal match, because the next
	# match will start at the first `<div>` and stop at the first `</div>`.
	$text =~ s{
				(						# save in $1
					^					# start of line  (with /m)
					<($block_tags_a)	# start tag = $2
					\b					# word break
					(.*\n)*?			# any number of lines, minimally matching
					</\2>				# the matching end tag
					[ \t]*				# trailing spaces/tabs
					(?=\n+|\Z)	# followed by a newline or end of document
				)
			}{
				my $key = md5_hex($1);
				$g_html_blocks{$key} = $1;
				"\n\n" . $key . "\n\n";
			}egmx;


	#
	# Now match more liberally, simply from `\n<tag>` to `</tag>\n`
	#
	$text =~ s{
				(						# save in $1
					^					# start of line  (with /m)
					<($block_tags_b)	# start tag = $2
					\b					# word break
					(.*\n)*?			# any number of lines, minimally matching
					.*</\2>				# the matching end tag
					[ \t]*				# trailing spaces/tabs
					(?=\n+|\Z)	# followed by a newline or end of document
				)
			}{
				my $key = md5_hex($1);
				$g_html_blocks{$key} = $1;
				"\n\n" . $key . "\n\n";
			}egmx;
	# Special case just for <hr />. It was easier to make a special case than
	# to make the other regex more complicated.	
	$text =~ s{
				(?:
					(?<=\n\n)		# Starting after a blank line
					|				# or
					\A\n?			# the beginning of the doc
				)
				(						# save in $1
					[ ]{0,$less_than_tab}
					<(hr)				# start tag = $2
					\b					# word break
					([^<>])*?			# 
					/?>					# the matching end tag
					[ \t]*
					(?=\n{2,}|\Z)		# followed by a blank line or end of document
				)
			}{
				my $key = md5_hex($1);
				$g_html_blocks{$key} = $1;
				"\n\n" . $key . "\n\n";
			}egx;

	# Special case for standalone HTML comments:
	$text =~ s{
				(?:
					(?<=\n\n)		# Starting after a blank line
					|				# or
					\A\n?			# the beginning of the doc
				)
				(						# save in $1
					[ ]{0,$less_than_tab}
					(?s:
						<!
						(--.*?--\s*)+
						>
					)
					[ \t]*
					(?=\n{2,}|\Z)		# followed by a blank line or end of document
				)
			}{
				my $key = md5_hex($1);
				$g_html_blocks{$key} = $1;
				"\n\n" . $key . "\n\n";
			}egx;


	return $text;
}


sub _RunBlockGamut {
#
# These are all the transformations that form block-level
# tags like paragraphs, headers, and list items.
#
	my $text = shift;

	$text = _DoHeaders($text);

	# Do Horizontal Rules:
	$text =~ s{^[ ]{0,2}([ ]?\*[ ]?){3,}[ \t]*$}{\n<hr$g_empty_element_suffix\n}gmx;
	$text =~ s{^[ ]{0,2}([ ]? -[ ]?){3,}[ \t]*$}{\n<hr$g_empty_element_suffix\n}gmx;
	$text =~ s{^[ ]{0,2}([ ]? _[ ]?){3,}[ \t]*$}{\n<hr$g_empty_element_suffix\n}gmx;

	$text = _DoLists($text);

	$text = _DoCodeBlocks($text);

	$text = _DoBlockQuotes($text);

	# We already ran _HashHTMLBlocks() before, in Markdown(), but that
	# was to escape raw HTML in the original Markdown source. This time,
	# we're escaping the markup we've just created, so that we don't wrap
	# <p> tags around block-level tags.
	$text = _HashHTMLBlocks($text);

	$text = _FormParagraphs($text);

	return $text;
}


sub _RunSpanGamut {
#
# These are all the transformations that occur *within* block-level
# tags like paragraphs, headers, and list items.
#
	my $text = shift;

	$text = _DoCodeSpans($text);

	$text = _EscapeSpecialChars($text);

	# Process anchor and image tags. Images must come first,
	# because ![foo][f] looks like an anchor.
	$text = _DoImages($text);
	$text = _DoAnchors($text);

	# Make links out of things like `<http://example.com/>`
	# Must come after _DoAnchors(), because you can use < and >
	# delimiters in inline links like [this](<url>).
	$text = _DoAutoLinks($text);

	$text = _EncodeAmpsAndAngles($text);

	$text = _DoItalicsAndBold($text);

	# Do hard breaks:
	$text =~ s/ {2,}\n/ <br$g_empty_element_suffix\n/g;

	return $text;
}


sub _EscapeSpecialChars {
	my $text = shift;
	my $tokens ||= _TokenizeHTML($text);

	$text = '';   # rebuild $text from the tokens
# 	my $in_pre = 0;	 # Keep track of when we're inside <pre> or <code> tags.
# 	my $tags_to_skip = qr!<(/?)(?:pre|code|kbd|script|math)[\s>]!;

	foreach my $cur_token (@$tokens) {
		if ($cur_token->[0] eq "tag") {
			# Within tags, encode * and _ so they don't conflict
			# with their use in Markdown for italics and strong.
			# We're replacing each such character with its
			# corresponding MD5 checksum value; this is likely
			# overkill, but it should prevent us from colliding
			# with the escape values by accident.
			$cur_token->[1] =~  s! \* !$g_escape_table{'*'}!gx;
			$cur_token->[1] =~  s! _  !$g_escape_table{'_'}!gx;
			$text .= $cur_token->[1];
		} else {
			my $t = $cur_token->[1];
			$t = _EncodeBackslashEscapes($t);
			$text .= $t;
		}
	}
	return $text;
}


sub _DoAnchors {
#
# Turn Markdown link shortcuts into XHTML <a> tags.
#
	my $text = shift;

	#
	# First, handle reference-style links: [link text] [id]
	#
	$text =~ s{
		(					# wrap whole match in $1
		  \[
		    ($g_nested_brackets)	# link text = $2
		  \]

		  [ ]?				# one optional space
		  (?:\n[ ]*)?		# one optional newline followed by spaces

		  \[
		    (.*?)		# id = $3
		  \]
		)
	}{
		my $result;
		my $whole_match = $1;
		my $link_text   = $2;
		my $link_id     = lc $3;

		if ($link_id eq "") {
			$link_id = lc $link_text;     # for shortcut links like [this][].
		}

		if (defined $g_urls{$link_id}) {
			my $url = $g_urls{$link_id};
			$url =~ s! \* !$g_escape_table{'*'}!gx;		# We've got to encode these to avoid
			$url =~ s!  _ !$g_escape_table{'_'}!gx;		# conflicting with italics/bold.
			$result = "<a href=\"$url\"";
			if ( defined $g_titles{$link_id} ) {
				my $title = $g_titles{$link_id};
				$title =~ s! \* !$g_escape_table{'*'}!gx;
				$title =~ s!  _ !$g_escape_table{'_'}!gx;
				$result .=  " title=\"$title\"";
			}
			$result .= ">$link_text</a>";
		}
		else {
			$result = $whole_match;
		}
		$result;
	}xsge;

	#
	# Next, inline-style links: [link text](url "optional title")
	#
	$text =~ s{
		(				# wrap whole match in $1
		  \[
		    ($g_nested_brackets)	# link text = $2
		  \]
		  \(			# literal paren
		  	[ \t]*
			<?(.*?)>?	# href = $3
		  	[ \t]*
			(			# $4
			  (['"])	# quote char = $5
			  (.*?)		# Title = $6
			  \5		# matching quote
			)?			# title is optional
		  \)
		)
	}{
		my $result;
		my $whole_match = $1;
		my $link_text   = $2;
		my $url	  		= $3;
		my $title		= $6;

		$url =~ s! \* !$g_escape_table{'*'}!gx;		# We've got to encode these to avoid
		$url =~ s!  _ !$g_escape_table{'_'}!gx;		# conflicting with italics/bold.
		$result = "<a href=\"$url\"";

		if (defined $title) {
			$title =~ s/"/&quot;/g;
			$title =~ s! \* !$g_escape_table{'*'}!gx;
			$title =~ s!  _ !$g_escape_table{'_'}!gx;
			$result .=  " title=\"$title\"";
		}

		$result .= ">$link_text</a>";

		$result;
	}xsge;

	return $text;
}


sub _DoImages {
#
# Turn Markdown image shortcuts into <img> tags.
#
	my $text = shift;

	#
	# First, handle reference-style labeled images: ![alt text][id]
	#
	$text =~ s{
		(				# wrap whole match in $1
		  !\[
		    (.*?)		# alt text = $2
		  \]

		  [ ]?				# one optional space
		  (?:\n[ ]*)?		# one optional newline followed by spaces

		  \[
		    (.*?)		# id = $3
		  \]

		)
	}{
		my $result;
		my $whole_match = $1;
		my $alt_text    = $2;
		my $link_id     = lc $3;

		if ($link_id eq "") {
			$link_id = lc $alt_text;     # for shortcut links like ![this][].
		}

		$alt_text =~ s/"/&quot;/g;
		if (defined $g_urls{$link_id}) {
			my $url = $g_urls{$link_id};
			$url =~ s! \* !$g_escape_table{'*'}!gx;		# We've got to encode these to avoid
			$url =~ s!  _ !$g_escape_table{'_'}!gx;		# conflicting with italics/bold.
			$result = "<img src=\"$url\" alt=\"$alt_text\"";
			if (defined $g_titles{$link_id}) {
				my $title = $g_titles{$link_id};
				$title =~ s! \* !$g_escape_table{'*'}!gx;
				$title =~ s!  _ !$g_escape_table{'_'}!gx;
				$result .=  " title=\"$title\"";
			}
			$result .= $g_empty_element_suffix;
		}
		else {
			# If there's no such link ID, leave intact:
			$result = $whole_match;
		}

		$result;
	}xsge;

	#
	# Next, handle inline images:  ![alt text](url "optional title")
	# Don't forget: encode * and _

	$text =~ s{
		(				# wrap whole match in $1
		  !\[
		    (.*?)		# alt text = $2
		  \]
		  \(			# literal paren
		  	[ \t]*
			<?(\S+?)>?	# src url = $3
		  	[ \t]*
			(			# $4
			  (['"])	# quote char = $5
			  (.*?)		# title = $6
			  \5		# matching quote
			  [ \t]*
			)?			# title is optional
		  \)
		)
	}{
		my $result;
		my $whole_match = $1;
		my $alt_text    = $2;
		my $url	  		= $3;
		my $title		= '';
		if (defined($6)) {
			$title		= $6;
		}

		$alt_text =~ s/"/&quot;/g;
		$title    =~ s/"/&quot;/g;
		$url =~ s! \* !$g_escape_table{'*'}!gx;		# We've got to encode these to avoid
		$url =~ s!  _ !$g_escape_table{'_'}!gx;		# conflicting with italics/bold.
		$result = "<img src=\"$url\" alt=\"$alt_text\"";
		if (defined $title) {
			$title =~ s! \* !$g_escape_table{'*'}!gx;
			$title =~ s!  _ !$g_escape_table{'_'}!gx;
			$result .=  " title=\"$title\"";
		}
		$result .= $g_empty_element_suffix;

		$result;
	}xsge;

	return $text;
}


sub _DoHeaders {
	my $text = shift;

	# Setext-style headers:
	#	  Header 1
	#	  ========
	#  
	#	  Header 2
	#	  --------
	#
	$text =~ s{ ^(.+)[ \t]*\n=+[ \t]*\n+ }{
		"<h1>"  .  _RunSpanGamut($1)  .  "</h1>\n\n";
	}egmx;

	$text =~ s{ ^(.+)[ \t]*\n-+[ \t]*\n+ }{
		"<h2>"  .  _RunSpanGamut($1)  .  "</h2>\n\n";
	}egmx;


	# atx-style headers:
	#	# Header 1
	#	## Header 2
	#	## Header 2 with closing hashes ##
	#	...
	#	###### Header 6
	#
	$text =~ s{
			^(\#{1,6})	# $1 = string of #'s
			[ \t]*
			(.+?)		# $2 = Header text
			[ \t]*
			\#*			# optional closing #'s (not counted)
			\n+
		}{
			my $h_level = length($1);
			"<h$h_level>"  .  _RunSpanGamut($2)  .  "</h$h_level>\n\n";
		}egmx;

	return $text;
}


sub _DoLists {
#
# Form HTML ordered (numbered) and unordered (bulleted) lists.
#
	my $text = shift;
	my $less_than_tab = $g_tab_width - 1;

	# Re-usable patterns to match list item bullets and number markers:
	my $marker_ul  = qr/[*+-]/;
	my $marker_ol  = qr/\d+[.]/;
	my $marker_any = qr/(?:$marker_ul|$marker_ol)/;

	# Re-usable pattern to match any entirel ul or ol list:
	my $whole_list = qr{
		(								# $1 = whole list
		  (								# $2
			[ ]{0,$less_than_tab}
			(${marker_any})				# $3 = first list item marker
			[ \t]+
		  )
		  (?s:.+?)
		  (								# $4
			  \z
			|
			  \n{2,}
			  (?=\S)
			  (?!						# Negative lookahead for another list item marker
				[ \t]*
				${marker_any}[ \t]+
			  )
		  )
		)
	}mx;

	# We use a different prefix before nested lists than top-level lists.
	# See extended comment in _ProcessListItems().
	#
	# Note: There's a bit of duplication here. My original implementation
	# created a scalar regex pattern as the conditional result of the test on
	# $g_list_level, and then only ran the $text =~ s{...}{...}egmx
	# substitution once, using the scalar as the pattern. This worked,
	# everywhere except when running under MT on my hosting account at Pair
	# Networks. There, this caused all rebuilds to be killed by the reaper (or
	# perhaps they crashed, but that seems incredibly unlikely given that the
	# same script on the same server ran fine *except* under MT. I've spent
	# more time trying to figure out why this is happening than I'd like to
	# admit. My only guess, backed up by the fact that this workaround works,
	# is that Perl optimizes the substition when it can figure out that the
	# pattern will never change, and when this optimization isn't on, we run
	# afoul of the reaper. Thus, the slightly redundant code to that uses two
	# static s/// patterns rather than one conditional pattern.

	if ($g_list_level) {
		$text =~ s{
				^
				$whole_list
			}{
				my $list = $1;
				my $list_type = ($3 =~ m/$marker_ul/) ? "ul" : "ol";
				# Turn double returns into triple returns, so that we can make a
				# paragraph for the last item in a list, if necessary:
				$list =~ s/\n{2,}/\n\n\n/g;
				my $result = _ProcessListItems($list, $marker_any);
				$result = "<$list_type>\n" . $result . "</$list_type>\n";
				$result;
			}egmx;
	}
	else {
		$text =~ s{
				(?:(?<=\n\n)|\A\n?)
				$whole_list
			}{
				my $list = $1;
				my $list_type = ($3 =~ m/$marker_ul/) ? "ul" : "ol";
				# Turn double returns into triple returns, so that we can make a
				# paragraph for the last item in a list, if necessary:
				$list =~ s/\n{2,}/\n\n\n/g;
				my $result = _ProcessListItems($list, $marker_any);
				$result = "<$list_type>\n" . $result . "</$list_type>\n";
				$result;
			}egmx;
	}


	return $text;
}


sub _ProcessListItems {
#
#	Process the contents of a single ordered or unordered list, splitting it
#	into individual list items.
#

	my $list_str = shift;
	my $marker_any = shift;


	# The $g_list_level global keeps track of when we're inside a list.
	# Each time we enter a list, we increment it; when we leave a list,
	# we decrement. If it's zero, we're not in a list anymore.
	#
	# We do this because when we're not inside a list, we want to treat
	# something like this:
	#
	#		I recommend upgrading to version
	#		8. Oops, now this line is treated
	#		as a sub-list.
	#
	# As a single paragraph, despite the fact that the second line starts
	# with a digit-period-space sequence.
	#
	# Whereas when we're inside a list (or sub-list), that line will be
	# treated as the start of a sub-list. What a kludge, huh? This is
	# an aspect of Markdown's syntax that's hard to parse perfectly
	# without resorting to mind-reading. Perhaps the solution is to
	# change the syntax rules such that sub-lists must start with a
	# starting cardinal number; e.g. "1." or "a.".

	$g_list_level++;

	# trim trailing blank lines:
	$list_str =~ s/\n{2,}\z/\n/;


	$list_str =~ s{
		(\n)?							# leading line = $1
		(^[ \t]*)						# leading whitespace = $2
		($marker_any) [ \t]+			# list marker = $3
		((?s:.+?)						# list item text   = $4
		(\n{1,2}))
		(?= \n* (\z | \2 ($marker_any) [ \t]+))
	}{
		my $item = $4;
		my $leading_line = $1;
		my $leading_space = $2;

		if ($leading_line or ($item =~ m/\n{2,}/)) {
			$item = _RunBlockGamut(_Outdent($item));
		}
		else {
			# Recursion for sub-lists:
			$item = _DoLists(_Outdent($item));
			chomp $item;
			$item = _RunSpanGamut($item);
		}

		"<li>" . $item . "</li>\n";
	}egmx;

	$g_list_level--;
	return $list_str;
}



sub _DoCodeBlocks {
#
#	Process Markdown `<pre><code>` blocks.
#	

	my $text = shift;

	$text =~ s{
			(?:\n\n|\A)
			(	            # $1 = the code block -- one or more lines, starting with a space/tab
			  (?:
			    (?:[ ]{$g_tab_width} | \t)  # Lines must start with a tab or a tab-width of spaces
			    .*\n+
			  )+
			)
			((?=^[ ]{0,$g_tab_width}\S)|\Z)	# Lookahead for non-space at line-start, or end of doc
		}{
			my $codeblock = $1;
			my $result; # return value

			$codeblock = _EncodeCode(_Outdent($codeblock));
			$codeblock = _Detab($codeblock);
			$codeblock =~ s/\A\n+//; # trim leading newlines
			$codeblock =~ s/\s+\z//; # trim trailing whitespace

			$result = "\n\n<pre><code>" . $codeblock . "\n</code></pre>\n\n";

			$result;
		}egmx;

	return $text;
}


sub _DoCodeSpans {
#
# 	*	Backtick quotes are used for <code></code> spans.
# 
# 	*	You can use multiple backticks as the delimiters if you want to
# 		include literal backticks in the code span. So, this input:
#     
#         Just type ``foo `bar` baz`` at the prompt.
#     
#     	Will translate to:
#     
#         <p>Just type <code>foo `bar` baz</code> at the prompt.</p>
#     
#		There's no arbitrary limit to the number of backticks you
#		can use as delimters. If you need three consecutive backticks
#		in your code, use four for delimiters, etc.
#
#	*	You can use spaces to get literal backticks at the edges:
#     
#         ... type `` `bar` `` ...
#     
#     	Turns to:
#     
#         ... type <code>`bar`</code> ...
#

	my $text = shift;

	$text =~ s@
			(`+)		# $1 = Opening run of `
			(.+?)		# $2 = The code block
			(?<!`)
			\1			# Matching closer
			(?!`)
		@
 			my $c = "$2";
 			$c =~ s/^[ \t]*//g; # leading whitespace
 			$c =~ s/[ \t]*$//g; # trailing whitespace
 			$c = _EncodeCode($c);
			"<code>$c</code>";
		@egsx;

	return $text;
}


sub _EncodeCode {
#
# Encode/escape certain characters inside Markdown code runs.
# The point is that in code, these characters are literals,
# and lose their special Markdown meanings.
#
    local $_ = shift;

	# Encode all ampersands; HTML entities are not
	# entities within a Markdown code span.
	s/&/&amp;/g;

	# Encode $'s, but only if we're running under Blosxom.
	# (Blosxom interpolates Perl variables in article bodies.)
	{
		no warnings 'once';
    	if (defined($blosxom::version)) {
    		s/\$/&#036;/g;	
    	}
    }


	# Do the angle bracket song and dance:
	s! <  !&lt;!gx;
	s! >  !&gt;!gx;

	# Now, escape characters that are magic in Markdown:
	s! \* !$g_escape_table{'*'}!gx;
	s! _  !$g_escape_table{'_'}!gx;
	s! {  !$g_escape_table{'{'}!gx;
	s! }  !$g_escape_table{'}'}!gx;
	s! \[ !$g_escape_table{'['}!gx;
	s! \] !$g_escape_table{']'}!gx;
	s! \\ !$g_escape_table{'\\'}!gx;

	return $_;
}


sub _DoItalicsAndBold {
	my $text = shift;

	# <strong> must go first:
	$text =~ s{ (\*\*|__) (?=\S) (.+?[*_]*) (?<=\S) \1 }
		{<strong>$2</strong>}gsx;

	$text =~ s{ (\*|_) (?=\S) (.+?) (?<=\S) \1 }
		{<em>$2</em>}gsx;

	return $text;
}


sub _DoBlockQuotes {
	my $text = shift;

	$text =~ s{
		  (								# Wrap whole match in $1
			(
			  ^[ \t]*>[ \t]?			# '>' at the start of a line
			    .+\n					# rest of the first line
			  (.+\n)*					# subsequent consecutive lines
			  \n*						# blanks
			)+
		  )
		}{
			my $bq = $1;
			$bq =~ s/^[ \t]*>[ \t]?//gm;	# trim one level of quoting
			$bq =~ s/^[ \t]+$//mg;			# trim whitespace-only lines
			$bq = _RunBlockGamut($bq);		# recurse

			$bq =~ s/^/  /g;
			# These leading spaces screw with <pre> content, so we need to fix that:
			$bq =~ s{
					(\s*<pre>.+?</pre>)
				}{
					my $pre = $1;
					$pre =~ s/^  //mg;
					$pre;
				}egsx;

			"<blockquote>\n$bq\n</blockquote>\n\n";
		}egmx;


	return $text;
}


sub _FormParagraphs {
#
#	Params:
#		$text - string to process with html <p> tags
#
	my $text = shift;

	# Strip leading and trailing lines:
	$text =~ s/\A\n+//;
	$text =~ s/\n+\z//;

	my @grafs = split(/\n{2,}/, $text);

	#
	# Wrap <p> tags.
	#
	foreach (@grafs) {
		unless (defined( $g_html_blocks{$_} )) {
			$_ = _RunSpanGamut($_);
			s/^([ \t]*)/<p>/;
			$_ .= "</p>";
		}
	}

	#
	# Unhashify HTML blocks
	#
	foreach (@grafs) {
		if (defined( $g_html_blocks{$_} )) {
			$_ = $g_html_blocks{$_};
		}
	}

	return join "\n\n", @grafs;
}


sub _EncodeAmpsAndAngles {
# Smart processing for ampersands and angle brackets that need to be encoded.

	my $text = shift;

	# Ampersand-encoding based entirely on Nat Irons's Amputator MT plugin:
	#   http://bumppo.net/projects/amputator/
 	$text =~ s/&(?!#?[xX]?(?:[0-9a-fA-F]+|\w+);)/&amp;/g;

	# Encode naked <'s
 	$text =~ s{<(?![a-z/?\$!])}{&lt;}gi;

	return $text;
}


sub _EncodeBackslashEscapes {
#
#   Parameter:  String.
#   Returns:    The string, with after processing the following backslash
#               escape sequences.
#
    local $_ = shift;

    s! \\\\  !$g_escape_table{'\\'}!gx;		# Must process escaped backslashes first.
    s! \\`   !$g_escape_table{'`'}!gx;
    s! \\\*  !$g_escape_table{'*'}!gx;
    s! \\_   !$g_escape_table{'_'}!gx;
    s! \\\{  !$g_escape_table{'{'}!gx;
    s! \\\}  !$g_escape_table{'}'}!gx;
    s! \\\[  !$g_escape_table{'['}!gx;
    s! \\\]  !$g_escape_table{']'}!gx;
    s! \\\(  !$g_escape_table{'('}!gx;
    s! \\\)  !$g_escape_table{')'}!gx;
    s! \\>   !$g_escape_table{'>'}!gx;
    s! \\\#  !$g_escape_table{'#'}!gx;
    s! \\\+  !$g_escape_table{'+'}!gx;
    s! \\\-  !$g_escape_table{'-'}!gx;
    s! \\\.  !$g_escape_table{'.'}!gx;
    s{ \\!  }{$g_escape_table{'!'}}gx;

    return $_;
}


sub _DoAutoLinks {
	my $text = shift;

	$text =~ s{<((https?|ftp):[^'">\s]+)>}{<a href="$1">$1</a>}gi;

	# Email addresses: <address@domain.foo>
	$text =~ s{
		<
        (?:mailto:)?
		(
			[-.\w]+
			\@
			[-a-z0-9]+(\.[-a-z0-9]+)*\.[a-z]+
		)
		>
	}{
		_EncodeEmailAddress( _UnescapeSpecialChars($1) );
	}egix;

	return $text;
}


sub _EncodeEmailAddress {
#
#	Input: an email address, e.g. "foo@example.com"
#
#	Output: the email address as a mailto link, with each character
#		of the address encoded as either a decimal or hex entity, in
#		the hopes of foiling most address harvesting spam bots. E.g.:
#
#	  <a href="&#x6D;&#97;&#105;&#108;&#x74;&#111;:&#102;&#111;&#111;&#64;&#101;
#       x&#x61;&#109;&#x70;&#108;&#x65;&#x2E;&#99;&#111;&#109;">&#102;&#111;&#111;
#       &#64;&#101;x&#x61;&#109;&#x70;&#108;&#x65;&#x2E;&#99;&#111;&#109;</a>
#
#	Based on a filter by Matthew Wickline, posted to the BBEdit-Talk
#	mailing list: <http://tinyurl.com/yu7ue>
#

	my $addr = shift;

	srand;
	my @encode = (
		sub { '&#' .                 ord(shift)   . ';' },
		sub { '&#x' . sprintf( "%X", ord(shift) ) . ';' },
		sub {                            shift          },
	);

	$addr = "mailto:" . $addr;

	$addr =~ s{(.)}{
		my $char = $1;
		if ( $char eq '@' ) {
			# this *must* be encoded. I insist.
			$char = $encode[int rand 1]->($char);
		} elsif ( $char ne ':' ) {
			# leave ':' alone (to spot mailto: later)
			my $r = rand;
			# roughly 10% raw, 45% hex, 45% dec
			$char = (
				$r > .9   ?  $encode[2]->($char)  :
				$r < .45  ?  $encode[1]->($char)  :
							 $encode[0]->($char)
			);
		}
		$char;
	}gex;

	$addr = qq{<a href="$addr">$addr</a>};
	$addr =~ s{">.+?:}{">}; # strip the mailto: from the visible part

	return $addr;
}


sub _UnescapeSpecialChars {
#
# Swap back in all the special characters we've hidden.
#
	my $text = shift;

	while( my($char, $hash) = each(%g_escape_table) ) {
		$text =~ s/$hash/$char/g;
	}
    return $text;
}


sub _TokenizeHTML {
#
#   Parameter:  String containing HTML markup.
#   Returns:    Reference to an array of the tokens comprising the input
#               string. Each token is either a tag (possibly with nested,
#               tags contained therein, such as <a href="<MTFoo>">, or a
#               run of text between tags. Each element of the array is a
#               two-element array; the first is either 'tag' or 'text';
#               the second is the actual value.
#
#
#   Derived from the _tokenize() subroutine from Brad Choate's MTRegex plugin.
#       <http://www.bradchoate.com/past/mtregex.php>
#

    my $str = shift;
    my $pos = 0;
    my $len = length $str;
    my @tokens;

    my $depth = 6;
    my $nested_tags = join('|', ('(?:<[a-z/!$](?:[^<>]') x $depth) . (')*>)' x  $depth);
    my $match = qr/(?s: <! ( -- .*? -- \s* )+ > ) |  # comment
                   (?s: <\? .*? \?> ) |              # processing instruction
                   $nested_tags/ix;                   # nested tags

    while ($str =~ m/($match)/g) {
        my $whole_tag = $1;
        my $sec_start = pos $str;
        my $tag_start = $sec_start - length $whole_tag;
        if ($pos < $tag_start) {
            push @tokens, ['text', substr($str, $pos, $tag_start - $pos)];
        }
        push @tokens, ['tag', $whole_tag];
        $pos = pos $str;
    }
    push @tokens, ['text', substr($str, $pos, $len - $pos)] if $pos < $len;
    \@tokens;
}


sub _Outdent {
#
# Remove one level of line-leading tabs or spaces
#
	my $text = shift;

	$text =~ s/^(\t|[ ]{1,$g_tab_width})//gm;
	return $text;
}


sub _Detab {
#
# Cribbed from a post by Bart Lateur:
# <http://www.nntp.perl.org/group/perl.macperl.anyperl/154>
#
	my $text = shift;

	$text =~ s{(.*?)\t}{$1.(' ' x ($g_tab_width - length($1) % $g_tab_width))}ge;
	return $text;
}


1;

__END__


=pod

=head1 NAME

B<Markdown>


=head1 SYNOPSIS

B<Markdown.pl> [ B<--html4tags> ] [ B<--version> ] [ B<-shortversion> ]
    [ I<file> ... ]


=head1 DESCRIPTION

Markdown is a text-to-HTML filter; it translates an easy-to-read /
easy-to-write structured text format into HTML. Markdown's text format
is most similar to that of plain text email, and supports features such
as headers, *emphasis*, code blocks, blockquotes, and links.

Markdown's syntax is designed not as a generic markup language, but
specifically to serve as a front-end to (X)HTML. You can  use span-level
HTML tags anywhere in a Markdown document, and you can use block level
HTML tags (like <div> and <table> as well).

For more information about Markdown's syntax, see:

    http://daringfireball.net/projects/markdown/


=head1 OPTIONS

Use "--" to end switch parsing. For example, to open a file named "-z", use:

	Markdown.pl -- -z

=over 4


=item B<--html4tags>

Use HTML 4 style for empty element tags, e.g.:

    <br>

instead of Markdown's default XHTML style tags, e.g.:

    <br />


=item B<-v>, B<--version>

Display Markdown's version number and copyright information.


=item B<-s>, B<--shortversion>

Display the short-form version number.


=back



=head1 BUGS

To file bug reports or feature requests (other than topics listed in the
Caveats section above) please send email to:

    support@daringfireball.net

Please include with your report: (1) the example input; (2) the output
you expected; (3) the output Markdown actually produced.


=head1 VERSION HISTORY

See the readme file for detailed release notes for this version.

1.0.1 - 14 Dec 2004

1.0 - 28 Aug 2004


=head1 AUTHOR

    John Gruber
    http://daringfireball.net

    PHP port and other contributions by Michel Fortin
    http://michelf.com


=head1 COPYRIGHT AND LICENSE

Copyright (c) 2003-2004 John Gruber   
<http://daringfireball.net/>   
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are
met:

* Redistributions of source code must retain the above copyright notice,
  this list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright
  notice, this list of conditions and the following disclaimer in the
  documentation and/or other materials provided with the distribution.

* Neither the name "Markdown" nor the names of its contributors may
  be used to endorse or promote products derived from this software
  without specific prior written permission.

This software is provided by the copyright holders and contributors "as
is" and any express or implied warranties, including, but not limited
to, the implied warranties of merchantability and fitness for a
particular purpose are disclaimed. In no event shall the copyright owner
or contributors be liable for any direct, indirect, incidental, special,
exemplary, or consequential damages (including, but not limited to,
procurement of substitute goods or services; loss of use, data, or
profits; or business interruption) however caused and on any theory of
liability, whether in contract, strict liability, or tort (including
negligence or otherwise) arising in any way out of the use of this
software, even if advised of the possibility of such damage.

=cut

# A First Level Header
====================

A Second Level Header
---------------------

Now is the time for all good men to come to
the aid of their country. This is just a
regular paragraph.

The quick brown fox jumped over the lazy
dog's back.

### Header 3

> This is a blockquote.
> 
> This is the second paragraph in the blockquote.
>
> ## This is an H2 in a blockquote
Вихід:

<h1>A First Level Header</h1>

<h2>A Second Level Header</h2>

<p>Now is the time for all good men to come to
the aid of their country. This is just a
regular paragraph.</p>

<p>The quick brown fox jumped over the lazy
dog's back.</p>

<h3>Header 3</h3>

<blockquote>
    <p>This is a blockquote.</p>

    <p>This is the second paragraph in the blockquote.</p>

    <h2>This is an H2 in a blockquote</h2>
</blockquote>
АКЦЕНТ НА ФРАЗУ
Markdown використовує зірочки та підкреслення для позначення проміжків акценту.

Markdown:

Some of these words *are emphasized*.
Some of these words _are emphasized also_.

Use two asterisks for **strong emphasis**.
Or, if you prefer, __use two underscores instead__.
Вихід:

<p>Some of these words <em>are emphasized</em>.
Some of these words <em>are emphasized also</em>.</p>

<p>Use two asterisks for <strong>strong emphasis</strong>.
Or, if you prefer, <strong>use two underscores instead</strong>.</p>
СПИСКИ
Невпорядковані (марковані) списки використовують зірочки, плюси і дефіси ( *, +і -) як список маркера. Ці три маркери взаємозамінні; це:

*   Candy.
*   Gum.
*   Booze.
це:

+   Candy.
+   Gum.
+   Booze.
і це:

-   Candy.
-   Gum.
-   Booze.
всі виробляють один і той же вихід:

<ul>
<li>Candy.</li>
<li>Gum.</li>
<li>Booze.</li>
</ul>
Впорядковані (нумеровані) списки використовують звичайні числа, за якими слідують періоди, як маркери списку:

1.  Red
2.  Green
3.  Blue
Вихід:

<ol>
<li>Red</li>
<li>Green</li>
<li>Blue</li>
</ol>
Якщо покласти порожні рядки між елементами, ви отримаєте <p>теги для тексту елемента списку. Ви можете створити елементи списку з кількома абзацами, відступивши абзаци 4 пробілами або 1 вкладкою:

*   A list item.

    With multiple paragraphs.

*   Another item in the list.
Вихід:

<ul>
<li><p>A list item.</p>
<p>With multiple paragraphs.</p></li>
<li><p>Another item in the list.</p></li>
</ul>
ПОСИЛАННЯ
Markdown підтримує два стилі для створення посилань: inline та reference . У обох стилях ви використовуєте квадратні дужки, щоб розділити текст, який ви хочете перетворити на посилання.

Вбудовані посилання використовують дужки відразу після тексту посилання. Наприклад:

This is an [example link](http://example.com/).
Вихід:

<p>This is an <a href="http://example.com/">
example link</a>.</p>
За бажанням, ви можете включити атрибут title у дужки:

This is an [example link](http://example.com/ "With a Title").
Вихід:

<p>This is an <a href="http://example.com/" title="With a Title">
example link</a>.</p>
Посилання посилального стилю дозволяють посилатися на свої посилання по іменах, які ви визначаєте в іншому місці вашого документа:

I get 10 times more traffic from [Google][1] than from
[Yahoo][2] or [MSN][3].

[1]: http://google.com/        "Google"
[2]: http://search.yahoo.com/  "Yahoo Search"
[3]: http://search.msn.com/    "MSN Search"
Вихід:

<p>I get 10 times more traffic from <a href="http://google.com/"
title="Google">Google</a> than from <a href="http://search.yahoo.com/"
title="Yahoo Search">Yahoo</a> or <a href="http://search.msn.com/"
title="MSN Search">MSN</a>.</p>
Атрибут title не є обов'язковим. Імена посилань можуть містити літери, цифри та пробіли, але не чутливі до регістру:

I start my morning with a cup of coffee and
[The New York Times][NY Times].

[ny times]: http://www.nytimes.com/
Вихід:

<p>I start my morning with a cup of coffee and
<a href="http://www.nytimes.com/">The New York Times</a>.</p>
ЗОБРАЖЕННЯ
Синтаксис зображення дуже схожий на синтаксис посилання.

Вбудовано (назви не обов'язкові):

![alt text](/path/to/img.jpg "Title")
Довідковий стиль:

![alt text][id]

[id]: /path/to/img.jpg "Title"
Обидва наведені вище приклади дають однаковий результат:

<img src="/path/to/img.jpg" alt="alt text" title="Title" />
КОД
У звичайному абзаці можна створити проміжок коду, обернувши текст у лапки. Будь-які амперсанди ( &) і кутові дужки ( <або >) автоматично переводяться до HTML-об'єктів. Це дозволяє легко використовувати Markdown для написання прикладів коду HTML:

I strongly recommend against using any `<blink>` tags.

I wish SmartyPants used named entities like `&mdash;`
instead of decimal-encoded entites like `&#8212;`.
Вихід:

<p>I strongly recommend against using any
<code>&lt;blink&gt;</code> tags.</p>

<p>I wish SmartyPants used named entities like
<code>&amp;mdash;</code> instead of decimal-encoded
entites like <code>&amp;#8212;</code>.</p>
Щоб вказати весь блок попередньо відформатованого коду, відступ кожному рядку блоку на 4 пробіли або 1 вкладку. Так само , як з кодом охоплює, &, <, і >символи будуть екрановані автоматично.

Markdown:

If you want your page to validate under XHTML 1.0 Strict,
you've got to put paragraph tags in your blockquotes:

    <blockquote>
        <p>For example.</p>
    </blockquote>
Вихід:

<p>If you want your page to validate under XHTML 1.0 Strict,
you've got to put paragraph tags in your blockquotes:</p>

<pre><code>&lt;blockquote&gt;
    &lt;p&gt;For example.&lt;/p&gt;
&lt;/blockquote&gt;
</code></pre>

{
    "_readme": [
        "This file locks the dependencies of your project to a known state",
        "Read more about it at https://getcomposer.org/doc/01-basic-usage.md#composer-lock-the-lock-file",
        "This file is @generated automatically"
    ],
    "content-hash": "c1b70f55d29e7833e3d354e7a1775641",
    "packages": [
        {
            "name": "defuse/php-encryption",
            "version": "v2.2.1",
            "source": {
                "type": "git",
                "url": "https://github.com/defuse/php-encryption.git",
                "reference": "0f407c43b953d571421e0020ba92082ed5fb7620"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/defuse/php-encryption/zipball/0f407c43b953d571421e0020ba92082ed5fb7620",
                "reference": "0f407c43b953d571421e0020ba92082ed5fb7620",
                "shasum": ""
            },
            "require": {
                "ext-openssl": "*",
                "paragonie/random_compat": ">= 2",
                "php": ">=5.4.0"
            },
            "require-dev": {
                "nikic/php-parser": "^2.0|^3.0|^4.0",
                "phpunit/phpunit": "^4|^5"
            },
            "bin": [
                "bin/generate-defuse-key"
            ],
            "type": "library",
            "autoload": {
                "psr-4": {
                    "Defuse\\Crypto\\": "src"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Taylor Hornby",
                    "email": "taylor@defuse.ca",
                    "homepage": "https://defuse.ca/"
                },
                {
                    "name": "Scott Arciszewski",
                    "email": "info@paragonie.com",
                    "homepage": "https://paragonie.com"
                }
            ],
            "description": "Secure PHP Encryption Library",
            "keywords": [
                "aes",
                "authenticated encryption",
                "cipher",
                "crypto",
                "cryptography",
                "encrypt",
                "encryption",
                "openssl",
                "security",
                "symmetric key cryptography"
            ],
            "time": "2018-07-24T23:27:56+00:00"
        },
        {
            "name": "doctrine/lexer",
            "version": "v1.0.1",
            "source": {
                "type": "git",
                "url": "https://github.com/doctrine/lexer.git",
                "reference": "83893c552fd2045dd78aef794c31e694c37c0b8c"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/doctrine/lexer/zipball/83893c552fd2045dd78aef794c31e694c37c0b8c",
                "reference": "83893c552fd2045dd78aef794c31e694c37c0b8c",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3.2"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.0.x-dev"
                }
            },
            "autoload": {
                "psr-0": {
                    "Doctrine\\Common\\Lexer\\": "lib/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Roman Borschel",
                    "email": "roman@code-factory.org"
                },
                {
                    "name": "Guilherme Blanco",
                    "email": "guilhermeblanco@gmail.com"
                },
                {
                    "name": "Johannes Schmitt",
                    "email": "schmittjoh@gmail.com"
                }
            ],
            "description": "Base library for a lexer that can be used in Top-Down, Recursive Descent Parsers.",
            "homepage": "http://www.doctrine-project.org",
            "keywords": [
                "lexer",
                "parser"
            ],
            "time": "2014-09-09T13:34:57+00:00"
        },
        {
            "name": "egulias/email-validator",
            "version": "2.1.7",
            "source": {
                "type": "git",
                "url": "https://github.com/egulias/EmailValidator.git",
                "reference": "709f21f92707308cdf8f9bcfa1af4cb26586521e"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/egulias/EmailValidator/zipball/709f21f92707308cdf8f9bcfa1af4cb26586521e",
                "reference": "709f21f92707308cdf8f9bcfa1af4cb26586521e",
                "shasum": ""
            },
            "require": {
                "doctrine/lexer": "^1.0.1",
                "php": ">= 5.5"
            },
            "require-dev": {
                "dominicsayers/isemail": "dev-master",
                "phpunit/phpunit": "^4.8.35||^5.7||^6.0",
                "satooshi/php-coveralls": "^1.0.1"
            },
            "suggest": {
                "ext-intl": "PHP Internationalization Libraries are required to use the SpoofChecking validation"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "2.0.x-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "Egulias\\EmailValidator\\": "EmailValidator"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Eduardo Gulias Davis"
                }
            ],
            "description": "A library for validating emails against several RFCs",
            "homepage": "https://github.com/egulias/EmailValidator",
            "keywords": [
                "email",
                "emailvalidation",
                "emailvalidator",
                "validation",
                "validator"
            ],
            "time": "2018-12-04T22:38:24+00:00"
        },
        {
            "name": "guzzlehttp/guzzle",
            "version": "6.3.3",
            "source": {
                "type": "git",
                "url": "https://github.com/guzzle/guzzle.git",
                "reference": "407b0cb880ace85c9b63c5f9551db498cb2d50ba"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/guzzle/guzzle/zipball/407b0cb880ace85c9b63c5f9551db498cb2d50ba",
                "reference": "407b0cb880ace85c9b63c5f9551db498cb2d50ba",
                "shasum": ""
            },
            "require": {
                "guzzlehttp/promises": "^1.0",
                "guzzlehttp/psr7": "^1.4",
                "php": ">=5.5"
            },
            "require-dev": {
                "ext-curl": "*",
                "phpunit/phpunit": "^4.8.35 || ^5.7 || ^6.4 || ^7.0",
                "psr/log": "^1.0"
            },
            "suggest": {
                "psr/log": "Required for using the Log middleware"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "6.3-dev"
                }
            },
            "autoload": {
                "files": [
                    "src/functions_include.php"
                ],
                "psr-4": {
                    "GuzzleHttp\\": "src/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Michael Dowling",
                    "email": "mtdowling@gmail.com",
                    "homepage": "https://github.com/mtdowling"
                }
            ],
            "description": "Guzzle is a PHP HTTP client library",
            "homepage": "http://guzzlephp.org/",
            "keywords": [
                "client",
                "curl",
                "framework",
                "http",
                "http client",
                "rest",
                "web service"
            ],
            "time": "2018-04-22T15:46:56+00:00"
        },
        {
            "name": "guzzlehttp/promises",
            "version": "v1.3.1",
            "source": {
                "type": "git",
                "url": "https://github.com/guzzle/promises.git",
                "reference": "a59da6cf61d80060647ff4d3eb2c03a2bc694646"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/guzzle/promises/zipball/a59da6cf61d80060647ff4d3eb2c03a2bc694646",
                "reference": "a59da6cf61d80060647ff4d3eb2c03a2bc694646",
                "shasum": ""
            },
            "require": {
                "php": ">=5.5.0"
            },
            "require-dev": {
                "phpunit/phpunit": "^4.0"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.4-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "GuzzleHttp\\Promise\\": "src/"
                },
                "files": [
                    "src/functions_include.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Michael Dowling",
                    "email": "mtdowling@gmail.com",
                    "homepage": "https://github.com/mtdowling"
                }
            ],
            "description": "Guzzle promises library",
            "keywords": [
                "promise"
            ],
            "time": "2016-12-20T10:07:11+00:00"
        },
        {
            "name": "guzzlehttp/psr7",
            "version": "1.5.2",
            "source": {
                "type": "git",
                "url": "https://github.com/guzzle/psr7.git",
                "reference": "9f83dded91781a01c63574e387eaa769be769115"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/guzzle/psr7/zipball/9f83dded91781a01c63574e387eaa769be769115",
                "reference": "9f83dded91781a01c63574e387eaa769be769115",
                "shasum": ""
            },
            "require": {
                "php": ">=5.4.0",
                "psr/http-message": "~1.0",
                "ralouphie/getallheaders": "^2.0.5"
            },
            "provide": {
                "psr/http-message-implementation": "1.0"
            },
            "require-dev": {
                "phpunit/phpunit": "~4.8.36 || ^5.7.27 || ^6.5.8"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.5-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "GuzzleHttp\\Psr7\\": "src/"
                },
                "files": [
                    "src/functions_include.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Michael Dowling",
                    "email": "mtdowling@gmail.com",
                    "homepage": "https://github.com/mtdowling"
                },
                {
                    "name": "Tobias Schultze",
                    "homepage": "https://github.com/Tobion"
                }
            ],
            "description": "PSR-7 message implementation that also provides common utility methods",
            "keywords": [
                "http",
                "message",
                "psr-7",
                "request",
                "response",
                "stream",
                "uri",
                "url"
            ],
            "time": "2018-12-04T20:46:45+00:00"
        },
        {
            "name": "html2text/html2text",
            "version": "4.2.1",
            "source": {
                "type": "git",
                "url": "https://github.com/mtibben/html2text.git",
                "reference": "f7555eaf271beea4e1098274d3ff37fbb7b21ea7"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/mtibben/html2text/zipball/f7555eaf271beea4e1098274d3ff37fbb7b21ea7",
                "reference": "f7555eaf271beea4e1098274d3ff37fbb7b21ea7",
                "shasum": ""
            },
            "require-dev": {
                "phpunit/phpunit": "~4"
            },
            "suggest": {
                "ext-mbstring": "For best performance",
                "symfony/polyfill-mbstring": "If you can't install ext-mbstring"
            },
            "type": "library",
            "autoload": {
                "psr-4": {
                    "Html2Text\\": [
                        "src/",
                        "test/"
                    ]
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "GPL-2.0-or-later"
            ],
            "description": "Converts HTML to formatted plain text",
            "time": "2018-08-13T12:05:08+00:00"
        },
        {
            "name": "matthiasmullie/minify",
            "version": "1.3.61",
            "source": {
                "type": "git",
                "url": "https://github.com/matthiasmullie/minify.git",
                "reference": "d5acb8ce5b6acb7d11bafe97cecc533f6e4fd751"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/matthiasmullie/minify/zipball/d5acb8ce5b6acb7d11bafe97cecc533f6e4fd751",
                "reference": "d5acb8ce5b6acb7d11bafe97cecc533f6e4fd751",
                "shasum": ""
            },
            "require": {
                "ext-pcre": "*",
                "matthiasmullie/path-converter": "~1.1",
                "php": ">=5.3.0"
            },
            "require-dev": {
                "friendsofphp/php-cs-fixer": "~2.0",
                "matthiasmullie/scrapbook": "~1.0",
                "phpunit/phpunit": "~4.8"
            },
            "suggest": {
                "psr/cache-implementation": "Cache implementation to use with Minify::cache"
            },
            "bin": [
                "bin/minifycss",
                "bin/minifyjs"
            ],
            "type": "library",
            "autoload": {
                "psr-4": {
                    "MatthiasMullie\\Minify\\": "src/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Matthias Mullie",
                    "email": "minify@mullie.eu",
                    "homepage": "http://www.mullie.eu",
                    "role": "Developer"
                }
            ],
            "description": "CSS & JavaScript minifier, in PHP. Removes whitespace, strips comments, combines files (incl. @import statements and small assets in CSS files), and optimizes/shortens a few common programming patterns.",
            "homepage": "http://www.minifier.org",
            "keywords": [
                "JS",
                "css",
                "javascript",
                "minifier",
                "minify"
            ],
            "time": "2018-11-26T23:10:39+00:00"
        },
        {
            "name": "matthiasmullie/path-converter",
            "version": "1.1.2",
            "source": {
                "type": "git",
                "url": "https://github.com/matthiasmullie/path-converter.git",
                "reference": "5e4b121c8b9f97c80835c1d878b0812ba1d607c9"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/matthiasmullie/path-converter/zipball/5e4b121c8b9f97c80835c1d878b0812ba1d607c9",
                "reference": "5e4b121c8b9f97c80835c1d878b0812ba1d607c9",
                "shasum": ""
            },
            "require": {
                "ext-pcre": "*",
                "php": ">=5.3.0"
            },
            "require-dev": {
                "phpunit/phpunit": "~4.8"
            },
            "type": "library",
            "autoload": {
                "psr-4": {
                    "MatthiasMullie\\PathConverter\\": "src/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Matthias Mullie",
                    "email": "pathconverter@mullie.eu",
                    "homepage": "http://www.mullie.eu",
                    "role": "Developer"
                }
            ],
            "description": "Relative path converter",
            "homepage": "http://github.com/matthiasmullie/path-converter",
            "keywords": [
                "converter",
                "path",
                "paths",
                "relative"
            ],
            "time": "2018-10-25T15:19:41+00:00"
        },
        {
            "name": "michelf/php-markdown",
            "version": "1.8.0",
            "source": {
                "type": "git",
                "url": "https://github.com/michelf/php-markdown.git",
                "reference": "01ab082b355bf188d907b9929cd99b2923053495"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/michelf/php-markdown/zipball/01ab082b355bf188d907b9929cd99b2923053495",
                "reference": "01ab082b355bf188d907b9929cd99b2923053495",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3.0"
            },
            "type": "library",
            "autoload": {
                "psr-4": {
                    "Michelf\\": "Michelf/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Michel Fortin",
                    "email": "michel.fortin@michelf.ca",
                    "homepage": "https://michelf.ca/",
                    "role": "Developer"
                },
                {
                    "name": "John Gruber",
                    "homepage": "https://daringfireball.net/"
                }
            ],
            "description": "PHP Markdown",
            "homepage": "https://michelf.ca/projects/php-markdown/",
            "keywords": [
                "markdown"
            ],
            "time": "2018-01-15T00:49:33+00:00"
        },
        {
            "name": "mtdowling/cron-expression",
            "version": "v1.2.1",
            "source": {
                "type": "git",
                "url": "https://github.com/mtdowling/cron-expression.git",
                "reference": "9504fa9ea681b586028adaaa0877db4aecf32bad"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/mtdowling/cron-expression/zipball/9504fa9ea681b586028adaaa0877db4aecf32bad",
                "reference": "9504fa9ea681b586028adaaa0877db4aecf32bad",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3.2"
            },
            "require-dev": {
                "phpunit/phpunit": "~4.0|~5.0"
            },
            "type": "library",
            "autoload": {
                "psr-4": {
                    "Cron\\": "src/Cron/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Michael Dowling",
                    "email": "mtdowling@gmail.com",
                    "homepage": "https://github.com/mtdowling"
                }
            ],
            "description": "CRON for PHP: Calculate the next or previous run date and determine if a CRON expression is due",
            "keywords": [
                "cron",
                "schedule"
            ],
            "time": "2017-01-23T04:29:33+00:00"
        },
        {
            "name": "paragonie/random_compat",
            "version": "v9.99.99",
            "source": {
                "type": "git",
                "url": "https://github.com/paragonie/random_compat.git",
                "reference": "84b4dfb120c6f9b4ff7b3685f9b8f1aa365a0c95"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/paragonie/random_compat/zipball/84b4dfb120c6f9b4ff7b3685f9b8f1aa365a0c95",
                "reference": "84b4dfb120c6f9b4ff7b3685f9b8f1aa365a0c95",
                "shasum": ""
            },
            "require": {
                "php": "^7"
            },
            "require-dev": {
                "phpunit/phpunit": "4.*|5.*",
                "vimeo/psalm": "^1"
            },
            "suggest": {
                "ext-libsodium": "Provides a modern crypto API that can be used to generate random bytes."
            },
            "type": "library",
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Paragon Initiative Enterprises",
                    "email": "security@paragonie.com",
                    "homepage": "https://paragonie.com"
                }
            ],
            "description": "PHP 5.x polyfill for random_bytes() and random_int() from PHP 7",
            "keywords": [
                "csprng",
                "polyfill",
                "pseudorandom",
                "random"
            ],
            "time": "2018-07-02T15:55:56+00:00"
        },
        {
            "name": "phpoffice/phpexcel",
            "version": "1.8.2",
            "source": {
                "type": "git",
                "url": "https://github.com/PHPOffice/PHPExcel.git",
                "reference": "1441011fb7ecdd8cc689878f54f8b58a6805f870"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/PHPOffice/PHPExcel/zipball/1441011fb7ecdd8cc689878f54f8b58a6805f870",
                "reference": "1441011fb7ecdd8cc689878f54f8b58a6805f870",
                "shasum": ""
            },
            "require": {
                "ext-mbstring": "*",
                "ext-xml": "*",
                "ext-xmlwriter": "*",
                "php": "^5.2|^7.0"
            },
            "require-dev": {
                "squizlabs/php_codesniffer": "2.*"
            },
            "type": "library",
            "autoload": {
                "psr-0": {
                    "PHPExcel": "Classes/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "LGPL-2.1"
            ],
            "authors": [
                {
                    "name": "Maarten Balliauw",
                    "homepage": "http://blog.maartenballiauw.be"
                },
                {
                    "name": "Erik Tilt"
                },
                {
                    "name": "Franck Lefevre",
                    "homepage": "http://rootslabs.net"
                },
                {
                    "name": "Mark Baker",
                    "homepage": "http://markbakeruk.net"
                }
            ],
            "description": "PHPExcel - OpenXML - Read, Create and Write Spreadsheet documents in PHP - Spreadsheet engine",
            "homepage": "https://github.com/PHPOffice/PHPExcel",
            "keywords": [
                "OpenXML",
                "excel",
                "php",
                "spreadsheet",
                "xls",
                "xlsx"
            ],
            "abandoned": "phpoffice/phpspreadsheet",
            "time": "2018-11-22T23:07:24+00:00"
        },
        {
            "name": "psr/http-message",
            "version": "1.0.1",
            "source": {
                "type": "git",
                "url": "https://github.com/php-fig/http-message.git",
                "reference": "f6561bf28d520154e4b0ec72be95418abe6d9363"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/php-fig/http-message/zipball/f6561bf28d520154e4b0ec72be95418abe6d9363",
                "reference": "f6561bf28d520154e4b0ec72be95418abe6d9363",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3.0"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.0.x-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "Psr\\Http\\Message\\": "src/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "PHP-FIG",
                    "homepage": "http://www.php-fig.org/"
                }
            ],
            "description": "Common interface for HTTP messages",
            "homepage": "https://github.com/php-fig/http-message",
            "keywords": [
                "http",
                "http-message",
                "psr",
                "psr-7",
                "request",
                "response"
            ],
            "time": "2016-08-06T14:39:51+00:00"
        },
        {
            "name": "psr/log",
            "version": "1.1.0",
            "source": {
                "type": "git",
                "url": "https://github.com/php-fig/log.git",
                "reference": "6c001f1daafa3a3ac1d8ff69ee4db8e799a654dd"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/php-fig/log/zipball/6c001f1daafa3a3ac1d8ff69ee4db8e799a654dd",
                "reference": "6c001f1daafa3a3ac1d8ff69ee4db8e799a654dd",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3.0"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.0.x-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "Psr\\Log\\": "Psr/Log/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "PHP-FIG",
                    "homepage": "http://www.php-fig.org/"
                }
            ],
            "description": "Common interface for logging libraries",
            "homepage": "https://github.com/php-fig/log",
            "keywords": [
                "log",
                "psr",
                "psr-3"
            ],
            "time": "2018-11-20T15:27:04+00:00"
        },
        {
            "name": "ralouphie/getallheaders",
            "version": "2.0.5",
            "source": {
                "type": "git",
                "url": "https://github.com/ralouphie/getallheaders.git",
                "reference": "5601c8a83fbba7ef674a7369456d12f1e0d0eafa"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/ralouphie/getallheaders/zipball/5601c8a83fbba7ef674a7369456d12f1e0d0eafa",
                "reference": "5601c8a83fbba7ef674a7369456d12f1e0d0eafa",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3"
            },
            "require-dev": {
                "phpunit/phpunit": "~3.7.0",
                "satooshi/php-coveralls": ">=1.0"
            },
            "type": "library",
            "autoload": {
                "files": [
                    "src/getallheaders.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Ralph Khattar",
                    "email": "ralph.khattar@gmail.com"
                }
            ],
            "description": "A polyfill for getallheaders.",
            "time": "2016-02-11T07:05:27+00:00"
        },
        {
            "name": "sabre/dav",
            "version": "3.2.3",
            "source": {
                "type": "git",
                "url": "https://github.com/sabre-io/dav.git",
                "reference": "a9780ce4f35560ecbd0af524ad32d9d2c8954b80"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sabre-io/dav/zipball/a9780ce4f35560ecbd0af524ad32d9d2c8954b80",
                "reference": "a9780ce4f35560ecbd0af524ad32d9d2c8954b80",
                "shasum": ""
            },
            "require": {
                "ext-ctype": "*",
                "ext-date": "*",
                "ext-dom": "*",
                "ext-iconv": "*",
                "ext-mbstring": "*",
                "ext-pcre": "*",
                "ext-simplexml": "*",
                "ext-spl": "*",
                "lib-libxml": ">=2.7.0",
                "php": ">=5.5.0",
                "psr/log": "^1.0",
                "sabre/event": ">=2.0.0, <4.0.0",
                "sabre/http": "^4.2.1",
                "sabre/uri": "^1.0.1",
                "sabre/vobject": "^4.1.0",
                "sabre/xml": "^1.4.0"
            },
            "require-dev": {
                "evert/phpdoc-md": "~0.1.0",
                "monolog/monolog": "^1.18",
                "phpunit/phpunit": "> 4.8, <6.0.0",
                "sabre/cs": "^1.0.0"
            },
            "suggest": {
                "ext-curl": "*",
                "ext-pdo": "*"
            },
            "bin": [
                "bin/sabredav",
                "bin/naturalselection"
            ],
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "3.1.0-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "Sabre\\DAV\\": "lib/DAV/",
                    "Sabre\\DAVACL\\": "lib/DAVACL/",
                    "Sabre\\CalDAV\\": "lib/CalDAV/",
                    "Sabre\\CardDAV\\": "lib/CardDAV/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Evert Pot",
                    "email": "me@evertpot.com",
                    "homepage": "http://evertpot.com/",
                    "role": "Developer"
                }
            ],
            "description": "WebDAV Framework for PHP",
            "homepage": "http://sabre.io/",
            "keywords": [
                "CalDAV",
                "CardDAV",
                "WebDAV",
                "framework",
                "iCalendar"
            ],
            "time": "2018-10-19T09:58:27+00:00"
        },
        {
            "name": "sabre/event",
            "version": "3.0.0",
            "source": {
                "type": "git",
                "url": "https://github.com/sabre-io/event.git",
                "reference": "831d586f5a442dceacdcf5e9c4c36a4db99a3534"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sabre-io/event/zipball/831d586f5a442dceacdcf5e9c4c36a4db99a3534",
                "reference": "831d586f5a442dceacdcf5e9c4c36a4db99a3534",
                "shasum": ""
            },
            "require": {
                "php": ">=5.5"
            },
            "require-dev": {
                "phpunit/phpunit": "*",
                "sabre/cs": "~0.0.4"
            },
            "type": "library",
            "autoload": {
                "psr-4": {
                    "Sabre\\Event\\": "lib/"
                },
                "files": [
                    "lib/coroutine.php",
                    "lib/Loop/functions.php",
                    "lib/Promise/functions.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Evert Pot",
                    "email": "me@evertpot.com",
                    "homepage": "http://evertpot.com/",
                    "role": "Developer"
                }
            ],
            "description": "sabre/event is a library for lightweight event-based programming",
            "homepage": "http://sabre.io/event/",
            "keywords": [
                "EventEmitter",
                "async",
                "events",
                "hooks",
                "plugin",
                "promise",
                "signal"
            ],
            "time": "2015-11-05T20:14:39+00:00"
        },
        {
            "name": "sabre/http",
            "version": "v4.2.4",
            "source": {
                "type": "git",
                "url": "https://github.com/sabre-io/http.git",
                "reference": "acccec4ba863959b2d10c1fa0fb902736c5c8956"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sabre-io/http/zipball/acccec4ba863959b2d10c1fa0fb902736c5c8956",
                "reference": "acccec4ba863959b2d10c1fa0fb902736c5c8956",
                "shasum": ""
            },
            "require": {
                "ext-ctype": "*",
                "ext-mbstring": "*",
                "php": ">=5.4",
                "sabre/event": ">=1.0.0,<4.0.0",
                "sabre/uri": "~1.0"
            },
            "require-dev": {
                "phpunit/phpunit": "~4.3",
                "sabre/cs": "~0.0.1"
            },
            "suggest": {
                "ext-curl": " to make http requests with the Client class"
            },
            "type": "library",
            "autoload": {
                "files": [
                    "lib/functions.php"
                ],
                "psr-4": {
                    "Sabre\\HTTP\\": "lib/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Evert Pot",
                    "email": "me@evertpot.com",
                    "homepage": "http://evertpot.com/",
                    "role": "Developer"
                }
            ],
            "description": "The sabre/http library provides utilities for dealing with http requests and responses. ",
            "homepage": "https://github.com/fruux/sabre-http",
            "keywords": [
                "http"
            ],
            "time": "2018-02-23T11:10:29+00:00"
        },
        {
            "name": "sabre/uri",
            "version": "1.2.1",
            "source": {
                "type": "git",
                "url": "https://github.com/sabre-io/uri.git",
                "reference": "ada354d83579565949d80b2e15593c2371225e61"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sabre-io/uri/zipball/ada354d83579565949d80b2e15593c2371225e61",
                "reference": "ada354d83579565949d80b2e15593c2371225e61",
                "shasum": ""
            },
            "require": {
                "php": ">=5.4.7"
            },
            "require-dev": {
                "phpunit/phpunit": ">=4.0,<6.0",
                "sabre/cs": "~1.0.0"
            },
            "type": "library",
            "autoload": {
                "files": [
                    "lib/functions.php"
                ],
                "psr-4": {
                    "Sabre\\Uri\\": "lib/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Evert Pot",
                    "email": "me@evertpot.com",
                    "homepage": "http://evertpot.com/",
                    "role": "Developer"
                }
            ],
            "description": "Functions for making sense out of URIs.",
            "homepage": "http://sabre.io/uri/",
            "keywords": [
                "rfc3986",
                "uri",
                "url"
            ],
            "time": "2017-02-20T19:59:28+00:00"
        },
        {
            "name": "sabre/vobject",
            "version": "4.2.0",
            "source": {
                "type": "git",
                "url": "https://github.com/sabre-io/vobject.git",
                "reference": "bd500019764e434ff65872d426f523e7882a0739"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sabre-io/vobject/zipball/bd500019764e434ff65872d426f523e7882a0739",
                "reference": "bd500019764e434ff65872d426f523e7882a0739",
                "shasum": ""
            },
            "require": {
                "ext-mbstring": "*",
                "php": ">=5.5",
                "sabre/xml": ">=1.5 <3.0"
            },
            "require-dev": {
                "phpunit/phpunit": "> 4.8.35, <6.0.0"
            },
            "suggest": {
                "hoa/bench": "If you would like to run the benchmark scripts"
            },
            "bin": [
                "bin/vobject",
                "bin/generate_vcards"
            ],
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "4.0.x-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "Sabre\\VObject\\": "lib/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Evert Pot",
                    "email": "me@evertpot.com",
                    "homepage": "http://evertpot.com/",
                    "role": "Developer"
                },
                {
                    "name": "Dominik Tobschall",
                    "email": "dominik@fruux.com",
                    "homepage": "http://tobschall.de/",
                    "role": "Developer"
                },
                {
                    "name": "Ivan Enderlin",
                    "email": "ivan.enderlin@hoa-project.net",
                    "homepage": "http://mnt.io/",
                    "role": "Developer"
                }
            ],
            "description": "The VObject library for PHP allows you to easily parse and manipulate iCalendar and vCard objects",
            "homepage": "http://sabre.io/vobject/",
            "keywords": [
                "availability",
                "freebusy",
                "iCalendar",
                "ical",
                "ics",
                "jCal",
                "jCard",
                "recurrence",
                "rfc2425",
                "rfc2426",
                "rfc2739",
                "rfc4770",
                "rfc5545",
                "rfc5546",
                "rfc6321",
                "rfc6350",
                "rfc6351",
                "rfc6474",
                "rfc6638",
                "rfc6715",
                "rfc6868",
                "vCalendar",
                "vCard",
                "vcf",
                "xCal",
                "xCard"
            ],
            "time": "2019-02-19T13:05:37+00:00"
        },
        {
            "name": "sabre/xml",
            "version": "1.5.1",
            "source": {
                "type": "git",
                "url": "https://github.com/sabre-io/xml.git",
                "reference": "a367665f1df614c3b8fefc30a54de7cd295e444e"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sabre-io/xml/zipball/a367665f1df614c3b8fefc30a54de7cd295e444e",
                "reference": "a367665f1df614c3b8fefc30a54de7cd295e444e",
                "shasum": ""
            },
            "require": {
                "ext-dom": "*",
                "ext-xmlreader": "*",
                "ext-xmlwriter": "*",
                "lib-libxml": ">=2.6.20",
                "php": ">=5.5.5",
                "sabre/uri": ">=1.0,<3.0.0"
            },
            "require-dev": {
                "phpunit/phpunit": "~4.8|~5.7",
                "sabre/cs": "~1.0.0"
            },
            "type": "library",
            "autoload": {
                "psr-4": {
                    "Sabre\\Xml\\": "lib/"
                },
                "files": [
                    "lib/Deserializer/functions.php",
                    "lib/Serializer/functions.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Evert Pot",
                    "email": "me@evertpot.com",
                    "homepage": "http://evertpot.com/",
                    "role": "Developer"
                },
                {
                    "name": "Markus Staab",
                    "email": "markus.staab@redaxo.de",
                    "role": "Developer"
                }
            ],
            "description": "sabre/xml is an XML library that you may not hate.",
            "homepage": "https://sabre.io/xml/",
            "keywords": [
                "XMLReader",
                "XMLWriter",
                "dom",
                "xml"
            ],
            "time": "2019-01-09T13:51:57+00:00"
        },
        {
            "name": "setasign/fpdi",
            "version": "1.6.2",
            "source": {
                "type": "git",
                "url": "https://github.com/Setasign/FPDI.git",
                "reference": "a6ad58897a6d97cc2d2cd2adaeda343b25a368ea"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/Setasign/FPDI/zipball/a6ad58897a6d97cc2d2cd2adaeda343b25a368ea",
                "reference": "a6ad58897a6d97cc2d2cd2adaeda343b25a368ea",
                "shasum": ""
            },
            "suggest": {
                "setasign/fpdf": "FPDI will extend this class but as it is also possible to use \"tecnickcom/tcpdf\" as an alternative there's no fixed dependency configured.",
                "setasign/fpdi-fpdf": "Use this package to automatically evaluate dependencies to FPDF.",
                "setasign/fpdi-tcpdf": "Use this package to automatically evaluate dependencies to TCPDF."
            },
            "type": "library",
            "autoload": {
                "classmap": [
                    "filters/",
                    "fpdi.php",
                    "fpdf_tpl.php",
                    "fpdi_pdf_parser.php",
                    "pdf_context.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Jan Slabon",
                    "email": "jan.slabon@setasign.com",
                    "homepage": "https://www.setasign.com"
                }
            ],
            "description": "FPDI is a collection of PHP classes facilitating developers to read pages from existing PDF documents and use them as templates in FPDF. Because it is also possible to use FPDI with TCPDF, there are no fixed dependencies defined. Please see suggestions for packages which evaluates the dependencies automatically.",
            "homepage": "https://www.setasign.com/fpdi",
            "keywords": [
                "fpdf",
                "fpdi",
                "pdf"
            ],
            "time": "2017-05-11T14:25:49+00:00"
        },
        {
            "name": "setasign/fpdi-tcpdf",
            "version": "1.6.1",
            "source": {
                "type": "git",
                "url": "https://github.com/Setasign/FPDI-TCPDF.git",
                "reference": "871699d429fcbad6dc809d57c7c8026fd7f41e93"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/Setasign/FPDI-TCPDF/zipball/871699d429fcbad6dc809d57c7c8026fd7f41e93",
                "reference": "871699d429fcbad6dc809d57c7c8026fd7f41e93",
                "shasum": ""
            },
            "require": {
                "setasign/fpdi": "1.6.*",
                "tecnickcom/tcpdf": "6.2.*"
            },
            "conflict": {
                "setasign/fpdf": "1.*"
            },
            "type": "library",
            "autoload": {
                "classmap": [
                    "fpdi_bridge.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Jan Slabon",
                    "email": "jan.slabon@setasign.com",
                    "homepage": "https://www.setasign.com"
                }
            ],
            "description": "Kind of metadata package for dependencies of the latest versions of FPDI and TCPDF.",
            "homepage": "https://www.setasign.com/fpdi",
            "keywords": [
                "TCPDF",
                "fpdi",
                "pdf"
            ],
            "time": "2015-11-30T11:09:50+00:00"
        },
        {
            "name": "swiftmailer/swiftmailer",
            "version": "v6.2.1",
            "source": {
                "type": "git",
                "url": "https://github.com/swiftmailer/swiftmailer.git",
                "reference": "5397cd05b0a0f7937c47b0adcb4c60e5ab936b6a"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/swiftmailer/swiftmailer/zipball/5397cd05b0a0f7937c47b0adcb4c60e5ab936b6a",
                "reference": "5397cd05b0a0f7937c47b0adcb4c60e5ab936b6a",
                "shasum": ""
            },
            "require": {
                "egulias/email-validator": "~2.0",
                "php": ">=7.0.0",
                "symfony/polyfill-iconv": "^1.0",
                "symfony/polyfill-intl-idn": "^1.10",
                "symfony/polyfill-mbstring": "^1.0"
            },
            "require-dev": {
                "mockery/mockery": "~0.9.1",
                "symfony/phpunit-bridge": "^3.4.19|^4.1.8"
            },
            "suggest": {
                "ext-intl": "Needed to support internationalized email addresses",
                "true/punycode": "Needed to support internationalized email addresses, if ext-intl is not installed"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "6.2-dev"
                }
            },
            "autoload": {
                "files": [
                    "lib/swift_required.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Chris Corbyn"
                },
                {
                    "name": "Fabien Potencier",
                    "email": "fabien@symfony.com"
                }
            ],
            "description": "Swiftmailer, free feature-rich PHP mailer",
            "homepage": "https://swiftmailer.symfony.com",
            "keywords": [
                "email",
                "mail",
                "mailer"
            ],
            "time": "2019-04-21T09:21:45+00:00"
        },
        {
            "name": "symfony/polyfill-iconv",
            "version": "v1.11.0",
            "source": {
                "type": "git",
                "url": "https://github.com/symfony/polyfill-iconv.git",
                "reference": "f037ea22acfaee983e271dd9c3b8bb4150bd8ad7"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/symfony/polyfill-iconv/zipball/f037ea22acfaee983e271dd9c3b8bb4150bd8ad7",
                "reference": "f037ea22acfaee983e271dd9c3b8bb4150bd8ad7",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3.3"
            },
            "suggest": {
                "ext-iconv": "For best performance"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.11-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "Symfony\\Polyfill\\Iconv\\": ""
                },
                "files": [
                    "bootstrap.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Nicolas Grekas",
                    "email": "p@tchwork.com"
                },
                {
                    "name": "Symfony Community",
                    "homepage": "https://symfony.com/contributors"
                }
            ],
            "description": "Symfony polyfill for the Iconv extension",
            "homepage": "https://symfony.com",
            "keywords": [
                "compatibility",
                "iconv",
                "polyfill",
                "portable",
                "shim"
            ],
            "time": "2019-02-06T07:57:58+00:00"
        },
        {
            "name": "symfony/polyfill-intl-idn",
            "version": "v1.11.0",
            "source": {
                "type": "git",
                "url": "https://github.com/symfony/polyfill-intl-idn.git",
                "reference": "c766e95bec706cdd89903b1eda8afab7d7a6b7af"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/symfony/polyfill-intl-idn/zipball/c766e95bec706cdd89903b1eda8afab7d7a6b7af",
                "reference": "c766e95bec706cdd89903b1eda8afab7d7a6b7af",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3.3",
                "symfony/polyfill-mbstring": "^1.3",
                "symfony/polyfill-php72": "^1.9"
            },
            "suggest": {
                "ext-intl": "For best performance"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.9-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "Symfony\\Polyfill\\Intl\\Idn\\": ""
                },
                "files": [
                    "bootstrap.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Symfony Community",
                    "homepage": "https://symfony.com/contributors"
                },
                {
                    "name": "Laurent Bassin",
                    "email": "laurent@bassin.info"
                }
            ],
            "description": "Symfony polyfill for intl's idn_to_ascii and idn_to_utf8 functions",
            "homepage": "https://symfony.com",
            "keywords": [
                "compatibility",
                "idn",
                "intl",
                "polyfill",
                "portable",
                "shim"
            ],
            "time": "2019-03-04T13:44:35+00:00"
        },
        {
            "name": "symfony/polyfill-mbstring",
            "version": "v1.11.0",
            "source": {
                "type": "git",
                "url": "https://github.com/symfony/polyfill-mbstring.git",
                "reference": "fe5e94c604826c35a32fa832f35bd036b6799609"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/symfony/polyfill-mbstring/zipball/fe5e94c604826c35a32fa832f35bd036b6799609",
                "reference": "fe5e94c604826c35a32fa832f35bd036b6799609",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3.3"
            },
            "suggest": {
                "ext-mbstring": "For best performance"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.11-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "Symfony\\Polyfill\\Mbstring\\": ""
                },
                "files": [
                    "bootstrap.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Nicolas Grekas",
                    "email": "p@tchwork.com"
                },
                {
                    "name": "Symfony Community",
                    "homepage": "https://symfony.com/contributors"
                }
            ],
            "description": "Symfony polyfill for the Mbstring extension",
            "homepage": "https://symfony.com",
            "keywords": [
                "compatibility",
                "mbstring",
                "polyfill",
                "portable",
                "shim"
            ],
            "time": "2019-02-06T07:57:58+00:00"
        },
        {
            "name": "symfony/polyfill-php72",
            "version": "v1.11.0",
            "source": {
                "type": "git",
                "url": "https://github.com/symfony/polyfill-php72.git",
                "reference": "ab50dcf166d5f577978419edd37aa2bb8eabce0c"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/symfony/polyfill-php72/zipball/ab50dcf166d5f577978419edd37aa2bb8eabce0c",
                "reference": "ab50dcf166d5f577978419edd37aa2bb8eabce0c",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3.3"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.11-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "Symfony\\Polyfill\\Php72\\": ""
                },
                "files": [
                    "bootstrap.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Nicolas Grekas",
                    "email": "p@tchwork.com"
                },
                {
                    "name": "Symfony Community",
                    "homepage": "https://symfony.com/contributors"
                }
            ],
            "description": "Symfony polyfill backporting some PHP 7.2+ features to lower PHP versions",
            "homepage": "https://symfony.com",
            "keywords": [
                "compatibility",
                "polyfill",
                "portable",
                "shim"
            ],
            "time": "2019-02-06T07:57:58+00:00"
        },
        {
            "name": "tecnickcom/tcpdf",
            "version": "6.2.21",
            "source": {
                "type": "git",
                "url": "https://github.com/tecnickcom/TCPDF.git",
                "reference": "a3273af312d032e317ad151a47d40691d796e5e5"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/tecnickcom/TCPDF/zipball/a3273af312d032e317ad151a47d40691d796e5e5",
                "reference": "a3273af312d032e317ad151a47d40691d796e5e5",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3.0"
            },
            "type": "library",
            "autoload": {
                "classmap": [
                    "config",
                    "include",
                    "tcpdf.php",
                    "tcpdf_parser.php",
                    "tcpdf_import.php",
                    "tcpdf_barcodes_1d.php",
                    "tcpdf_barcodes_2d.php",
                    "include/tcpdf_colors.php",
                    "include/tcpdf_filters.php",
                    "include/tcpdf_font_data.php",
                    "include/tcpdf_fonts.php",
                    "include/tcpdf_images.php",
                    "include/tcpdf_static.php",
                    "include/barcodes/datamatrix.php",
                    "include/barcodes/pdf417.php",
                    "include/barcodes/qrcode.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "LGPL-3.0"
            ],
            "authors": [
                {
                    "name": "Nicola Asuni",
                    "email": "info@tecnick.com",
                    "role": "lead"
                }
            ],
            "description": "TCPDF is a PHP class for generating PDF documents and barcodes.",
            "homepage": "http://www.tcpdf.org/",
            "keywords": [
                "PDFD32000-2008",
                "TCPDF",
                "barcodes",
                "datamatrix",
                "pdf",
                "pdf417",
                "qrcode"
            ],
            "time": "2018-09-14T13:49:09+00:00"
        }
    ],
    "packages-dev": [
        {
            "name": "doctrine/instantiator",
            "version": "1.0.5",
            "source": {
                "type": "git",
                "url": "https://github.com/doctrine/instantiator.git",
                "reference": "8e884e78f9f0eb1329e445619e04456e64d8051d"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/doctrine/instantiator/zipball/8e884e78f9f0eb1329e445619e04456e64d8051d",
                "reference": "8e884e78f9f0eb1329e445619e04456e64d8051d",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3,<8.0-DEV"
            },
            "require-dev": {
                "athletic/athletic": "~0.1.8",
                "ext-pdo": "*",
                "ext-phar": "*",
                "phpunit/phpunit": "~4.0",
                "squizlabs/php_codesniffer": "~2.0"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.0.x-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "Doctrine\\Instantiator\\": "src/Doctrine/Instantiator/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Marco Pivetta",
                    "email": "ocramius@gmail.com",
                    "homepage": "http://ocramius.github.com/"
                }
            ],
            "description": "A small, lightweight utility to instantiate objects in PHP without invoking their constructors",
            "homepage": "https://github.com/doctrine/instantiator",
            "keywords": [
                "constructor",
                "instantiate"
            ],
            "time": "2015-06-14T21:17:01+00:00"
        },
        {
            "name": "myclabs/deep-copy",
            "version": "1.7.0",
            "source": {
                "type": "git",
                "url": "https://github.com/myclabs/DeepCopy.git",
                "reference": "3b8a3a99ba1f6a3952ac2747d989303cbd6b7a3e"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/myclabs/DeepCopy/zipball/3b8a3a99ba1f6a3952ac2747d989303cbd6b7a3e",
                "reference": "3b8a3a99ba1f6a3952ac2747d989303cbd6b7a3e",
                "shasum": ""
            },
            "require": {
                "php": "^5.6 || ^7.0"
            },
            "require-dev": {
                "doctrine/collections": "^1.0",
                "doctrine/common": "^2.6",
                "phpunit/phpunit": "^4.1"
            },
            "type": "library",
            "autoload": {
                "psr-4": {
                    "DeepCopy\\": "src/DeepCopy/"
                },
                "files": [
                    "src/DeepCopy/deep_copy.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "description": "Create deep copies (clones) of your objects",
            "keywords": [
                "clone",
                "copy",
                "duplicate",
                "object",
                "object graph"
            ],
            "time": "2017-10-19T19:58:43+00:00"
        },
        {
            "name": "phar-io/manifest",
            "version": "1.0.1",
            "source": {
                "type": "git",
                "url": "https://github.com/phar-io/manifest.git",
                "reference": "2df402786ab5368a0169091f61a7c1e0eb6852d0"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/phar-io/manifest/zipball/2df402786ab5368a0169091f61a7c1e0eb6852d0",
                "reference": "2df402786ab5368a0169091f61a7c1e0eb6852d0",
                "shasum": ""
            },
            "require": {
                "ext-dom": "*",
                "ext-phar": "*",
                "phar-io/version": "^1.0.1",
                "php": "^5.6 || ^7.0"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.0.x-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Arne Blankerts",
                    "email": "arne@blankerts.de",
                    "role": "Developer"
                },
                {
                    "name": "Sebastian Heuer",
                    "email": "sebastian@phpeople.de",
                    "role": "Developer"
                },
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de",
                    "role": "Developer"
                }
            ],
            "description": "Component for reading phar.io manifest information from a PHP Archive (PHAR)",
            "time": "2017-03-05T18:14:27+00:00"
        },
        {
            "name": "phar-io/version",
            "version": "1.0.1",
            "source": {
                "type": "git",
                "url": "https://github.com/phar-io/version.git",
                "reference": "a70c0ced4be299a63d32fa96d9281d03e94041df"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/phar-io/version/zipball/a70c0ced4be299a63d32fa96d9281d03e94041df",
                "reference": "a70c0ced4be299a63d32fa96d9281d03e94041df",
                "shasum": ""
            },
            "require": {
                "php": "^5.6 || ^7.0"
            },
            "type": "library",
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Arne Blankerts",
                    "email": "arne@blankerts.de",
                    "role": "Developer"
                },
                {
                    "name": "Sebastian Heuer",
                    "email": "sebastian@phpeople.de",
                    "role": "Developer"
                },
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de",
                    "role": "Developer"
                }
            ],
            "description": "Library for handling version information and constraints",
            "time": "2017-03-05T17:38:23+00:00"
        },
        {
            "name": "phpdocumentor/reflection-common",
            "version": "1.0.1",
            "source": {
                "type": "git",
                "url": "https://github.com/phpDocumentor/ReflectionCommon.git",
                "reference": "21bdeb5f65d7ebf9f43b1b25d404f87deab5bfb6"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/phpDocumentor/ReflectionCommon/zipball/21bdeb5f65d7ebf9f43b1b25d404f87deab5bfb6",
                "reference": "21bdeb5f65d7ebf9f43b1b25d404f87deab5bfb6",
                "shasum": ""
            },
            "require": {
                "php": ">=5.5"
            },
            "require-dev": {
                "phpunit/phpunit": "^4.6"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.0.x-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "phpDocumentor\\Reflection\\": [
                        "src"
                    ]
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Jaap van Otterdijk",
                    "email": "opensource@ijaap.nl"
                }
            ],
            "description": "Common reflection classes used by phpdocumentor to reflect the code structure",
            "homepage": "http://www.phpdoc.org",
            "keywords": [
                "FQSEN",
                "phpDocumentor",
                "phpdoc",
                "reflection",
                "static analysis"
            ],
            "time": "2017-09-11T18:02:19+00:00"
        },
        {
            "name": "phpdocumentor/reflection-docblock",
            "version": "4.3.1",
            "source": {
                "type": "git",
                "url": "https://github.com/phpDocumentor/ReflectionDocBlock.git",
                "reference": "bdd9f737ebc2a01c06ea7ff4308ec6697db9b53c"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/phpDocumentor/ReflectionDocBlock/zipball/bdd9f737ebc2a01c06ea7ff4308ec6697db9b53c",
                "reference": "bdd9f737ebc2a01c06ea7ff4308ec6697db9b53c",
                "shasum": ""
            },
            "require": {
                "php": "^7.0",
                "phpdocumentor/reflection-common": "^1.0.0",
                "phpdocumentor/type-resolver": "^0.4.0",
                "webmozart/assert": "^1.0"
            },
            "require-dev": {
                "doctrine/instantiator": "~1.0.5",
                "mockery/mockery": "^1.0",
                "phpunit/phpunit": "^6.4"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "4.x-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "phpDocumentor\\Reflection\\": [
                        "src/"
                    ]
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Mike van Riel",
                    "email": "me@mikevanriel.com"
                }
            ],
            "description": "With this component, a library can provide support for annotations via DocBlocks or otherwise retrieve information that is embedded in a DocBlock.",
            "time": "2019-04-30T17:48:53+00:00"
        },
        {
            "name": "phpdocumentor/type-resolver",
            "version": "0.4.0",
            "source": {
                "type": "git",
                "url": "https://github.com/phpDocumentor/TypeResolver.git",
                "reference": "9c977708995954784726e25d0cd1dddf4e65b0f7"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/phpDocumentor/TypeResolver/zipball/9c977708995954784726e25d0cd1dddf4e65b0f7",
                "reference": "9c977708995954784726e25d0cd1dddf4e65b0f7",
                "shasum": ""
            },
            "require": {
                "php": "^5.5 || ^7.0",
                "phpdocumentor/reflection-common": "^1.0"
            },
            "require-dev": {
                "mockery/mockery": "^0.9.4",
                "phpunit/phpunit": "^5.2||^4.8.24"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.0.x-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "phpDocumentor\\Reflection\\": [
                        "src/"
                    ]
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Mike van Riel",
                    "email": "me@mikevanriel.com"
                }
            ],
            "time": "2017-07-14T14:27:02+00:00"
        },
        {
            "name": "phpspec/prophecy",
            "version": "1.8.0",
            "source": {
                "type": "git",
                "url": "https://github.com/phpspec/prophecy.git",
                "reference": "4ba436b55987b4bf311cb7c6ba82aa528aac0a06"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/phpspec/prophecy/zipball/4ba436b55987b4bf311cb7c6ba82aa528aac0a06",
                "reference": "4ba436b55987b4bf311cb7c6ba82aa528aac0a06",
                "shasum": ""
            },
            "require": {
                "doctrine/instantiator": "^1.0.2",
                "php": "^5.3|^7.0",
                "phpdocumentor/reflection-docblock": "^2.0|^3.0.2|^4.0",
                "sebastian/comparator": "^1.1|^2.0|^3.0",
                "sebastian/recursion-context": "^1.0|^2.0|^3.0"
            },
            "require-dev": {
                "phpspec/phpspec": "^2.5|^3.2",
                "phpunit/phpunit": "^4.8.35 || ^5.7 || ^6.5 || ^7.1"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.8.x-dev"
                }
            },
            "autoload": {
                "psr-0": {
                    "Prophecy\\": "src/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Konstantin Kudryashov",
                    "email": "ever.zet@gmail.com",
                    "homepage": "http://everzet.com"
                },
                {
                    "name": "Marcello Duarte",
                    "email": "marcello.duarte@gmail.com"
                }
            ],
            "description": "Highly opinionated mocking framework for PHP 5.3+",
            "homepage": "https://github.com/phpspec/prophecy",
            "keywords": [
                "Double",
                "Dummy",
                "fake",
                "mock",
                "spy",
                "stub"
            ],
            "time": "2018-08-05T17:53:17+00:00"
        },
        {
            "name": "phpunit/php-code-coverage",
            "version": "5.3.2",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/php-code-coverage.git",
                "reference": "c89677919c5dd6d3b3852f230a663118762218ac"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/php-code-coverage/zipball/c89677919c5dd6d3b3852f230a663118762218ac",
                "reference": "c89677919c5dd6d3b3852f230a663118762218ac",
                "shasum": ""
            },
            "require": {
                "ext-dom": "*",
                "ext-xmlwriter": "*",
                "php": "^7.0",
                "phpunit/php-file-iterator": "^1.4.2",
                "phpunit/php-text-template": "^1.2.1",
                "phpunit/php-token-stream": "^2.0.1",
                "sebastian/code-unit-reverse-lookup": "^1.0.1",
                "sebastian/environment": "^3.0",
                "sebastian/version": "^2.0.1",
                "theseer/tokenizer": "^1.1"
            },
            "require-dev": {
                "phpunit/phpunit": "^6.0"
            },
            "suggest": {
                "ext-xdebug": "^2.5.5"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "5.3.x-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de",
                    "role": "lead"
                }
            ],
            "description": "Library that provides collection, processing, and rendering functionality for PHP code coverage information.",
            "homepage": "https://github.com/sebastianbergmann/php-code-coverage",
            "keywords": [
                "coverage",
                "testing",
                "xunit"
            ],
            "time": "2018-04-06T15:36:58+00:00"
        },
        {
            "name": "phpunit/php-file-iterator",
            "version": "1.4.5",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/php-file-iterator.git",
                "reference": "730b01bc3e867237eaac355e06a36b85dd93a8b4"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/php-file-iterator/zipball/730b01bc3e867237eaac355e06a36b85dd93a8b4",
                "reference": "730b01bc3e867237eaac355e06a36b85dd93a8b4",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3.3"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.4.x-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sb@sebastian-bergmann.de",
                    "role": "lead"
                }
            ],
            "description": "FilterIterator implementation that filters files based on a list of suffixes.",
            "homepage": "https://github.com/sebastianbergmann/php-file-iterator/",
            "keywords": [
                "filesystem",
                "iterator"
            ],
            "time": "2017-11-27T13:52:08+00:00"
        },
        {
            "name": "phpunit/php-text-template",
            "version": "1.2.1",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/php-text-template.git",
                "reference": "31f8b717e51d9a2afca6c9f046f5d69fc27c8686"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/php-text-template/zipball/31f8b717e51d9a2afca6c9f046f5d69fc27c8686",
                "reference": "31f8b717e51d9a2afca6c9f046f5d69fc27c8686",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3.3"
            },
            "type": "library",
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de",
                    "role": "lead"
                }
            ],
            "description": "Simple template engine.",
            "homepage": "https://github.com/sebastianbergmann/php-text-template/",
            "keywords": [
                "template"
            ],
            "time": "2015-06-21T13:50:34+00:00"
        },
        {
            "name": "phpunit/php-timer",
            "version": "1.0.9",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/php-timer.git",
                "reference": "3dcf38ca72b158baf0bc245e9184d3fdffa9c46f"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/php-timer/zipball/3dcf38ca72b158baf0bc245e9184d3fdffa9c46f",
                "reference": "3dcf38ca72b158baf0bc245e9184d3fdffa9c46f",
                "shasum": ""
            },
            "require": {
                "php": "^5.3.3 || ^7.0"
            },
            "require-dev": {
                "phpunit/phpunit": "^4.8.35 || ^5.7 || ^6.0"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.0-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sb@sebastian-bergmann.de",
                    "role": "lead"
                }
            ],
            "description": "Utility class for timing",
            "homepage": "https://github.com/sebastianbergmann/php-timer/",
            "keywords": [
                "timer"
            ],
            "time": "2017-02-26T11:10:40+00:00"
        },
        {
            "name": "phpunit/php-token-stream",
            "version": "2.0.2",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/php-token-stream.git",
                "reference": "791198a2c6254db10131eecfe8c06670700904db"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/php-token-stream/zipball/791198a2c6254db10131eecfe8c06670700904db",
                "reference": "791198a2c6254db10131eecfe8c06670700904db",
                "shasum": ""
            },
            "require": {
                "ext-tokenizer": "*",
                "php": "^7.0"
            },
            "require-dev": {
                "phpunit/phpunit": "^6.2.4"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "2.0-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de"
                }
            ],
            "description": "Wrapper around PHP's tokenizer extension.",
            "homepage": "https://github.com/sebastianbergmann/php-token-stream/",
            "keywords": [
                "tokenizer"
            ],
            "time": "2017-11-27T05:48:46+00:00"
        },
        {
            "name": "phpunit/phpunit",
            "version": "6.5.14",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/phpunit.git",
                "reference": "bac23fe7ff13dbdb461481f706f0e9fe746334b7"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/phpunit/zipball/bac23fe7ff13dbdb461481f706f0e9fe746334b7",
                "reference": "bac23fe7ff13dbdb461481f706f0e9fe746334b7",
                "shasum": ""
            },
            "require": {
                "ext-dom": "*",
                "ext-json": "*",
                "ext-libxml": "*",
                "ext-mbstring": "*",
                "ext-xml": "*",
                "myclabs/deep-copy": "^1.6.1",
                "phar-io/manifest": "^1.0.1",
                "phar-io/version": "^1.0",
                "php": "^7.0",
                "phpspec/prophecy": "^1.7",
                "phpunit/php-code-coverage": "^5.3",
                "phpunit/php-file-iterator": "^1.4.3",
                "phpunit/php-text-template": "^1.2.1",
                "phpunit/php-timer": "^1.0.9",
                "phpunit/phpunit-mock-objects": "^5.0.9",
                "sebastian/comparator": "^2.1",
                "sebastian/diff": "^2.0",
                "sebastian/environment": "^3.1",
                "sebastian/exporter": "^3.1",
                "sebastian/global-state": "^2.0",
                "sebastian/object-enumerator": "^3.0.3",
                "sebastian/resource-operations": "^1.0",
                "sebastian/version": "^2.0.1"
            },
            "conflict": {
                "phpdocumentor/reflection-docblock": "3.0.2",
                "phpunit/dbunit": "<3.0"
            },
            "require-dev": {
                "ext-pdo": "*"
            },
            "suggest": {
                "ext-xdebug": "*",
                "phpunit/php-invoker": "^1.1"
            },
            "bin": [
                "phpunit"
            ],
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "6.5.x-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de",
                    "role": "lead"
                }
            ],
            "description": "The PHP Unit Testing framework.",
            "homepage": "https://phpunit.de/",
            "keywords": [
                "phpunit",
                "testing",
                "xunit"
            ],
            "time": "2019-02-01T05:22:47+00:00"
        },
        {
            "name": "phpunit/phpunit-mock-objects",
            "version": "5.0.10",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/phpunit-mock-objects.git",
                "reference": "cd1cf05c553ecfec36b170070573e540b67d3f1f"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/phpunit-mock-objects/zipball/cd1cf05c553ecfec36b170070573e540b67d3f1f",
                "reference": "cd1cf05c553ecfec36b170070573e540b67d3f1f",
                "shasum": ""
            },
            "require": {
                "doctrine/instantiator": "^1.0.5",
                "php": "^7.0",
                "phpunit/php-text-template": "^1.2.1",
                "sebastian/exporter": "^3.1"
            },
            "conflict": {
                "phpunit/phpunit": "<6.0"
            },
            "require-dev": {
                "phpunit/phpunit": "^6.5.11"
            },
            "suggest": {
                "ext-soap": "*"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "5.0.x-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de",
                    "role": "lead"
                }
            ],
            "description": "Mock Object library for PHPUnit",
            "homepage": "https://github.com/sebastianbergmann/phpunit-mock-objects/",
            "keywords": [
                "mock",
                "xunit"
            ],
            "abandoned": true,
            "time": "2018-08-09T05:50:03+00:00"
        },
        {
            "name": "sebastian/code-unit-reverse-lookup",
            "version": "1.0.1",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/code-unit-reverse-lookup.git",
                "reference": "4419fcdb5eabb9caa61a27c7a1db532a6b55dd18"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/code-unit-reverse-lookup/zipball/4419fcdb5eabb9caa61a27c7a1db532a6b55dd18",
                "reference": "4419fcdb5eabb9caa61a27c7a1db532a6b55dd18",
                "shasum": ""
            },
            "require": {
                "php": "^5.6 || ^7.0"
            },
            "require-dev": {
                "phpunit/phpunit": "^5.7 || ^6.0"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.0.x-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de"
                }
            ],
            "description": "Looks up which function or method a line of code belongs to",
            "homepage": "https://github.com/sebastianbergmann/code-unit-reverse-lookup/",
            "time": "2017-03-04T06:30:41+00:00"
        },
        {
            "name": "sebastian/comparator",
            "version": "2.1.3",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/comparator.git",
                "reference": "34369daee48eafb2651bea869b4b15d75ccc35f9"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/comparator/zipball/34369daee48eafb2651bea869b4b15d75ccc35f9",
                "reference": "34369daee48eafb2651bea869b4b15d75ccc35f9",
                "shasum": ""
            },
            "require": {
                "php": "^7.0",
                "sebastian/diff": "^2.0 || ^3.0",
                "sebastian/exporter": "^3.1"
            },
            "require-dev": {
                "phpunit/phpunit": "^6.4"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "2.1.x-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Jeff Welch",
                    "email": "whatthejeff@gmail.com"
                },
                {
                    "name": "Volker Dusch",
                    "email": "github@wallbash.com"
                },
                {
                    "name": "Bernhard Schussek",
                    "email": "bschussek@2bepublished.at"
                },
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de"
                }
            ],
            "description": "Provides the functionality to compare PHP values for equality",
            "homepage": "https://github.com/sebastianbergmann/comparator",
            "keywords": [
                "comparator",
                "compare",
                "equality"
            ],
            "time": "2018-02-01T13:46:46+00:00"
        },
        {
            "name": "sebastian/diff",
            "version": "2.0.1",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/diff.git",
                "reference": "347c1d8b49c5c3ee30c7040ea6fc446790e6bddd"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/diff/zipball/347c1d8b49c5c3ee30c7040ea6fc446790e6bddd",
                "reference": "347c1d8b49c5c3ee30c7040ea6fc446790e6bddd",
                "shasum": ""
            },
            "require": {
                "php": "^7.0"
            },
            "require-dev": {
                "phpunit/phpunit": "^6.2"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "2.0-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Kore Nordmann",
                    "email": "mail@kore-nordmann.de"
                },
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de"
                }
            ],
            "description": "Diff implementation",
            "homepage": "https://github.com/sebastianbergmann/diff",
            "keywords": [
                "diff"
            ],
            "time": "2017-08-03T08:09:46+00:00"
        },
        {
            "name": "sebastian/environment",
            "version": "3.1.0",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/environment.git",
                "reference": "cd0871b3975fb7fc44d11314fd1ee20925fce4f5"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/environment/zipball/cd0871b3975fb7fc44d11314fd1ee20925fce4f5",
                "reference": "cd0871b3975fb7fc44d11314fd1ee20925fce4f5",
                "shasum": ""
            },
            "require": {
                "php": "^7.0"
            },
            "require-dev": {
                "phpunit/phpunit": "^6.1"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "3.1.x-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de"
                }
            ],
            "description": "Provides functionality to handle HHVM/PHP environments",
            "homepage": "http://www.github.com/sebastianbergmann/environment",
            "keywords": [
                "Xdebug",
                "environment",
                "hhvm"
            ],
            "time": "2017-07-01T08:51:00+00:00"
        },
        {
            "name": "sebastian/exporter",
            "version": "3.1.0",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/exporter.git",
                "reference": "234199f4528de6d12aaa58b612e98f7d36adb937"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/exporter/zipball/234199f4528de6d12aaa58b612e98f7d36adb937",
                "reference": "234199f4528de6d12aaa58b612e98f7d36adb937",
                "shasum": ""
            },
            "require": {
                "php": "^7.0",
                "sebastian/recursion-context": "^3.0"
            },
            "require-dev": {
                "ext-mbstring": "*",
                "phpunit/phpunit": "^6.0"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "3.1.x-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Jeff Welch",
                    "email": "whatthejeff@gmail.com"
                },
                {
                    "name": "Volker Dusch",
                    "email": "github@wallbash.com"
                },
                {
                    "name": "Bernhard Schussek",
                    "email": "bschussek@2bepublished.at"
                },
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de"
                },
                {
                    "name": "Adam Harvey",
                    "email": "aharvey@php.net"
                }
            ],
            "description": "Provides the functionality to export PHP variables for visualization",
            "homepage": "http://www.github.com/sebastianbergmann/exporter",
            "keywords": [
                "export",
                "exporter"
            ],
            "time": "2017-04-03T13:19:02+00:00"
        },
        {
            "name": "sebastian/global-state",
            "version": "2.0.0",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/global-state.git",
                "reference": "e8ba02eed7bbbb9e59e43dedd3dddeff4a56b0c4"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/global-state/zipball/e8ba02eed7bbbb9e59e43dedd3dddeff4a56b0c4",
                "reference": "e8ba02eed7bbbb9e59e43dedd3dddeff4a56b0c4",
                "shasum": ""
            },
            "require": {
                "php": "^7.0"
            },
            "require-dev": {
                "phpunit/phpunit": "^6.0"
            },
            "suggest": {
                "ext-uopz": "*"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "2.0-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de"
                }
            ],
            "description": "Snapshotting of global state",
            "homepage": "http://www.github.com/sebastianbergmann/global-state",
            "keywords": [
                "global state"
            ],
            "time": "2017-04-27T15:39:26+00:00"
        },
        {
            "name": "sebastian/object-enumerator",
            "version": "3.0.3",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/object-enumerator.git",
                "reference": "7cfd9e65d11ffb5af41198476395774d4c8a84c5"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/object-enumerator/zipball/7cfd9e65d11ffb5af41198476395774d4c8a84c5",
                "reference": "7cfd9e65d11ffb5af41198476395774d4c8a84c5",
                "shasum": ""
            },
            "require": {
                "php": "^7.0",
                "sebastian/object-reflector": "^1.1.1",
                "sebastian/recursion-context": "^3.0"
            },
            "require-dev": {
                "phpunit/phpunit": "^6.0"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "3.0.x-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de"
                }
            ],
            "description": "Traverses array structures and object graphs to enumerate all referenced objects",
            "homepage": "https://github.com/sebastianbergmann/object-enumerator/",
            "time": "2017-08-03T12:35:26+00:00"
        },
        {
            "name": "sebastian/object-reflector",
            "version": "1.1.1",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/object-reflector.git",
                "reference": "773f97c67f28de00d397be301821b06708fca0be"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/object-reflector/zipball/773f97c67f28de00d397be301821b06708fca0be",
                "reference": "773f97c67f28de00d397be301821b06708fca0be",
                "shasum": ""
            },
            "require": {
                "php": "^7.0"
            },
            "require-dev": {
                "phpunit/phpunit": "^6.0"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.1-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de"
                }
            ],
            "description": "Allows reflection of object attributes, including inherited and non-public ones",
            "homepage": "https://github.com/sebastianbergmann/object-reflector/",
            "time": "2017-03-29T09:07:27+00:00"
        },
        {
            "name": "sebastian/recursion-context",
            "version": "3.0.0",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/recursion-context.git",
                "reference": "5b0cd723502bac3b006cbf3dbf7a1e3fcefe4fa8"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/recursion-context/zipball/5b0cd723502bac3b006cbf3dbf7a1e3fcefe4fa8",
                "reference": "5b0cd723502bac3b006cbf3dbf7a1e3fcefe4fa8",
                "shasum": ""
            },
            "require": {
                "php": "^7.0"
            },
            "require-dev": {
                "phpunit/phpunit": "^6.0"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "3.0.x-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Jeff Welch",
                    "email": "whatthejeff@gmail.com"
                },
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de"
                },
                {
                    "name": "Adam Harvey",
                    "email": "aharvey@php.net"
                }
            ],
            "description": "Provides functionality to recursively process PHP variables",
            "homepage": "http://www.github.com/sebastianbergmann/recursion-context",
            "time": "2017-03-03T06:23:57+00:00"
        },
        {
            "name": "sebastian/resource-operations",
            "version": "1.0.0",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/resource-operations.git",
                "reference": "ce990bb21759f94aeafd30209e8cfcdfa8bc3f52"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/resource-operations/zipball/ce990bb21759f94aeafd30209e8cfcdfa8bc3f52",
                "reference": "ce990bb21759f94aeafd30209e8cfcdfa8bc3f52",
                "shasum": ""
            },
            "require": {
                "php": ">=5.6.0"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.0.x-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de"
                }
            ],
            "description": "Provides a list of PHP built-in functions that operate on resources",
            "homepage": "https://www.github.com/sebastianbergmann/resource-operations",
            "time": "2015-07-28T20:34:47+00:00"
        },
        {
            "name": "sebastian/version",
            "version": "2.0.1",
            "source": {
                "type": "git",
                "url": "https://github.com/sebastianbergmann/version.git",
                "reference": "99732be0ddb3361e16ad77b68ba41efc8e979019"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/sebastianbergmann/version/zipball/99732be0ddb3361e16ad77b68ba41efc8e979019",
                "reference": "99732be0ddb3361e16ad77b68ba41efc8e979019",
                "shasum": ""
            },
            "require": {
                "php": ">=5.6"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "2.0.x-dev"
                }
            },
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Sebastian Bergmann",
                    "email": "sebastian@phpunit.de",
                    "role": "lead"
                }
            ],
            "description": "Library that helps with managing the version number of Git-hosted PHP projects",
            "homepage": "https://github.com/sebastianbergmann/version",
            "time": "2016-10-03T07:35:21+00:00"
        },
        {
            "name": "symfony/polyfill-ctype",
            "version": "v1.11.0",
            "source": {
                "type": "git",
                "url": "https://github.com/symfony/polyfill-ctype.git",
                "reference": "82ebae02209c21113908c229e9883c419720738a"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/symfony/polyfill-ctype/zipball/82ebae02209c21113908c229e9883c419720738a",
                "reference": "82ebae02209c21113908c229e9883c419720738a",
                "shasum": ""
            },
            "require": {
                "php": ">=5.3.3"
            },
            "suggest": {
                "ext-ctype": "For best performance"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.11-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "Symfony\\Polyfill\\Ctype\\": ""
                },
                "files": [
                    "bootstrap.php"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Symfony Community",
                    "homepage": "https://symfony.com/contributors"
                },
                {
                    "name": "Gert de Pagter",
                    "email": "backendtea@gmail.com"
                }
            ],
            "description": "Symfony polyfill for ctype functions",
            "homepage": "https://symfony.com",
            "keywords": [
                "compatibility",
                "ctype",
                "polyfill",
                "portable"
            ],
            "time": "2019-02-06T07:57:58+00:00"
        },
        {
            "name": "theseer/tokenizer",
            "version": "1.1.2",
            "source": {
                "type": "git",
                "url": "https://github.com/theseer/tokenizer.git",
                "reference": "1c42705be2b6c1de5904f8afacef5895cab44bf8"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/theseer/tokenizer/zipball/1c42705be2b6c1de5904f8afacef5895cab44bf8",
                "reference": "1c42705be2b6c1de5904f8afacef5895cab44bf8",
                "shasum": ""
            },
            "require": {
                "ext-dom": "*",
                "ext-tokenizer": "*",
                "ext-xmlwriter": "*",
                "php": "^7.0"
            },
            "type": "library",
            "autoload": {
                "classmap": [
                    "src/"
                ]
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "BSD-3-Clause"
            ],
            "authors": [
                {
                    "name": "Arne Blankerts",
                    "email": "arne@blankerts.de",
                    "role": "Developer"
                }
            ],
            "description": "A small library for converting tokenized PHP source code into XML and potentially other formats",
            "time": "2019-04-04T09:56:43+00:00"
        },
        {
            "name": "webmozart/assert",
            "version": "1.4.0",
            "source": {
                "type": "git",
                "url": "https://github.com/webmozart/assert.git",
                "reference": "83e253c8e0be5b0257b881e1827274667c5c17a9"
            },
            "dist": {
                "type": "zip",
                "url": "https://api.github.com/repos/webmozart/assert/zipball/83e253c8e0be5b0257b881e1827274667c5c17a9",
                "reference": "83e253c8e0be5b0257b881e1827274667c5c17a9",
                "shasum": ""
            },
            "require": {
                "php": "^5.3.3 || ^7.0",
                "symfony/polyfill-ctype": "^1.8"
            },
            "require-dev": {
                "phpunit/phpunit": "^4.6",
                "sebastian/version": "^1.0.1"
            },
            "type": "library",
            "extra": {
                "branch-alias": {
                    "dev-master": "1.3-dev"
                }
            },
            "autoload": {
                "psr-4": {
                    "Webmozart\\Assert\\": "src/"
                }
            },
            "notification-url": "https://packagist.org/downloads/",
            "license": [
                "MIT"
            ],
            "authors": [
                {
                    "name": "Bernhard Schussek",
                    "email": "bschussek@gmail.com"
                }
            ],
            "description": "Assertions to validate method input/output with nice error messages.",
            "keywords": [
                "assert",
                "check",
                "validate"
            ],
            "time": "2018-12-25T11:19:39+00:00"
        }
    ],
    "aliases": [],
    "minimum-stability": "stable",
    "stability-flags": {
        "swiftmailer/swiftmailer": 0
    },
    "prefer-stable": false,
    "prefer-lowest": false,
    "platform": {
        "php": ">=7.0.0",
        "ext-pcre": "*",
        "ext-mbstring": "*",
        "ext-ctype": "*",
        "ext-date": "*",
        "ext-iconv": "*",
        "ext-curl": "*",
        "ext-zip": "*",
        "ext-soap": "*",
        "ext-gd": "*",
        "ext-pdo": "*",
        "ext-pdo_mysql": "*",
        "ext-calendar": "*",
        "ext-xml": "*",
        "ext-intl": "*"
    },
    "platform-dev": []
}



    
