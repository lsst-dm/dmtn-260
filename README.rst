.. image:: https://img.shields.io/badge/dmtn--260-lsst.io-brightgreen.svg
   :target: https://dmtn-260.lsst.io/
.. image:: https://github.com/lsst-dm/dmtn-260/workflows/CI/badge.svg
   :target: https://github.com/lsst-dm/dmtn-260/actions/

######################################################
Failure Modes and Error Handling for Prompt Processing
######################################################

DMTN-260
========

The Prompt Processing system will be responsible for processing roughly a thousand visits per night, and distributing the results in near real time, for at least ten years of Rubin Observatory operations. As such, it must be highly robust to algorithmic, network, and infrastructure failures, ranging from momentary glitches to extended downtimes. `DMTN-219 <https://dmtn-219.lsst.io/>`_ introduced the initial design for the Prompt Processing framework; this document expands on the design to address expected failure modes and recovery strategies for each.

**Links:**

- Publication URL: https://dmtn-260.lsst.io/
- Alternative editions: https://dmtn-260.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-260
- Build system: https://github.com/lsst-dm/dmtn-260/actions/

Build this technical note
=========================

You can clone this repository and build the technote locally if your system has Python 3.12 or later:

.. code-block:: bash

   git clone https://github.com/lsst-dm/dmtn-260
   cd dmtn-260
   make init
   make html

Repeat the ``make html`` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run ``make clean``.

The built technote is located at ``_build/html/index.html``.

Publishing changes to the web
=============================

This technote is published to https://dmtn-260.lsst.io/ whenever you push changes to the ``main`` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-260.lsst.io/v.

Editing this technical note
===========================

The main content of this technote is in ``index.rst`` (a reStructuredText file).
Metadata and configuration is in the ``technote.toml`` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
