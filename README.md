# hello-world#
Traceback (most recent call last):
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\nose\case.py", line 198, in runTest
    self.test(*self.arg)
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\nose\util.py", line 620, in newfunc
    return func(*arg, **kw)
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\django\test\utils.py", line 111, in setup_test_environment
    "setup_test_environment() was already called and can't be called "
RuntimeError: setup_test_environment() was already called and can't be called again without first calling teardown_test_environment().
Traceback (most recent call last):
  File "manage.py", line 21, in <module>
    main()
  File "manage.py", line 17, in main
    execute_from_command_line(sys.argv)
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\django\core\management\__init__.py", line 381, in execute_from_command_line
    utility.execute()
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\django\core\management\__init__.py", line 375, in execute
    self.fetch_command(subcommand).run_from_argv(self.argv)
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\django\core\management\commands\test.py", line 23, in run_from_argv
    super().run_from_argv(argv)
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\django\core\management\base.py", line 323, in run_from_argv
    self.execute(*args, **cmd_options)
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\django\core\management\base.py", line 364, in execute
    output = self.handle(*args, **options)
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\django\core\management\commands\test.py", line 53, in handle
    failures = test_runner.run_tests(test_labels)
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\django_nose\runner.py", line 308, in run_tests
    result = self.run_suite(nose_argv)
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\django_nose\runner.py", line 245, in run_suite
    addplugins=plugins_to_add)
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\nose\core.py", line 121, in __init__
    **extra_args)
  File "C:\Program Files\Python37\Lib\unittest\main.py", line 101, in __init__
    self.runTests()
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\nose\core.py", line 207, in runTests
    result = self.testRunner.run(self.test)
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\nose\core.py", line 66, in run
    result.printErrors()
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\nose\plugins\manager.py", line 99, in __call__
    return self.call(*arg, **kw)
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\nose\plugins\manager.py", line 167, in simple
    result = meth(*arg, **kw)
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\django_nose\plugin.py", line 89, in finalize
    self.runner.teardown_test_environment()
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\django\test\runner.py", line 588, in teardown_test_environment
    teardown_test_environment()
  File "C:\Users\tay\.virtualenvs\relog-XSTo8UAo\lib\site-packages\django\test\utils.py", line 144, in teardown_test_environment
    saved_data = _TestState.saved_data
AttributeError: type object '_TestState
