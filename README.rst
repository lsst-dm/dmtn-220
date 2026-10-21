.. image:: https://img.shields.io/badge/dmtn--220-lsst.io-brightgreen.svg
   :target: https://dmtn-220.lsst.io/
.. image:: https://github.com/lsst-dm/dmtn-220/workflows/CI/badge.svg
   :target: https://github.com/lsst-dm/dmtn-220/actions/

#########################################################
Middleware Support for Campaign Definition and Management
#########################################################

DMTN-220
========

The description of the Campaign Management system in the first draft of DMTN-181 :cite:`DMTN-181` implies a few butler/pipeline middleware feature requests, and the processes and tooling for defining campaigns are expected to need new middleware support as well. This technote will attempt to aggregate those feature requests, while outlining a few possible approaches for how campaign definition and management will interact with the middleware, based on the absence or presence of potential new middleware functionality.

**Links:**

- Publication URL: https://dmtn-220.lsst.io/
- Alternative editions: https://dmtn-220.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-220
- Build system: https://github.com/lsst-dm/dmtn-220/actions/

Build this technical note
=========================

You can clone this repository and build the technote locally if your system has Python 3.12 or later:

.. code-block:: bash

   git clone https://github.com/lsst-dm/dmtn-220
   cd dmtn-220
   make init
   make html

Repeat the ``make html`` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run ``make clean``.

The built technote is located at ``_build/html/index.html``.

Publishing changes to the web
=============================

This technote is published to https://dmtn-220.lsst.io/ whenever you push changes to the ``main`` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-220.lsst.io/v.

Editing this technical note
===========================

The main content of this technote is in ``index.rst`` (a reStructuredText file).
Metadata and configuration is in the ``technote.toml`` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
