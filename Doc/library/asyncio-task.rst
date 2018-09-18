.. currentmodule:: asyncio


====================
Coroutines and Tasks
====================

This section outlines high-level asyncio APIs to work with coroutines
and Tasks.

.. contents::
   :depth: 1
   :local:


.. _coroutine:

Coroutines
==========

Coroutines declared with async/await syntax is the preferred way of
writing asyncio applications.  For example, the following snippet
of code prints "hello", waits 1 second, and then prints "world"::

    >>> import asyncio

    >>> async def main():
    ...     print('hello')
    ...     await asyncio.sleep(1)
    ...     print('world')

    >>> asyncio.run(main())
    hello
    world

Note that simply calling a coroutine will not schedule it to
be executed::

    >>> main()
    <coroutine object main at 0x1053bb7c8>

To actually run a coroutine asyncio provides three main mechanisms:

* The :func:`asyncio.run` function to run the top-level
  entry point "main()" function (see the above example.)

* Awaiting on a coroutine.  The following snippet of code will
  print "hello" after waiting for 1 second, and then print "world"
  after waiting for *another* 2 seconds::

      import asyncio
      import time

      async def say_after(delay, what):
          await asyncio.sleep(delay)
          print(what)

      async def main():
          print('started at', time.strftime('%X'))

          await say_after(1, 'hello')
          await say_after(2, 'world')

          print('finished at', time.strftime('%X'))

      asyncio.run(main())

  Expected output::

      started at 17:13:52
      hello
      world
      finished at 17:13:55

* The :func:`asyncio.create_task` function to run coroutines
  concurrently as asyncio :class:`Tasks <Task>`.

  Let's modify the above example and run two "set_after" coroutines
  *concurrently*::

      async def main():
          task1 = asyncio.create_task(
              say_after(1, 'hello'))

          task2 = asyncio.create_task(
              say_after(2, 'world'))

          print('started at', time.strftime('%X'))

          # Wait until both tasks are completed (should take
          # around 2 seconds.)
          await task1
          await task2

          print('finished at', time.strftime('%X'))

  Note that expected output now shows that the snippet runs
  1 second faster than before::

      started at 17:14:32
      hello
      world
      finished at 17:14:34

Note that in this documentation the term "coroutine" can be used for
two closely related concepts:

* a *coroutine function*: an :keyword:`async def` function;

* a *coroutine object*: object returned by calling a
  *coroutine function*.


Running an asyncio Program
==========================

.. function:: run(coro, \*, debug=False)

    This function runs the passed coroutine, taking care of
    managing the asyncio event loop and finalizing asynchronous
    generators.

    This function cannot be called when another asyncio event loop is
    running in the same thread.

    If *debug* is ``True``, the event loop will be run in debug mode.

    This function always creates a new event loop and closes it at
    the end.  It should be used as a main entry point for asyncio
    programs, and should ideally only be called once.

    .. versionadded:: 3.7
       **Important:** this function has been added to asyncio in
       Python 3.7 on a :term:`provisional basis <provisional api>`.


Creating Tasks
==============

.. function:: create_task(coro)

   Wrap the *coro* :ref:`coroutine <coroutine>` into a task and schedule
   its execution. Return the task object.

   The task is executed in the loop returned by :func:`get_running_loop`,
   :exc:`RuntimeError` is raised if there is no running loop in
   current thread.

   .. versionadded:: 3.7


Sleeping
========

.. coroutinefunction:: sleep(delay, result=None, \*, loop=None)

   Block for *delay* seconds.

   If *result* is provided, it is returned to the caller
   when the coroutine completes.

   .. _asyncio_example_sleep:

   Example of coroutine displaying the current date every second
   for 5 seconds::

    import asyncio
    import datetime

    async def display_date():
        loop = asyncio.get_running_loop()
        end_time = loop.time() + 5.0
        while True:
            print(datetime.datetime.now())
            if (loop.time() + 1.0) >= end_time:
                break
            await asyncio.sleep(1)

    asyncio.run(display_date())


Running Tasks Concurrently
==========================

.. function:: gather(\*fs, loop=None, return_exceptions=False)

   Return a Future aggregating results from the given coroutine objects,
   Tasks, or Futures.

   If all Tasks/Futures are completed successfully, the result is an
   aggregate list of returned values.  The result values are in the
   order of the original *fs* sequence.

   All coroutines in the *fs* list are automatically
   scheduled as :class:`Tasks <Task>`.

   If *return_exceptions* is ``True``, exceptions in the Tasks/Futures
   are treated the same as successful results, and gathered in the
   result list.  Otherwise, the first raised exception is immediately
   propagated to the returned Future.

   If the outer Future is *cancelled*, all submitted Tasks/Futures
   (that have not completed yet) are also *cancelled*.

   If any child is *cancelled*, it is treated as if it raised
   :exc:`CancelledError` -- the outer Future is **not** cancelled in
   this case.  This is to prevent the cancellation of one submitted
   Task/Future to cause other Tasks/Futures to be cancelled.

   All futures must share the same event loop.

   .. versionchanged:: 3.7
      If the *gather* itself is cancelled, the cancellation is
      propagated regardless of *return_exceptions*.

   .. _asyncio_example_gather:

   Example::

      import asyncio

      async def factorial(name, number):
          f = 1
          for i in range(2, number + 1):
              print(f"Task {name}: Compute factorial({i})...")
              await asyncio.sleep(1)
              f *= i
          print(f"Task {name}: factorial({number}) = {f}")

      async def main():
          await asyncio.gather(
              factorial("A", 2),
              factorial("B", 3),
              factorial("C", 4),
          ))

      asyncio.run(main())

      # Expected output:
      #
      #     Task A: Compute factorial(2)...
      #     Task B: Compute factorial(2)...
      #     Task C: Compute factorial(2)...
      #     Task A: factorial(2) = 2
      #     Task B: Compute factorial(3)...
      #     Task C: Compute factorial(3)...
      #     Task B: factorial(3) = 6
      #     Task C: Compute factorial(4)...
      #     Task C: factorial(4) = 24


Shielding Tasks From Cancellation
=================================

.. coroutinefunction:: shield(fut, \*, loop=None)

   Wait for a Future/Task while protecting it from being cancelled.

   *fut* can be a coroutine, a Task, or a Future-like object.  If
   *fut* is a coroutine it is automatically scheduled as a
   :class:`Task`.

   The statement::

       res = await shield(something())

   is equivalent to::

       res = await something()

   *except* that if the coroutine containing it is cancelled, the
   Task running in ``something()`` is not cancelled.  From the point
   of view of ``something()``, the cancellation did not happen.
   Although its caller is still cancelled, so the "await" expression
   still raises a :exc:`CancelledError`.

   If ``something()`` is cancelled by other means (i.e. from within
   itself) that would also cancel ``shield()``.

   If it is desired to completely ignore cancellation (not recommended)
   the ``shield()`` function should be combined with a try/except
   clause, as follows::

       try:
           res = await shield(something())
       except CancelledError:
           res = None


Timeouts
========

.. coroutinefunction:: wait_for(fut, timeout, \*, loop=None)

   Wait for a coroutine, Task, or Future to complete with timeout.

   *fut* can be a coroutine, a Task, or a Future-like object.  If
   *fut* is a coroutine it is automatically scheduled as a
   :class:`Task`.

   *timeout* can either be ``None`` or a float or int number of seconds
   to wait for.  If *timeout* is ``None``, block until the future
   completes.

   If a timeout occurs, it cancels the task and raises
   :exc:`asyncio.TimeoutError`.

   To avoid the task cancellation, wrap it in :func:`shield`.

   The function will wait until the future is actually cancelled,
   so the total wait time may exceed the *timeout*.

   If the wait is cancelled, the future *fut* is also cancelled.

   .. _asyncio_example_waitfor:

   Example::

       async def eternity():
           # Sleep for one hour
           await asyncio.sleep(3600)
           print('yay!')

       async def main():
           # Wait for at most 1 second
           try:
               await asyncio.wait_for(eternity(), timeout=1.0)
           except asyncio.TimeoutError:
               print('timeout!')

       asyncio.run(main())

       # Expected output:
       #
       #     timeout!

   .. versionchanged:: 3.7
      When *fut* is cancelled due to a timeout, ``wait_for`` waits
      for *fut* to be cancelled.  Previously, it raised
      :exc:`asyncio.TimeoutError` immediately.


Waiting Primitives
==================

.. coroutinefunction:: wait(fs, \*, loop=None, timeout=None,\
                            return_when=ALL_COMPLETED)

   Wait for a set of coroutines, Tasks, or Futures to complete.

   *fs* is a list of coroutines, Futures, and/or Tasks.  Coroutines
   are automatically scheduled as :class:`Tasks <Task>`.

   Returns two sets of Tasks/Futures: ``(done, pending)``.

   *timeout* (a float or int), if specified, can be used to control
   the maximum number of seconds to wait before returning.

   Note that this function does not raise :exc:`asyncio.TimeoutError`.
   Futures or Tasks that aren't done when the timeout occurs are simply
   returned in the second set.

   *return_when* indicates when this function should return.  It must
   be one of the following constants:

   .. tabularcolumns:: |l|L|

   +-----------------------------+----------------------------------------+
   | Constant                    | Description                            |
   +=============================+========================================+
   | :const:`FIRST_COMPLETED`    | The function will return when any      |
   |                             | future finishes or is cancelled.       |
   +-----------------------------+----------------------------------------+
   | :const:`FIRST_EXCEPTION`    | The function will return when any      |
   |                             | future finishes by raising an          |
   |                             | exception.  If no future raises an     |
   |                             | exception then it is equivalent to     |
   |                             | :const:`ALL_COMPLETED`.                |
   +-----------------------------+----------------------------------------+
   | :const:`ALL_COMPLETED`      | The function will return when all      |
   |                             | futures finish or are cancelled.       |
   +-----------------------------+----------------------------------------+

   Unlike :func:`~asyncio.wait_for`, ``wait()`` does not cancel the
   futures when a timeout occurs.

   Usage::

        done, pending = await asyncio.wait(fs)


.. function:: as_completed(fs, \*, loop=None, timeout=None)

   Return an iterator of awaitables which return
   :class:`Future` instances.

   Raises :exc:`asyncio.TimeoutError` if the timeout occurs before
   all Futures are done.

   Example::

       for f in as_completed(fs):
           result = await f
           # ...


Scheduling From Other Threads
=============================

.. function:: run_coroutine_threadsafe(coro, loop)

   Submit a coroutine to the given event loop.  Thread-safe.

   Return a :class:`concurrent.futures.Future` to access the result.

   This function is meant to be called from a different OS thread
   than the one where the event loop is running.  Example::

     # Create a coroutine
     coro = asyncio.sleep(1, result=3)

     # Submit the coroutine to a given loop
     future = asyncio.run_coroutine_threadsafe(coro, loop)

     # Wait for the result with an optional timeout argument
     assert future.result(timeout) == 3

   If an exception is raised in the coroutine, the returned Future
   will be notified.  It can also be used to cancel the task in
   the event loop::

     try:
         result = future.result(timeout)
     except asyncio.TimeoutError:
         print('The coroutine took too long, cancelling the task...')
         future.cancel()
     except Exception as exc:
         print('The coroutine raised an exception: {!r}'.format(exc))
     else:
         print('The coroutine returned: {!r}'.format(result))

   See the :ref:`concurrency and multithreading <asyncio-multithreading>`
   section of the documentation.

   Unlike other asyncio functions this functions requires the *loop*
   argument to be passed explicitly.

   .. versionadded:: 3.5.1


Introspection
=============


.. function:: current_task(loop=None)

   Return the currently running :class:`Task` instance, or ``None`` if
   no task is running.

   If *loop* is ``None`` :func:`get_running_loop` is used to get
   the current loop.

   .. versionadded:: 3.7


.. function:: all_tasks(loop=None)

   Return a set of not yet finished :class:`Task` objects run by
   the loop.

   If *loop* is ``None``, :func:`get_running_loop` is used for getting
   current loop.

   .. versionadded:: 3.7


Task Object
===========

.. class:: Task(coro, \*, loop=None)

   A :class:`Future`-like object that wraps a Python
   :ref:`coroutine <coroutine>`.  Not thread-safe.

   Tasks are used to run coroutines in event loops.
   If a coroutine awaits on a Future, the Task suspends
   the execution of the coroutine and waits for the completion
   of the Future.  When the Future is *done*, the execution of
   the wrapped coroutine resumes.

   Event loops use cooperative scheduling: an event loop runs
   one Task at a time.  While a Task awaits for the completion of a
   Future, the event loop runs other Tasks, callbacks, or performs
   IO operations.

   Use the high-level :func:`asyncio.create_task` function to create
   Tasks, or the low-level :meth:`loop.create_task` or
   :func:`ensure_future` functions.  Manual instantiation of Tasks
   is discouraged.

   To cancel a running Task use the :meth:`cancel` method.  Calling it
   will cause the Task to throw a :exc:`CancelledError` exception into
   the wrapped coroutine.  If a coroutine is awaiting on a Future
   object during cancellation, the Future object will be cancelled.

   :meth:`cancelled` can be used to check if the Task was cancelled.
   The method returns ``True`` if the wrapped coroutine did not
   suppress the :exc:`CancelledError` exception and was actually
   cancelled.

   :class:`asyncio.Task` inherits from :class:`Future` all of its
   APIs except :meth:`Future.set_result` and
   :meth:`Future.set_exception`.

   Tasks support the :mod:`contextvars` module.  When a Task
   is created it copies the current context and later runs its
   coroutine in the copied context.

   .. versionchanged:: 3.7
      Added support for the :mod:`contextvars` module.

   .. method:: cancel()

      Request the Task to be cancelled.

      This arranges for a :exc:`CancelledError` exception to be thrown
      into the wrapped coroutine on the next cycle of the event loop.

      The coroutine then has a chance to clean up or even deny the
      request by suppressing the exception with a :keyword:`try` ...
      ... ``except CancelledError`` ... :keyword:`finally` block.
      Therefore, unlike :meth:`Future.cancel`, :meth:`Task.cancel` does
      not guarantee that the Task will be cancelled, although
      suppressing cancellation completely is not common and is actively
      discouraged.

      .. _asyncio_example_task_cancel:

      The following example illustrates how coroutines can intercept
      the cancellation request::

          async def cancel_me():
              print('cancel_me(): before sleep')

              try:
                  # Wait for 1 hour
                  await asyncio.sleep(3600)
              except asyncio.CancelledError:
                  print('cancel_me(): cancel sleep')
                  raise
              finally:
                  print('cancel_me(): after sleep')

          async def main():
              # Create a "cancel_me" Task
              task = asyncio.create_task(cancel_me())

              # Wait for 1 second
              await asyncio.sleep(1)

              task.cancel()
              try:
                  await task
              except asyncio.CancelledError:
                  print("main(): cancel_me is cancelled now")

          asyncio.run(main())

          # Expected output:
          #
          #     cancel_me(): before sleep
          #     cancel_me(): cancel sleep
          #     cancel_me(): after sleep
          #     main(): cancel_me is cancelled now

   .. method:: cancelled()

      Return ``True`` if the Task is *cancelled*.

      The Task is *cancelled* when the cancellation was requested with
      :meth:`cancel` and the wrapped coroutine propagated the
      :exc:`CancelledError` exception thrown into it.

   .. method:: done()

      Return ``True`` if the Task is *done*.

      A Task is *done* when the wrapped coroutine either returned
      a value, raised an exception, or the Task was cancelled.

   .. method:: get_stack(\*, limit=None)

      Return the list of stack frames for this Task.

      If the wrapped coroutine is not done, this returns the stack
      where it is suspended.  If the coroutine has completed
      successfully or was cancelled, this returns an empty list.
      If the coroutine was terminated by an exception, this returns
      the list of traceback frames.

      The frames are always ordered from oldest to newest.

      Only one stack frame is returned for a suspended coroutine.

      The optional *limit* argument sets the maximum number of frames
      to return; by default all available frames are returned.
      The ordering of the returned list differs depending on whether
      a stack or a traceback is returned: the newest frames of a
      stack are returned, but the oldest frames of a traceback are
      returned.  (This matches the behavior of the traceback module.)

   .. method:: print_stack(\*, limit=None, file=None)

      Print the stack or traceback for this Task.

      This produces output similar to that of the traceback module
      for the frames retrieved by :meth:`get_stack`.

      The *limit* argument is passed to :meth:`get_stack` directly.

      The *file* argument is an I/O stream to which the output
      is written; by default output is written to :data:`sys.stderr`.

   .. classmethod:: all_tasks(loop=None)

      Return a set of all tasks for an event loop.

      By default all tasks for the current event loop are returned.
      If *loop* is ``None``, the :func:`get_event_loop` function
      is used to get the current loop.

      This method is **deprecated** and will be removed in
      Python 3.9.  Use the :func:`all_tasks` function instead.

   .. classmethod:: current_task(loop=None)

      Return the currently running task or ``None``.

      If *loop* is ``None``, the :func:`get_event_loop` function
      is used to get the current loop.

      This method is **deprecated** and will be removed in
      Python 3.9.  Use the :func:`current_task` function instead.


.. _asyncio_generator_based_coro:

Generator-based Coroutines
==========================

.. note::

   Support for generator-based coroutines is **deprecated** and
   is scheduled for removal in Python 4.0.

Generator-based coroutines predate async/await syntax.  They are
Python generators that use ``yield from`` expressions to await
on Futures and other coroutines.

Generator-based coroutines should be decorated with
:func:`@asyncio.coroutine <asyncio.coroutine>`, although this is not
enforced.


.. decorator:: coroutine

    Decorator to mark generator-based coroutines.

    This decorator enables legacy generator-based coroutines to be
    compatible with async/await code::

        @asyncio.coroutine
        def old_style_coroutine():
            yield from asyncio.sleep(1)

        async def main():
            await old_style_coroutine()

    This decorator is **deprecated** and is scheduled for removal in
    Python 4.0.

    This decorator should not be used for :keyword:`async def`
    coroutines.

.. function:: iscoroutine(obj)

   Return ``True`` if *obj* is a :ref:`coroutine object <coroutine>`.

   This method is different from :func:`inspect.iscoroutine` because
   it returns ``True`` for generator-based coroutines decorated with
   :func:`@coroutine <coroutine>`.

.. function:: iscoroutinefunction(func)

   Return ``True`` if *func* is a :ref:`coroutine function
   <coroutine>`.

   This method is different from :func:`inspect.iscoroutinefunction`
   because it returns ``True`` for generator-based coroutine functions
   decorated with :func:`@coroutine <coroutine>`.
