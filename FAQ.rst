.. _frequently-asked-questions:

Frequently Asked Questions
==========================

Frequently asked questions on the usage of ``schedule``.

How to execute jobs in parallel?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*I am trying to execute 50 items every 10 seconds, but from the my logs it says it executes every item in 10 second schedule serially, is there a work around?*

By default, schedule executes all jobs serially. The reasoning behind this is that it would be difficult to find a model for parallel execution that makes everyone happy.

You can work around this restriction by running each of the jobs in its own thread:

.. code-block:: python

    import threading
    import time
    import schedule


    def job():
        print("I'm running on thread %s" % threading.current_thread())


    def run_threaded(job_func):
        job_thread = threading.Thread(target=job_func)
        job_thread.start()


    schedule.every(10).seconds.do(run_threaded, job)
    schedule.every(10).seconds.do(run_threaded, job)
    schedule.every(10).seconds.do(run_threaded, job)
    schedule.every(10).seconds.do(run_threaded, job)
    schedule.every(10).seconds.do(run_threaded, job)


    while 1:
        schedule.run_pending()
        time.sleep(1)

If you want tighter control on the number of threads use a shared jobqueue and one or more worker threads:

.. code-block:: python

    import Queue
    import time
    import threading
    import schedule


    def job():
        print("I'm working")


    def worker_main():
        while 1:
            job_func = jobqueue.get()
            job_func()
            jobqueue.task_done()

    jobqueue = Queue.Queue()

    schedule.every(10).seconds.do(jobqueue.put, job)
    schedule.every(10).seconds.do(jobqueue.put, job)
    schedule.every(10).seconds.do(jobqueue.put, job)
    schedule.every(10).seconds.do(jobqueue.put, job)
    schedule.every(10).seconds.do(jobqueue.put, job)

    worker_thread = threading.Thread(target=worker_main)
    worker_thread.start()

    while 1:
        schedule.run_pending()
        time.sleep(1)

This model also makes sense for a distributed application where the workers are separate processes that receive jobs from a distributed work queue. I like using beanstalkd with the beanstalkc Python library.

How to continuously run the scheduler without blocking the main thread?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Run the scheduler in a separate thread. Mrwhick wrote up a nice solution in to this problem `here <https://github.com/mrhwick/schedule/blob/master/schedule/__init__.py>`__ (look for ``run_continuously()``)

Does schedule support corutines?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Vanilla schedule technicly does support corutines.
It is supported (but discouraged) in way that in job you can create eventloop for running corutines run them and lose it on end of job 

Better solution is to run you corutines as tasks on scheduler loop that is already running as corutine as shown in example.

.. code-block:: python
    import asyncio
    import schedule
    import functools

    def asCoroTask(func,name=None):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            asyncio.create_task(func(*args,**kwargs))
        return wrapper


    @asCoroTask
    async def longRunningTask():
        print("starting long running io bound task")
        await asyncio.sleep(5) #pretend something usefull
        print("task done")

    async def corutineLoop():
        while 1:
            schedule.run_pending()
            print("running")
            await asyncio.sleep(1)


    schedule.every(1).seconds.do(longRunningTask)

    asyncio.run(corutineLoop())

When to use corutines as jobs?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- when you are using asyncio frameworks and want to exploit asyncio cooperative multitasking with.
- when you need handle os siglals and terminate scheduler loop immediately without waiting for sleep/wait to finished

note you code/including signal handlers may still delay other jobs by blocking, long runing and cpu bound, jobs
it may be sidesteped by running your tasks in thread/executor example:
https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.run_in_executor

Can scheduler loop delay handling signals if executed in main thread?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This referes to os signals like sigkill https://docs.python.org/3/library/signal.html.

Answer is it depends when signal is invoked. Worst case is durring sleep/wait between runs of schedule.run_pending().
Othervise execution of python would progress and virtual machine would notice flag that indicates it should execute signal handler and 
it would execute it. This also closely depends on what is used for timing runs of schedule.run_pending().

There are 3 for timing runs:
1) time.sleep - handler of signal is executed immediately but also time sleeps continue sleeping for remaining time
2) threading.event.wait - wait is actively blocking any executinon until timeout expores then signal handler is runned so you can not set event from handler
3) asyncio.wait with asyncio.Event - handler of signal is runned without regards of current wait. If handler sets event then loop terminate as soon as it is set. Example folowing code

corutine based loop
note (python will not exit untill all running corutines running threads and work submited to pool is finished)
.. code-block:: python
    import signal
    import asyncio
    import schedule
    async def cancelOrWait(event,sleepFor=1):
        try:
            return await asyncio.wait_for(event.wait(), timeout=sleepFor)
        except asyncio.TimeoutError:
            return False

    def setupEvent(*signals):
        terminateCondition = asyncio.Event()

        addSignal= None

        def universalTerminator(signum):
            print("terminating due "+str(signum))
            terminateCondition.set()
        
        from sys import platform

        if platform == "win32":
            def winAdd(signum):
                signal.signal(signum,lambda sig,frame: universalTerminator(signum))

            addSignal=winAdd
        else: ##needed because of https://docs.python.org/3/library/asyncio-eventloop.html#unix-signals
            #note this may be broken on non unix os like cygwin, riscos, atheos etc. Try use other variant
            loop=asyncio.get_running_loop()
            def unixAdd(signum):
                loop.add_signal_handler(signum, lambda: universalTerminator(signum))
            addSignal= unixAdd
        
        for sig in signals:
            addSignal(sig)
        
        return terminateCondition

    async def interuptableLoop(interuptEvent):
        while not await cancelOrWait(interuptEvent):
            print("runnining pending")
            schedule.run_pending()
        print("loop exited")

    async def main():
    await interuptableLoop(setupEvent(signal.SIGINT,signal.SIGABRT))

    asyncio.run(main())

Does schedule support timezones?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Vanilla schedule doesn't support timezones at the moment. If you need this functionality please check out @imiric's work `here <https://github.com/dbader/schedule/pull/16>`__. He added timezone support to schedule using python-dateutil.

What if my task throws an exception?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Schedule doesn't catch exceptions that happen during job execution. Therefore any exceptions thrown during job execution will bubble up and interrupt schedule's run_xyz function.

If you want to guard against exceptions you can wrap your job function
in a decorator like this:

.. code-block:: python

    import functools

    def catch_exceptions(cancel_on_failure=False):
        def catch_exceptions_decorator(job_func):
            @functools.wraps(job_func)
            def wrapper(*args, **kwargs):
                try:
                    return job_func(*args, **kwargs)
                except:
                    import traceback
                    print(traceback.format_exc())
                    if cancel_on_failure:
                        return schedule.CancelJob
            return wrapper
        return catch_exceptions_decorator

    @catch_exceptions(cancel_on_failure=True)
    def bad_task():
        return 1 / 0

    schedule.every(5).minutes.do(bad_task)

Another option would be to subclass Schedule like @mplewis did in `this example <https://gist.github.com/mplewis/8483f1c24f2d6259aef6>`_.

How can I run a job only once?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    def job_that_executes_once():
        # Do some work ...
        return schedule.CancelJob

    schedule.every().day.at('22:30').do(job_that_executes_once)


How can I cancel several jobs at once?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can cancel the scheduling of a group of jobs selecting them by a unique identifier.

.. code-block:: python

    def greet(name):
        print('Hello {}'.format(name))

    schedule.every().day.do(greet, 'Andrea').tag('daily-tasks', 'friend')
    schedule.every().hour.do(greet, 'John').tag('hourly-tasks', 'friend')
    schedule.every().hour.do(greet, 'Monica').tag('hourly-tasks', 'customer')
    schedule.every().day.do(greet, 'Derek').tag('daily-tasks', 'guest')

    schedule.clear('daily-tasks')

Will prevent every job tagged as ``daily-tasks`` from running again.


I'm getting an ``AttributeError: 'module' object has no attribute 'every'`` when I try to use schedule. How can I fix this?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This happens if your code imports the wrong ``schedule`` module. Make sure you don't have a ``schedule.py`` file in your project that overrides the ``schedule`` module provided by this library.

How can I add generic logging to my scheduled jobs?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The easiest way to add generic logging functionality to your schedule
job functions is to implement a decorator that handles logging
in a reusable way:

.. code-block:: python

    import functools
    import time

    import schedule


    # This decorator can be applied to
    def with_logging(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            print('LOG: Running job "%s"' % func.__name__)
            result = func(*args, **kwargs)
            print('LOG: Job "%s" completed' % func.__name__)
            return result
        return wrapper

    @with_logging
    def job():
        print('Hello, World.')

    schedule.every(3).seconds.do(job)

    while 1:
        schedule.run_pending()
        time.sleep(1)

How to run a job at random intervals?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    def my_job():
        # This job will execute every 5 to 10 seconds.
        print('Foo')

    schedule.every(5).to(10).seconds.do(my_job)

How can I pass arguments to the job function?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``do()`` passes extra arguments to the job function:

.. code-block:: python

    def greet(name):
        print('Hello', name)

    schedule.every(2).seconds.do(greet, name='Alice')
    schedule.every(4).seconds.do(greet, name='Bob')

How can I make sure long-running jobs are always executed on time?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Schedule does not account for the time it takes the job function to execute. To guarantee a stable execution schedule you need to move long-running jobs off the main-thread (where the scheduler runs). See "How to execute jobs in parallel?" in the FAQ for a sample implementation. 

