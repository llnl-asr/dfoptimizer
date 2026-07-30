API Reference
=============

Application-facing API (``dfoptimizer``)
----------------------------------------

The top-level package re-exports the runtime primitives an instrumented
application needs: ``optimizer_context``, ``tunable``, and ``knob``.

``@tunable`` (``dfoptimizer.runtime.tunable``)
   Decorator that marks a function as tunable by DFOptimizer::

      @tunable(knobs={
          "prefetch_size": knob(default=2, range=(1, 16), type=int,
                                responds_to={
                                    "dataloader_prefetch": {"direction": "increase", "step": 2},
                                }),
      })
      def make_dataloader(prefetch_size, **kwargs):
          ...

   When the decorated function is called, pending optimization actions are
   applied by overriding matching keyword arguments, and the knob
   definitions are published to the optimizer service.

``knob(default, type=int, range=None, values=None, scope="job", responds_to=None)``
   (``dfoptimizer.runtime.knob``) Helper to declare a knob inside
   ``@tunable``. ``responds_to`` maps opportunity tags to adjustment specs;
   unspecified fields get sensible defaults from ``KnobResponse``.

``OptimizerContext`` (``dfoptimizer.runtime.context``)
   Bridge between the optimizer service and the instrumented app. Spawns a
   background thread that subscribes to incoming ``ActionPlan`` messages,
   queues them per target function, and publishes the knob registry on
   first use of a ``@tunable`` function.

Core types (``dfoptimizer.types``)
----------------------------------

``KnobDef``
   Definition of a tunable knob: ``id``, ``default``, ``type``, ``range``,
   ``values``, ``scope``, and a ``responds_to`` mapping of opportunity tags
   to ``KnobResponse``.

``KnobResponse``
   How a knob responds to a specific opportunity tag: ``direction``
   (``increase`` / ``decrease`` / ``set``), ``step``, ``step_mode``,
   ``set_to``, ``min_severity``, ``min_persistence``, ``cooldown_windows``,
   ``apply_when``, and ``effectiveness_threshold``.

``ActionPlan`` / ``ActionAck`` / ``DiagnosisFindingMsg``
   Wire types exchanged between DFDiagnoser, the optimizer service, and the
   application context.

Planner (``dfoptimizer.planner``)
---------------------------------

``Planner`` (``dfoptimizer.planner.planner``)
   Decision engine: consumes ``DiagnosisFindingMsg``, produces
   ``ActionPlan``. Dynamically populated from knob registrations; runs a
   4-gate decision pipeline (severity, persistence, cooldown,
   effectiveness) per finding.

``CooldownTracker`` (``dfoptimizer.planner.cooldown``)
   Tracks per-knob cooldown windows and pending plans so only one change
   per knob is in flight at a time.

Streaming transports (``dfoptimizer.streaming``)
------------------------------------------------

``dfoptimizer.streaming.zmq_io``
   ZMQ-based findings consumer / plans publisher (requires the
   ``streaming`` extra, i.e. ``pyzmq``).

``dfoptimizer.streaming.mofka_io``
   Mofka-based transport for the full cluster loop (``diagnosis_findings``
   in, ``optimizer_plans`` out).

Service entry point
-------------------

``dfoptimizer.optimizer`` and the repository's ``main.py`` wire a transport
(``--transport {mofka,zmq,file}``) to the planner. Set
``DFOPTIMIZER_BOOTSTRAP_DLIO=1`` to bootstrap the bundled DLIO knobs for
local/offline use.
