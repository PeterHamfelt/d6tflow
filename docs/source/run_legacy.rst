Running Tasks and Managing Workflows
==============================================

Previewing Task Execution Status
------------------------------------------------------------

Running a task will automatically run all the upstream dependencies. Before running a workflow, you can preview which tasks will be run.

.. code-block:: python

    d6tflow.preview(TaskTrain()) # single task
    d6tflow.preview([TaskPreprocess(),TaskTrain()]) # multiple tasks

Running Multiple Tasks as Workflows
------------------------------------------------------------

To run all tasks in a workflow, run the downstream task you want to complete. It will check if all the upstream dependencies are complete and if not it will run them intelligently for you.

.. code-block:: python

    d6tflow.run(TaskTrain()) # single task
    d6tflow.run([TaskPreprocess(),TaskTrain()]) # multiple tasks

If you get ``Error: luigi.worker.TaskException: Can not schedule non-task <class 'TaskTrain'>``, make sure you run an instantiated task object ``d6tflow.run(TaskTrain())`` not just the class ``d6tflow.run(TaskTrain)``.

How is a task marked complete?
------------------------------------------------------------

Tasks are complete when task output exists. This is typically the existance of a file, database table or cache. See :doc:`Task I/O Formats <../targets>` how task output is stored to understand what needs to exist for a task to be complete.

.. code-block:: python

    TaskTrain().complete() # status
    TaskTrain().output().path # where is output saved?
    TaskTrain().output()['output1'].path # multiple outputs

Task Completion with Parameters
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If a task has parameters, it needs to be run separately for each parameter to be complete when using different parameter settings.

.. code-block:: python

    d6tflow.run(TaskTrain()) # default param
    TaskTrain().complete() # True
    TaskTrain(do_preprocess).complete() # False

Disable Dependency Checks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

By default, for a task to be complete, it checks if all dependencies are complete also, not just the task itself. To check if just the task is complete without checking dependencies, set ``d6tflow.settings.check_dependencies=False``

.. code-block:: python

    TaskGetData().reset(confirm=False)
    d6tflow.settings.check_dependencies=True # default
    d6tflow.preview(TaskTrain()) # TaskGetData is pending so all tasks are pending
    '''
    └─--[TaskTrain-{'do_preprocess': 'True'} (PENDING)]
       └─--[TaskPreprocess-{'do_preprocess': 'True'} (PENDING)]
          └─--[TaskGetData-{} (PENDING)]
    '''
    d6tflow.settings.check_dependencies=False # deactivate dependency checks
    d6tflow.preview(TaskTrain())
    └─--[TaskTrain-{'do_preprocess': 'True'} (COMPLETE)]
       └─--[TaskPreprocess-{'do_preprocess': 'True'} (COMPLETE)]
          └─--[TaskGetData-{} (PENDING)]
    d6tflow.settings.check_dependencies=True # set to default


Debugging Failures
------------------------------------------------------------

If a task fails, it will show the stack trace. You need to look further up in the stack trace to find the line that caused the error. You can also set breakpoints in the task obviously.

::

    File "tasks.py", line 37, in run => error is here
        1/0
    ZeroDivisionError: division by zero

    [...] => look further up to find error

    ===== Luigi Execution Summary =====
    Scheduled 2 tasks of which:
    * 1 complete ones were encountered:
        - 1 TaskPreprocess(do_preprocess=True)
    * 1 failed:
        - 1 TaskTrain(do_preprocess=True)
    This progress looks :( because there were failed tasks
    ===== Luigi Execution Summary =====

     File
         raise RuntimeError('Exception found running flow, check trace')
    RuntimeError: Exception found running flow, check trace

    => look further up to find error


Rerun Tasks When You Make Changes
------------------------------------------------------------

You have several options to force tasks to reset and rerun. See sections below on how to handle parameter, data and code changes.

.. code-block:: python

    # force execution including downstream tasks
    d6tflow.run([TaskTrain()],force=[TaskGetData()])

    # reset single task
    TaskGetData().reset()

    # reset all downstream tasks
    d6tflow.invalidate_downstream(TaskGetData(), TaskTrain())

    # reset all upstream tasks
    d6tflow.invalidate_upstream(TaskTrain())


When to reset and rerun tasks?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Typically you want to reset and rerun tasks when:

* parameters changed
* data changed
* code changed

Handling Parameter Change
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As long as the parameter is defined in the task, d6tflow will automatically rerun tasks with different parameters.

.. code-block:: python

    d6tflow.run([TaskTrain(do_preprocess=True)]) # first experiment
    d6tflow.run([TaskTrain(do_preprocess=False)]) # another experiment

For d6tflow to intelligently figure out which tasks to rerun, the parameter has to be defined in the task. The downstream task (`TaskTrain`) has to pass on the parameter to the upstream task (`TaskPreprocess`).

.. code-block:: python

    class TaskGetData(d6tflow.tasks.TaskPqPandas):
    # no parameter dependence

    class TaskPreprocess(d6tflow.tasks.TaskCachePandas):  # save data in memory
        do_preprocess = luigi.BoolParameter(default=True) # parameter for preprocessing yes/no

    class TaskTrain(d6tflow.tasks.TaskPickle):
        # pass parameter upstream
        do_preprocess = luigi.BoolParameter(default=True)

        def requires(self):
            # pass parameter upstream
            return TaskPreprocess(do_preprocess=self.do_preprocess)

See [luigi docs for handling parameter inheritance](https://luigi.readthedocs.io/en/stable/api/luigi.util.html#using-inherits-and-requires-to-ease-parameter-pain)

Default Parameter Values in Config
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As an alternative to inheriting parameters, you can define defaults in a config files. When you change the config it will automatically rerun tasks. **The DOWNSIDE is that previously saved data will be overwritten!**

.. code-block:: python

    class TaskPreprocess(d6tflow.tasks.TaskCachePandas):
        do_preprocess = luigi.BoolParameter(default=cfg.do_preprocess) # store default in config


Avoid repeating parameters when referring to tasks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To run tasks and load their output for different parameters, you have to pass them to the task. Instead of hardcoding them each time, it is best to keep them in a dictionary and pass that to the task.

.. code-block:: python

    # avoid this
    d6tflow.run(TaskTrain(do_preprocess=False, model='nnet'))
    TaskTrain(do_preprocess=False, model='nnet').outputLoad()

    # better
    params = dict(do_preprocess=False, model='nnet')
    d6tflow.run(TaskTrain(**params))
    TaskTrain(**params).outputLoad()


Handling Data Change
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In future releases, d6tflow will automatically detect data changes. For now you have to manually reset tasks.

Handling Code Change
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Code changes likely lead to data changes. Code changes are difficult to detect and it is best if you manually force tasks to rerun.

Forcing a Single Task to Run
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can always run single tasks by calling the `run()` function. This is useful during debugging. However, this will only run this one task and not take care of any downstream dependencies.

.. code-block:: python

    # forcing execution
    TaskTrain().run()

Hiding Execution Output
------------------------------------------------------------

By default, the workflow execution summary is shown, because it shows important information which tasks were run and if any failed. At times, eg during deployment, it can be desirable to not show the execution output.

.. code-block:: python

    d6tflow.settings.execution_summary = False # global
    # or
    d6tflow.run(Task() ,execution_summary=False) # at each run

While typically not necessary, you can control change the log level to see additional log data. Default is ``WARNING``. It is a global setting, modify before you execute ``d6tflow.run()``.

.. code-block:: python

    d6tflow.settings.log_level = 'WARNING' # 'DEBUG', 'INFO', 'WARNING', 'ERROR', 'CRITICAL'

