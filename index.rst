:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml

.. TODO: Delete the note below before merging new content to the main branch.

.. note::

   **This technote is a work-in-progress.**

Abstract
========

The Prompt Processing system will be responsible for processing roughly a thousand visits per night, and distributing the results in near real time, for at least ten years of Rubin Observatory operations.
As such, it must be highly robust to algorithmic, network, and infrastructure failures, ranging from momentary glitches to extended downtimes.
`DMTN-219`_ introduced the initial design for the Prompt Processing framework; this document expands on the design to address expected failure modes and recovery strategies for each.

.. _DMTN-219: https://dmtn-219.lsst.io/

Add content here
================

Add content here.
See the `reStructuredText Style Guide <https://developer.lsst.io/restructuredtext/style.html>`__ to learn how to create sections, links, images, tables, equations, and more.

.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. .. rubric:: References
.. 
.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
