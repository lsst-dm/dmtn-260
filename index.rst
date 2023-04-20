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

.. _retries:

Retrying Processing
===================

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
