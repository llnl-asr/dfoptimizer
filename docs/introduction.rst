Introduction
============

DFOptimizer turns DFDiagnoser's **diagnosis findings** into **ActionPlans** —
concrete knob changes (e.g. *raise dataloader prefetch from 2 to 4*) — and,
online, feeds them back to the running application to **re-tune it in
flight**. It runs the same way **offline** (replay saved findings to inspect
the decisions) and **online** (consume a live stream over Mofka or ZMQ and
actuate).

How it works
------------

A finding from DFDiagnoser says *what's wrong, how persistent, and what class
of fix* (``fetch_pressure``, ``persistent_pressure``,
``opportunity_tags=[dataloader_prefetch, reader_parallelism]``, ...).
DFOptimizer's **planner** maps those ``opportunity_tags`` to registered
**knobs** via each knob's ``responds_to`` rule and emits an ``ActionPlan``,
gated so it acts responsibly:

- **persistence / trend / severity** — only act on a bottleneck that's real
  and sticky
- **cooldown + pending-plan** — one change per knob at a time; don't thrash
- **apply_when** — apply at a safe boundary (e.g. ``window_boundary``)

Knobs come from the application: it registers them (id, range,
``responds_to``, ``target_function``) through ``OptimizerContext``, which
also receives the plans and applies them at the next boundary — closing the
diagnose → tune → re-measure loop. For local/offline use, the bundled DLIO
knobs can be bootstrapped with ``DFOPTIMIZER_BOOTSTRAP_DLIO=1``.

Features
--------

- **Dynamic knob registry** — apps declare knobs with the ``@tunable``
  decorator and ``knob(...)`` helper; no hardcoded per-app rules.
- **Gated 4-stage decision pipeline** — severity, persistence, cooldown, and
  effectiveness gates prevent thrashing.
- **Pluggable transports** — offline file replay, ZMQ, or Mofka streaming.
- **Structured logging** — every plan carries a human-readable rationale.

Installation
------------

.. code-block:: bash

   pip install dfoptimizer                 # core (offline replay)
   pip install "dfoptimizer[streaming]"    # + online transports (pyzmq / mofka)

From source: ``uv sync && uv pip install -e .``

Quick start
-----------

``main.py`` selects a transport with ``--transport {mofka,zmq,file}``.

Offline — replay saved findings into plans (a ``.jsonl`` of DFDiagnoser
finding wire-dicts, or a JSON array):

.. code-block:: bash

   DFOPTIMIZER_BOOTSTRAP_DLIO=1 python main.py --transport file \
       --findings-file findings.jsonl

Online — live actuation over ZMQ (consume findings DFDiagnoser streams and
publish the plans onward; the app's ``OptimizerContext`` applies them):

.. code-block:: bash

   python main.py --transport zmq \
       --address "tcp://127.0.0.1:5557" \
       --plans-address "tcp://*:5558"

Online — Mofka (full cluster loop: consume ``diagnosis_findings``, publish
``optimizer_plans``, learn knobs from the app's registry):

.. code-block:: bash

   python main.py --group-file "$MOFKA_GROUP_FILE"
