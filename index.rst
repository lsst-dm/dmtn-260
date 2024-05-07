:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml


.. _abstract:

Abstract
========

The Prompt Processing system will be responsible for processing roughly a thousand visits per night, and distributing the results in near real time, for at least ten years of Rubin Observatory operations.
As such, it must be highly robust to algorithmic, network, and infrastructure failures, ranging from momentary glitches to extended downtimes.
`DMTN-219`_ introduced the initial design for the Prompt Processing framework; this document expands on the design to address expected failure modes and recovery strategies for each.

.. _DMTN-219: https://dmtn-219.lsst.io/

.. _pp-overview:

Prompt Processing System Overview
=================================

The Prompt Processing system will receive images from LATISS, LSSTComCam, and LSSTCam, and run one of several pipelines on them, depending on the purpose of the observations.
In the baseline case (running the full Alert Production pipeline on images for the Legacy Survey of Space and Time), the system will create multiple outputs: the transient alert stream, the alert production database (APDB), datasets in a central repository (including the prompt data products described in the `DPDD`_), and Sasquatch metrics for pipeline analytics and debugging.

.. _DPDD: https://lse-163.lsst.io/

As of October 2023, the Prompt Processing system starts processing when it receives a ``next_visit`` event from the summit before observations are taken.
The event is received by a fan-out service, which forwards a derived event for each detector to a pool of workers.
Each fanned-out event represents the exposure(s) for a given visit with a given detector.
Because the workers handle their assigned exposures independently and in parallel, this framework is efficient and flexible with respect to unexpected computational demands.
However, it's impossible to make assumptions about the order of events between workers, and any given worker may be created or destroyed at any time.

Each of the several hundred workers maintains an internal ("local") Butler repository for its processing; this keeps the central repository from becoming a performance bottleneck for pipeline execution.
On receiving its visit-detector assignment, each worker downloads any calibrations, templates, etc. it will need but does not yet have from the central repo.
The worker then waits for the raw exposures to arrive at USDF (or confirms they already arrived) and downloads and ingests them.
When all exposures have arrived, the worker runs the pipeline, updating the APDB and sending alerts as appropriate; if not all exposures arrive before a timeout, it attempts to process what it has.
Once pipeline processing is complete, the worker uploads its output datasets to the central repository.
The worker will likely also compute metrics at this stage, but the details (in particular, whether they are computed before or after the central repository sync) have not yet been decided.

For performance reasons, each worker places pipeline outputs from all visits into a small number of Butler runs, which are persisted when each worker uploads to the central repository.
The run name is a deterministic function of the Science Pipelines configuration, the instrument and choice of pipeline, and the processing date.
The use of a separate run for each pipeline is required by the Gen 3 provenance system (in particular, the storage of pipeline configuration information and "init-output" datasets), but makes it difficult for the Prompt Processing system to adjust its choice of pipeline in response to missing inputs or to processing failures.

.. _general-priorities:

General Priorities
==================

Of the outputs produced by the Alert Production pipeline, the most important to have correct and uncorrupted is the APDB, which serves as the source of truth both for later processing and for the externally available Prompt Products database.
The alert stream is a close second, since it is the primary user interface for transient follow-up, and while erroneous alerts can be retracted (see :ref:`bad-data`), they may trigger other actions in the meantime.
The central repository and metrics are less important: while the repository is the source of processed visit images (PVIs) and difference images for science users, all other data products are exclusively for internal use.

As far as possible, the Prompt Processing pipeline should be idempotent -- in other words, multiple tries of the same pipeline on the same dataset should yield the same results, no matter what the state of the system.
This is possible when rebroadcasting the alert stream -- in the worst case, duplicate alerts exist but have the same position and time -- and when running the pipeline from ISR through image differencing.
However, for reasons explored in :ref:`association`, idempotence cannot be safely achieved with source association.

.. _retries:

Retrying Processing
===================

The execution framework can automatically recover from failures, in the sense of restoring full processing capacity without compromising the framework's state.
However, recovering the data processing itself is the responsibility of the code running on each worker.

Visit processing can fail at any point, including within and between tasks, and while performing external I/O.
Even within a single worker, the order of events is not fully deterministic (e.g., two snaps could be ingested and processed up to snap combination in parallel, though we have not yet tried to do so).
However, the possible failures can be divided into a handful of cases:

- Any failure before the start of APDB writing (writes are not necessarily atomic) has no consequences beyond that worker's local repository.
  The repository may have ingested raws, or partial pipeline outputs.
- A failure before or during DiaObject writing, but before the start of DiaSource writing, is recoverable if DiaObject entries can be updated without conflict.
  All of a DiaObject's properties, except for its presence, are derived from its associated DiaSources, so no change is permanent until the sources are written.
- A failure after the start of DiaSource writing but before the end of ``DiaPipelineTask`` is recoverable only if the task can account for the fact that a previous session already wrote to the APDB.
  As discussed in :ref:`association`, it is not possible to make APDB writes idempotent.
- A failure after APDB writing but before or during alert distribution cannot be aborted without leaving the APDB inconsistent with the alert stream.
  However, retrying is possible, and easy if duplicate alerts are acceptable (see :ref:`general-priorities`).
- A failure after alert distribution but before or during central repo sync may leave the latter missing PVIs and difference images, which are supposed to be exported to the Rubin Science Platform.
- A failure after central repo sync may interfere with internal metrics and monitoring, but has no consequences for science users.

In general, the Rubin Observatory project has followed a policy of processing data at most once, not at least once.
We propose to do the same thing in Prompt Processing, by attempting to retry processing only in the case of specific failure modes that we know to be recoverable.
As a first approximation, this means that retries are allowed before starting DiaSource writes, but not afterward.
Any failures that are not retried automatically can still be handled in next-day processing.


.. _association:

APDB and Source Association
===========================

The persistent nature of the APDB makes it difficult to retry processing runs that modify it.
One danger is ID collisions, which cannot be entirely prevented simply by choice of the ID generation algorithm.
If DiaSource and DiaObject IDs are deterministic functions of only their visit, then pipeline code might handle retries by testing for these IDs in the APDB, and ignoring or overwriting them.
However, if IDs are not unique (which is hard to verify), treating ID collisions as normal events would lead to silent database corruption.
On the other hand, if IDs are non-deterministic or depend on context (e.g., the set of existing DiaObjects), then retries may create duplicate entries in the APDB.
In either case, the best resolution for any conflict depends on the situation, and therefore requires human judgment.

A more fundamental problem is that the source association algorithm is not time-symmetric.
If there is a DiaObject at a DiaSource's position, the source is merged into the existing object; it not, a new DiaObject is created.
It follows that the final set of DiaObjects depends on the order in which DiaSources are processed.
This characteristic is unlikely to change in the future.

The expectation of idempotence (see :ref:`general-priorities`) amounts to making the association results independent of the processing order.
However, any attempt to achieve this will lead to inconsistencies in the APDB.
For example, daytime corrections could invalidate the creation of a DiaObject, forcing any later DiaSources associated with it to be reassociated.
However, since such a recalculation would change which DiaObjects are available for association, the associations of *other* DiaSources with nearby DiaObjects might no longer satisfy the association algorithm's guarantees, unless all associations are recomputed from scratch.

On the other hand, trying to enforce an effective processing order on the fly also leads to inconsistent output.
For example, preventing a retry or a delayed processing run from using any APDB entries added after the "correct" time can lead to two visits creating DiaObjects at the same position because each is required to ignore the other.
More complex strategies using validity ranges or other tools can avoid such paradoxes, but may lead to more subtle bugs.

The simplest way to keep a consistent association order when recovering from processing failures is to allow all runs to use the state of the APDB at the final processing time.
If we (and science users) think of out-of-order visits as precoveries, then there shouldn't be any confusion over the processing order not being strictly chronological.

.. _consistency:

Inconsistent Output
===================

As noted in :ref:`general-priorities`, the most important pipeline outputs are, in order, the APDB, the alert stream, and the central repository and metrics.
As of October 2023, this is also the order in which the pipeline produces outputs.
If pipeline processing fails in its late stages, these outputs may be inconsistent with each other; for example, the APDB may contain DiaSources for which alerts were never sent.

The alert stream can be easily restored from the APDB, which contains a (possibly incomplete) record of which alerts were sent.
The reverse conversion is unsafe, because injecting associations after other visits have been processed could lead to contradictory source association histories (see :ref:`association`).

The central repository can itself acquire inconsistencies in two ways.
First, we will try to transfer the outputs from failed (i.e., incomplete) pipelines, as these may help in diagnosing the problem.
Such a strategy is safe so long as no code assumes that the existence of one dataset implies the existence of another.
Second, the central repository sync itself may fail, leaving an undefined subset of datasets transferred.
Again, the immediate risk is that something might assume the repository contains a self-consistent set of outputs; in the longer term, the datasets can be regenerated through daytime processing.


.. _bad-data:

Corrupted Pipeline Outputs
==========================

It's possible that some processing errors will allow the pipeline to run to completion while producing large numbers of invalid sources.
Such sources will clutter the alert stream with false positives, and may confuse source association on later visits.

The Prompt Processing framework does not itself have any way to detect nonsense output.
However, the Alert Production team is incorporating "circuit breaker" checks into the pipeline; see `DM-37142`_ and its follow-up issues.
These checks will escalate suspicious outputs into pipeline failures, which can be handled as described above.
As of October 2023, the proposed checks focus on poor-quality raw inputs; there are no checks specifically guarding DiaSource detection or the APDB.

.. _DM-37142: https://jira.lsstcorp.org/browse/DM-37142

If invalid sources are reported through the alert stream, a way to retract alerts will be useful.
Such a design is described in `DMTN-259`_.
It's out of scope for this note, since retraction will require human intervention and cannot be done at processing time.

.. _DMTN-259: https://dmtn-259.lsst.io/

.. _timeout:

Pipeline Timeouts
=================

Another risk is that, under some circumstances, the pipeline code may fail to terminate.
This risk is mitigated by the scalable processing framework; taking a single worker out of action will not interfere with the processing of other images or with overall system performance.

The obvious way to handle stuck pipelines is to impose a timeout on either the job or the worker.
The former carries the risk that the worker will be left in an inconsistent state, corrupting future processing.
The latter adds the overhead of starting a new worker and preparing its local repository, and (depending on the pipeline state and shutdown handling) risks losing all intermediate data.
In either case, the possibility of a timeout needs to be accounted for when solving the problems discussed in :ref:`association` and :ref:`consistency`, since a timeout can interrupt processing at *any* time, including during I/O.

It might be possible to impose a timeout at the task level rather than the worker level, to better distinguish between a slow job and a stuck one.
However, no current framework allows tasks to be timed in real time; for example, timing metrics are only available after a task has completed.

.. _major-downtime:

System Downtime
===============

All of the above assumes that failures are single events -- an exception from a single task, a network glitch, an invalid DiaObject.
However, it's also possible that the system itself will fail for periods much longer than the minute or so it takes to process an image.
For example, the summit link may be cut, or USDF servers may go offline.

Recovering from such outages will be the responsibility of daytime processing rather than the Prompt Processing framework.
This catch-up processing will need to create its own alerts, per `DMTN-248`_, and update the APDB accordingly.
The main concern for Prompt Processing will be ensuring that any ``next_visit`` messages posted during system downtime don't overload it.
Flushing the event queue when the Prompt Processing service starts will prevent this.

.. _DMTN-248: https://dmtn-248.lsst.io/

.. _future:

Future Work
===========

We will continue revising our error-handling strategies as we gain more experience running the pipeline in real time, and as other parts of the system are clarified.

APDB History
------------

In early discussions, we considered adding validity ranges to all APDB tables so that the information could be used to make retroactive corrections to the database.
However, the proposal was dropped on the grounds that validities are intended to mark out-of-date, not erroneous, information.
However, augmenting the APDB schema with an eye to improving the APDB's robustness is still on the table.

Daytime Corrections
-------------------

This note assumes that daytime processing will be able to "catch up" on any visits that failed during prompt processing or that were missed because of system downtime.
It further assumes that the outputs of such processing can be smoothly integrated into the APDB and the central repo.
However, the exact capabilities of daytime processing are still unclear, and in particular interaction with the APDB will need to be handled carefully.

.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. .. rubric:: References
.. 
.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
