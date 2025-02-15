Fault Tolerance
===============

This document describes how Ray handles machine and process failures.

Tasks
-----

When a worker is executing a task, if the worker dies unexpectedly, either
because the process crashed or because the machine failed, Ray will rerun
the task until either the task succeeds or the maximum number of retries is
exceeded. The default number of retries is 3 and can be overridden by
specifying ``max_retries`` in the ``@ray.remote`` decorator. Specifying -1
allows infinite retries, and 0 disables retries.

You can experiment with this behavior by running the following code.

.. code-block:: python

    import numpy as np
    import os
    import ray
    import time

    ray.init(ignore_reinit_error=True)

    @ray.remote(max_retries=1)
    def potentially_fail(failure_probability):
        time.sleep(0.2)
        if np.random.random() < failure_probability:
            os._exit(0)
        return 0

    for _ in range(3):
        try:
            # If this task crashes, Ray will retry it up to one additional
            # time. If either of the attempts succeeds, the call to ray.get
            # below will return normally. Otherwise, it will raise an
            # exception.
            ray.get(potentially_fail.remote(0.5))
            print('SUCCESS')
        except ray.exceptions.WorkerCrashedError:
            print('FAILURE')

.. _object-reconstruction:

Ray also implements *lineage reconstruction* to recover task outputs that are
lost from the distributed object store. This can occur during node failures.
Ray will first automatically attempt to recover the value by looking for copies
of the same object on other nodes. If none are found, then Ray will
automatically recover the value by re-executing the task that created the
value. Arguments to the task are recursively reconstructed with the same
method.

Note that lineage reconstruction can cause higher than usual driver memory
usage because the driver keeps the descriptions of any tasks that may be
re-executed in case of a failure. To limit the amount of memory used by
lineage, set the environment variable ``RAY_max_lineage_bytes`` (default 1GB)
to evict lineage if the threshold is exceeded.

To disable this behavior, set the environment variable
``RAY_lineage_pinning_enabled=0`` during ``ray start`` or ``ray.init``.  With
this setting, if there are no copies of an object left, an ``ObjectLostError``
will be raised.

.. _actor-fault-tolerance:

Actors
------

Ray will automatically restart actors that crash unexpectedly.
This behavior is controlled using ``max_restarts``,
which sets the maximum number of times that an actor will be restarted.
If 0, the actor won't be restarted. If -1, it will be restarted infinitely.
The default value of ``max_restarts`` is 0.
When an actor is restarted, its state will be recreated by rerunning its
constructor.
After the specified number of restarts, subsequent actor methods will
raise a ``RayActorError``.
You can experiment with this behavior by running the following code.

.. code-block:: python

    import os
    import ray
    import time

    ray.init(ignore_reinit_error=True)

    @ray.remote(max_restarts=5)
    class Actor:
        def __init__(self):
            self.counter = 0

        def increment_and_possibly_fail(self):
            self.counter += 1
            time.sleep(0.2)
            if self.counter == 10:
                os._exit(0)
            return self.counter

    actor = Actor.remote()

    # The actor will be restarted up to 5 times. After that, methods will
    # always raise a `RayActorError` exception. The actor is restarted by
    # rerunning its constructor. Methods that were sent or executing when the
    # actor died will also raise a `RayActorError` exception.
    for _ in range(100):
        try:
            counter = ray.get(actor.increment_and_possibly_fail.remote())
            print(counter)
        except ray.exceptions.RayActorError:
            print('FAILURE')

By default, actor tasks execute with at-most-once semantics
(``max_task_retries=0`` in the ``@ray.remote`` decorator). This means that if an
actor task is submitted to an actor that is unreachable, Ray will report the
error with ``RayActorError``, a Python-level exception that is thrown when
``ray.get`` is called on the future returned by the task. Note that this
exception may be thrown even though the task did indeed execute successfully.
For example, this can happen if the actor dies immediately after executing the
task.

Ray also offers at-least-once execution semantics for actor tasks
(``max_task_retries=-1`` or ``max_task_retries > 0``). This means that if an
actor task is submitted to an actor that is unreachable, the system will
automatically retry the task until it receives a reply from the actor. With
this option, the system will only throw a ``RayActorError`` to the application
if one of the following occurs: (1) the actor’s ``max_restarts`` limit has been
exceeded and the actor cannot be restarted anymore, or (2) the
``max_task_retries`` limit has been exceeded for this particular task. The
limit can be set to infinity with ``max_task_retries = -1``.

You can experiment with this behavior by running the following code.

.. code-block:: python

    import os
    import ray

    ray.init(ignore_reinit_error=True)

    @ray.remote(max_restarts=5, max_task_retries=-1)
    class Actor:
        def __init__(self):
            self.counter = 0

        def increment_and_possibly_fail(self):
            # Exit after every 10 tasks.
            if self.counter == 10:
                os._exit(0)
            self.counter += 1
            return self.counter

    actor = Actor.remote()

    # The actor will be reconstructed up to 5 times. The actor is
    # reconstructed by rerunning its constructor. Methods that were
    # executing when the actor died will be retried and will not
    # raise a `RayActorError`. Retried methods may execute twice, once
    # on the failed actor and a second time on the restarted actor.
    for _ in range(50):
        counter = ray.get(actor.increment_and_possibly_fail.remote())
        print(counter)  # Prints the sequence 1-10 5 times.

    # After the actor has been restarted 5 times, all subsequent methods will
    # raise a `RayActorError`.
    for _ in range(10):
        try:
            counter = ray.get(actor.increment_and_possibly_fail.remote())
            print(counter)  # Unreachable.
        except ray.exceptions.RayActorError:
            print('FAILURE')  # Prints 10 times.

For at-least-once actors, the system will still guarantee execution ordering
according to the initial submission order. For example, any tasks submitted
after a failed actor task will not execute on the actor until the failed actor
task has been successfully retried. The system will not attempt to re-execute
any tasks that executed successfully before the failure (unless :ref:`object reconstruction <object-reconstruction>` is enabled).

At-least-once execution is best suited for read-only actors or actors with
ephemeral state that does not need to be rebuilt after a failure. For actors
that have critical state, it is best to take periodic checkpoints and either
manually restart the actor or automatically restart the actor with at-most-once
semantics. If the actor’s exact state at the time of failure is needed, the
application is responsible for resubmitting all tasks since the last
checkpoint.

.. note::
    For :ref:`async or threaded actors <async-actors>`, the tasks might
    be executed out of order. Upon actor restart, the system will only retry
    *incomplete* tasks. Previously completed tasks will not be
    re-executed.
