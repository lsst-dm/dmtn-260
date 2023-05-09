:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml

.. TODO: Delete the note below before merging new content to the main branch.

.. note::

   **This technote is a work-in-progress.**

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
The alert stream is a close second, since it is the primary user interface for transient follow-up, and while erroneous alerts can be retracted, they may trigger other actions in the meantime.
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

- Any failure before the end of APDB writing (which is atomic as of October 2023) has no consequences beyond that worker's local repository.
  The repository may have ingested raws, or partial pipeline outputs.
- A failure after APDB writing but before the end of ``DiaPipelineTask``, which contains it, is recoverable only if the task can account for the fact that a previous session already wrote to the APDB.
  As discussed in :ref:`association`, it is not possible to make APDB writes idempotent.
- A failure after APDB writing but before or during alert distribution cannot be aborted without leaving the APDB inconsistent with the alert stream.
  However, retrying is possible, and easy if duplicate alerts are acceptable (see :ref:`general-priorities`).
- A failure after alert distribution but before or during central repo sync may leave the latter missing PVIs and difference images, which are supposed to be exported to the Rubin Science Platform.
- A failure after central repo sync may interfere with internal metrics and monitoring, but has no consequences for science users.

In general, the Rubin Observatory project has followed a policy of processing data at most once, not at least once.
We propose to do the same thing in Prompt Processing, by attempting to retry processing only in the case of specific failure modes that we know to be recoverable.
As a first approximation, this means that retries are allowed before writing to the APDB, but not afterward.
Any failures that are not retried automatically can still be handled in next-day processing.


.. _association:

APDB and Source Association
===========================

.. _consistency:

Inconsistent Output
===================

.. _bad-data:

Corrupted Pipeline Outputs
==========================

.. _timeout:

Pipeline Timeouts
=================

.. _major-downtime:

System Downtime
===============

.. _summary:

Summary
=======

.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. .. rubric:: References
.. 
.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
