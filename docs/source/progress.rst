.. _progress:

Progress Display
================

Rich can display continuously updated information regarding the progress of long running tasks / file copies etc. The information displayed is configurable, the default will display a description of the 'task', a progress bar, percentage complete, and estimated time remaining.

Rich progress display supports multiple tasks, each with a bar and progress information. You can use this to track concurrent tasks where the work is happening in threads or processes.

To see how the progress display looks, try this from the command line::

    python -m rich.progress

Basic Usage
-----------

For basic usage call the :func:`~rich.progress.track` function, which accepts a sequence (such as a list or range object) and an optional description of the job you are working on. The track method will yield values from the sequence and update the progress information on each iteration. Here's an example::

    from rich.progress import track

    for n in track(range(n), description="Processing..."):
        do_work(n)

Advanced usage
--------------

If you require multiple tasks in the display, or wish to configure the columns in the progress display, you can work directly with the :class:`~rich.progress.Progress` class. Once you have constructed a Progress object, add task(s) with (:meth:`~rich.progress.Progress.add_task`) and update progress with :meth:`~rich.progress.Progress.update`.

The Progress class is designed to be used as a *context manager* which will start and stop the progress display automatically.

Here's a simple example::

    from rich.progress import Progress

    with Progress() as progress:

        task1 = progress.add_task("[red]Downloading...", total=1000)
        task2 = progress.add_task("[green]Processing...", total=1000)
        task3 = progress.add_task("[cyan]Cooking...", total=1000)

        while not progress.finished:
            progress.update(task1, advance=0.5)
            progress.update(task2, advance=0.3)
            progress.update(task3, advance=0.9)
            time.sleep(0.02)

The ``total`` value associated with a task is the number of steps that must be completed for the progress to reach 100%. A *step* in this context is whatever makes sense for your application; it could be number of bytes of a file read, or number of images processed, etc.


Updating tasks
~~~~~~~~~~~~~~

When you call :meth:`~rich.progress.Progress.add_task` you get back a `Task ID`. Use this ID to call :meth:`~rich.progress.Progress.update` whenever you have completed some work, or any information has changed. Typically you will need to update ``completed`` every time you have completed a step. You can do this by updated ``completed`` directly or by setting ``advance`` which will add to the current ``completed`` value.

The :meth:`~rich.progress.Progress.update` method collects keyword arguments which are also associated with the task. Use this to supply any additional information you would like to render in the progress display. The additional arguments are stored in ``task.fields`` and  may be referenced in :ref:`Column classes<Columns>`.

Hiding tasks
~~~~~~~~~~~~

You can show or hide tasks by updating the tasks ``visible`` value. Tasks are visible by default, but you can also add a invisible task by calling :meth:`~rich.progress.Progress.add_task` with ``visible=False``.


Transient progress
~~~~~~~~~~~~~~~~~~

Normally when you exit the progress context manager (or call :meth:`~rich.progress.Progress.stop`) the last refreshed display remains in the terminal with the cursor on the following line. You can also make the progress display disappear on exit by setting ``transient=True`` on the Progress constructor. Here's an example::

    with Progress(transient=True) as progress:
        task = progress.add_task("Working", total=100)
        do_work(task)

Transient progress displays are useful if you want more minimal output in the terminal when tasks are complete.

Starting tasks
~~~~~~~~~~~~~~

When you add a task it is automatically *started* which means it will show a progress bar at 0% and the time remaining will be calculated from the current time. This may not work well if there is a long delay before you can start updating progress, you may need to wait for a response from a server, or count files in a directory (for example) before you can begin tracking progress. In these cases you can call :meth:`~rich.progress.Progress.add_task` with ``start=False`` which will display a pulsing animation that lets the user know something is working. When you have the number of steps you can call :meth:`~rich.progress.Progress.start_task` which will display the progress bar at 0%, then :meth:`~rich.progress.Progress.update` as normal.


Auto refresh
~~~~~~~~~~~~

By default, the progress information will refresh 10 times a second. Refreshing at a predictable rate can make numbers more readable if they are updating quickly. Auto refresh can also prevent excessive rendering to the terminal.

You can set the refresh rate with the ``refresh_per_second`` argument on the :class:`~rich.progress.Progress` constructor. You could set this to something lower than 10 if you know your updates will not be that frequent.

You can disable auto-refresh by setting ``auto_refresh=False`` on the constructor and call :meth:`~rich.progress.Progress.refresh` manually when there are updates to display.

Columns
~~~~~~~

You may customize the columns in the progress display with the positional arguments to the :class:`~rich.progress.Progress` constructor. The columns are specified as either a format string or a :class:`~rich.progress.ProgressColumn` object.

Format strings will be rendered with a single value `"task"` which will be a :class:`~rich.progress.Task` instance. For example ``"{task.description}"`` would display the task description in the column, and ``"{task.completed} of {task.total}"`` would display how many of the total steps have been completed.

The defaults are roughly equivalent to the following::

    progress = Progress(
        "[progress.description]{task.description}",
        BarColumn(),
        "[progress.percentage]{task.percentage:>3.0f}%",
        TimeRemainingColumn(),
    )

The following column objects are available:

- :class:`~rich.progress.BarColumn` Displays the bar.
- :class:`~rich.progress.TextColumn` Displays text.
- :class:`~rich.progress.TimeRemainingColumn` Displays the estimated time remaining.
- :class:`~rich.progress.FileSizeColumn` Displays progress as file size (assumes the steps are bytes).
- :class:`~rich.progress.TotalFileSizeColumn` Displays total file size (assumes the steps are bytes).
- :class:`~rich.progress.DownloadColumn` Displays download progress (assumes the steps are bytes).
- :class:`~rich.progress.TransferSpeedColumn` Displays transfer speed (assumes the steps are bytes.

To implement your own columns, extend the :class:`~rich.progress.Progress` and use it as you would the other columns.


Print / log
~~~~~~~~~~~

The Progress class will create an internal Console object which you can access via ``progress.console``. If you print or log to this console, the output will be displayed *above* the progress display. Here's an example::

    with Progress() as progress:
        task = progress.add_task(total=10)
        for job in range(10):
            progress.console.print("Working on job #{job}")
            run_job(job)
            progress.advance(task)

If you have another Console object you want to use, pass it in to the :class:`~rich.progress.Progress` constructor. Here's an example::

    from my_project import my_console

    with Progress(console=my_console) as progress:
        my_console.print("[bold blue]Starting work!")
        do_work(progress)
        

Redirecting stdout / stderr
~~~~~~~~~~~~~~~~~~~~~~~~~~~

To avoid breaking the progress display visuals, Rich will redirect ``stdout`` and ``stdout`` so that you can use the builtin ``print`` statement. This feature is enabled by default, but you can disable by setting ``redirect_stdout`` or ``redirect_stderr`` to ``False``


Customizing
~~~~~~~~~~~

If the :class:`~rich.progress.Progress` class doesn't offer exactly what you need in terms of a progress display, you can override the :class:`~rich.progress.Progress.get_renderables` method. For example, the following class will render a :class:`~rich.panel.Panel` around the progress display::

    from rich.panel import Panel
    from rich.progress import Progress

    class MyProgress(Progress):
        def get_renderables(self):
            yield Panel(self.make_tasks_table(self.tasks))            

Example
-------

See `downloader.py <https://github.com/willmcgugan/rich/blob/master/examples/downloader.py>`_ for a realistic application of a progress display. This script can download multiple concurrent files with a progress bar, transfer speed and file size.
