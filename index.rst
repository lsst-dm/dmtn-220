:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the main branch.

.. note::

   **This technote is not yet published.**

   The description of the Campaign Management system in the first draft of DMTN-181 :cite:`DMTN-181` implies a few butler/pipeline middleware feature requests, and the processes and tooling for defining campaigns are expected to need new middleware support as well.
   This technote will attempt to aggregate those feature requests, while outlining a few possible approaches for how campaign definition and management will interact with the middleware, based on the absence or presence of potential new middleware functionality.

Introduction and Scope
======================

DMTN-181 :cite:`DMTN-181` describes a Campaign Management system that provides tooling for organizing and tracking the multiple workflows that comprise a pipeline processing campaign.
It also touches on what we will here call (for lack of a better term, so far) *Campaign Definition*: the tools and processes for determining the datasets, software, and configuration that will go into a campaign.
The datasets are our primary concern here, and for these :cite:`DMTN-181` focuses on the concept of exposure *exclusion lists* represented as *data ID sets*.
A data ID set is a table (at least conceptually) whose columns are data ID keys (butler data dimensions), with each data ID a row; :cite:`DMTN-181` proposes using ``git``-controlled JSON files as the source of truth for these, and uploading them to a table in the `Registry` database for use in `QuantumGraph` generation.
It also proposes that these be permanent database tables for provenance purposes.

The processes and tools for creating and maintaining exclusion lists are considered out of scope for campaign management by :cite:`DMTN-181`, but it correctly notes that these must be assembled from many disparate inputs that cannot be assumed to be present in any single database (let along the `Registry` database), so generating exclusion lists from these inputs implicitly as part of the queries that back `QuantumGraph` generation is not viable.
Defining those processes and tools fully is out of scope for this technote, too, but we will try to anticipate their demands on the middleware.
That includes support for using the data repository as a place to store and query metrics, dataset annotations, and other criteria that play a role in building exclusion lists, as well as making the `Registry` database the source of truth for curated data ID sets (such as the exclusion lists themselves).
It should be emphasized that this support has not been requested by those involved in pushing for and designing the Campaign Management system; on the contrary, they regard relying on the `Registry` database as over-centralization and poor separation of concerns.
These are valid worries - it is fair to say that the butler tries to do too much already, schema stability and migration are not trivial problems, and the middleware codebase currently has few contributors - but if the alternative is standing up, populating, and maintaining another SQL database with content similar to or substantially overlapping the `Registry`'s, then tighter coupling between middleware and Campaign Definition and middleware scope increases seem like the lesser evils.

.. note::

   The Consolidated Database probably has a role to play here; perhaps it is the database that should support Campaign Definition, and perhaps it is even a database that we should consider merging with the `Registry`.
   We don't have a clear understanding of what role (if any) it will play, so we have refrained from relying on it elsewhere in this document.
   It may be necessary to make significant edits to this technote after the Consolidated Database design and role are clarified.

It is not a given that a SQL database needs to exist to support Campaign Definition, and it seems overkill for generating and maintaining exclusion lists alone (and :cite:`DMTN-181` explicitly argues for the source of truth for exclusion lists, at least, to be ``git``-controlled files instead, which is at least somewhat in tension with this idea).
Specifying the datasets that are input to a campaign involves more than exclusion lists, however, at least in general.
In some cases, the extra information that must be provided to define a campaign's input datasets may be trivial, such as a cutoff date for Wide-Fast-Deep data release processing, and could plausibly be included in `QuantumGraph` generation via its queries against the `Registry` database (along with other query constraints for splitting up the campaign into separate workflows, which are more obviously a Campaign Management responsibility).
In test processing and commissioning, however, the input dataset specification may be much more complex.
Building that specification may involve significant investigatory work (the identification of the HSC RC, RC2, and pending RC3 subsets are excellent examples of this), which may draw on information from many of the same disparate sources as exclusion list generation.
Having a database to support Campaign Definition starts to make more sense in this context.

Furthermore, even in WFD DR processing, there are multiple different input dataset specifications, for different tasks in the pipeline, as our processing is not driven by inputs alone (`QuantumGraph` is a graph, not a flat list!).
So we must consider not just global filterig, but data ID *groups* and *relationships*.
Whether this is part of the definition of a campaign is, we believe, the single biggest point of contention about the scope of Campaign Management/Definition and its relationship to middleware, in large part because resolving it involves both technical considerations about how to represent data ID relationships and management/process considerations for how to assign responsibilities and schedule work.

As a concrete example, we'll consider building different kinds of coadds: while some coadds may attempt to include as many input visits as possible, others may include only include visits that with certain date ranges, that meet certain processing-generated criteria (e.g. PSF model quality), or are simply random subsets (e.g. for cross-validation).
We can imagine specifying the inputs to these coadds in three different ways:

- We might define which visits to include in each coadd explicitly, in advance, as part of the definition of a campaign, suggesting an implementation in which these relationships are also maintained in JSON data ID sets that are uploaded to the `Registry` database.

- In other cases, we might prefer (or need) to strictly define the data ID relationships, via dimension relationships stored in the `Registry` (as we do with exposure-visit snap membership).

- And finally, we can define relationships via filtering logic in the `PipelineTask` code itself, either during execution (in `run` or `runQuantum`) or during `QuantumGraph` generation (via the `PipelineTaskConnections.adjustQuantum` hook).

Which of these seems preferable depends on both the type of campaign and the grouping criteria; there is no one right answer, and all probably need to be supported to some degree.

This technote thus attempts to explore two separate but related questions about various potential new middleware features:

- How do they support *passing* data ID sets and data ID relationships from Campaign Management to middleware for QuantumGraph generation?

- How might they support *creating* data ID sets and data ID relationships in Campaign Definition?

One important aspect of the second question is how middleware support (or lack thereof) affects the tradeoffs involved in using middleware - and the `Registry` database in particular - to store and organize data relevant for Campaign Definition.

We organize this exploration as follows:

- In :ref:`current-middleware`, we describe how to meet Campaign Management/Definition needs with no new middleware features whatsoever.
  This will necessarily assume entirely external handling for Campaign Definition data, and impose inconveniences and rigidity on Campaign Management processes, but it serves as useful starting point.

- In :ref:`middleware-feature-requests`, we will walk though the various potential middleware enhancements under consideration, discussing how the improve (individually and in concert) upon the current level of middleware support for Campaign Management/Definition.

- In :ref:`other-drivers`, we will discuss other considerations driving some of the same middleware features, including Science Platform use cases.

- In :ref:`summary-and-recommendations`, we attempt to briefly synthesize this exploration into a few concrete recommendations about how Campaign Management/Definition should work, and which middleware features we should to prioritize to support it.
  This will not resolve all open questions, but we hope it provides some useful boundary conditions on the Campaign Management/Definition design and a framework for further discussion.

.. _current-middleware:

Campaign Definition/Management with Current Middleware
======================================================

.. _middleware-feature-requests:

Middleware Feature Requests
===========================

.. _feature-data-id-set-upload:

Data ID Set Upload
------------------

.. _feature-dynamic-dimensions:

Dynamic Dimensions
------------------

.. _feature-quantum-provenance:

Quantum Provenance
------------------

.. _feature-queryable-opaque-tables:

Queryable Opaque Tables
-----------------------

.. _feature-opaque-table-datastore:

Opaque Table Datastore
----------------------

.. _feature-public-registry-sql-interface:

Public Registry SQL Interface
-----------------------------

.. _feature-dataset-annotations:

Dataset Annotations
-------------------

.. _feature-per-task-quantum-graph-generation:

Per-Task QuantumGraph Generation
--------------------------------

.. _feature-in-memory-query-engine-for-quantum-graph-generation:

In-Memory Query Engine for QuantumGraph Generation
--------------------------------------------------

.. _other-drivers:

Other Drivers for Middleware Features
=====================================

.. _summary-and-recommendations:

Summary and Recommendations
===========================

.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
