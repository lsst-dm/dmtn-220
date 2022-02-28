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
So we must consider not just global filterig, but data ID *groups* and *relationships* as well.
Whether this is part of the definition of a campaign is, we believe, the single biggest point of contention about the scope of Campaign Management/Definition and its relationship to middleware, in large part because resolving it involves both technical considerations about how to represent data ID relationships and management/process considerations for how to assign responsibilities and schedule work.

As a concrete example, we'll consider building different kinds of coadds: while some coadds may attempt to include as many input visits as possible, others may include only include visits that with certain date ranges, that meet certain processing-generated criteria (e.g. PSF model quality), or are simply random subsets (e.g. for cross-validation).
We can imagine specifying the inputs to these coadds in three different ways:

- We might define which visits to include in each coadd explicitly, in advance, as part of the definition of a campaign, suggesting an implementation in which these relationships are also maintained in JSON data ID sets that are uploaded to the `Registry` database.

- In other cases, we might prefer (or need) to strictly follow the dimension relationships predefined in the `Registry`, as we do with exposure-visit snap membership relationships.

- And finally, we can define relationships via logic in the `PipelineTask` code itself, either during execution (in `run` or `runQuantum`) or during `QuantumGraph` generation (via the `PipelineTaskConnections.adjustQuantum` hook).

Which of these seems preferable depends on both the type of campaign and the grouping criteria; there is no one right answer, and all probably need to be supported to some degree.

This technote thus attempts to explore two separate but related questions about various potential new middleware features:

- How do they support *passing* data ID sets and data ID relationships from Campaign Management to middleware for QuantumGraph generation?

- How might they support *creating* data ID sets and data ID relationships in Campaign Definition?

One important aspect of the second question is how middleware support (or lack thereof) affects the tradeoffs involved in using middleware - and the `Registry` database in particular - to store and organize data relevant for Campaign Definition.

We organize this exploration as follows:

- In :ref:`current-middleware`, we describe how to meet Campaign Management/Definition needs with no new middleware features whatsoever.
  This will impose inconveniences and rigidity on Campaign Management/Definition processes, but it serves as useful starting point.

- In :ref:`middleware-feature-requests`, we will walk though the various potential middleware enhancements under consideration, discussing how they improve (individually and in concert) upon the current level of middleware support for Campaign Management/Definition.

- In :ref:`other-drivers`, we will discuss other considerations driving some of the same middleware features, including Science Platform use cases.

- In :ref:`summary-and-recommendations`, we attempt to briefly synthesize this exploration into a few concrete recommendations about how Campaign Management/Definition should work, and which middleware features we should to prioritize to support it.
  This will not resolve all open questions, but we hope it provides some useful boundary conditions on the Campaign Management/Definition design and a framework for further discussion.

.. _current-middleware:

Campaign Definition/Management with Current Middleware
======================================================

Exclusion Lists
---------------

At present, our query system and hence our `QuantumGraph` generation system do not provide a way to pass sets of data IDs in from files, and our query expression language - while flexible in other respects - has size limits that prohibit it from being used to pass in thousands of data IDs.

For ``raw``-data exclusion lists, the clear alternative is to use `TAGGED` collections, and it's arguable that this is better than uploading data ID sets anyway, at least in some respects (exclusion lists were actually one of the motivating use cases for `TAGGED` collections):

- Using a `TAGGED` collection very directly controls exactly one input dataset type, rather than data IDs that would apply to multiple input and output dataset types and task quanta all over the pipeline (at least until :ref:`feature-per-task-quantum-graph-generation` might allow them to be targeted more precisely).

- The `TAGGED` collection would naturally be persistent, rather than ephemeral as data ID set uploads would be (until :ref:`feature-queryable-opaque-tables`),as requested for provenance reasons by :cite:`DMTN-181`.
  Making a new `TAGGED` collection for each campaign and updating it within that campaign as necessary, seems a reasonable use of the collection system, as does maintaining one `TAGGED` collection representing our best current exclusion list.
  Neither of these provides strict reproducibility, as a `TAGGED` collection would still be subject to change after being used to drive processing, but we maintain that this is better handled by :ref:`feature-quantum-provenance` anyway.

Intermediate/Output Filtering
-----------------------------

We don't currently have any way to provide data ID sets in bulk to `QuantumGraph` generation that correspond to intermediate or output datasets.
That isn't seen as a significant limitation - in all cases at present, the output data IDs are either a direct or dimension-driven mapping from the exclusion list (e.g. ``exposure`` or ``visit`` dimensions), derived directly from what is possible given input collections, or are skymap tracts for which the number per workflow is limited by other constraints to be small enough to easily fit in the query expression.

`PipelineTask` code may already perform filtering of input datasets, during either  `QuantumGraph` generation or execution.
In both cases, the task's configuration and the fully-expanded data IDs are available to the filtering algorithm, and during execution datasets may of course be loaded and used to drive the filtering as well.
This is how filtering for coaddition works today: dedicated "selectImages" `PipelineTasks` process the per-visit summary datasets to produce a list of the visits that should go into a particular coadd, and then save that list to another dataset.
The coaddition tasks load those input-list datasets and use them to filter the actual input images they combine.

This works quite well in practice, and is probably the right long-term model for filtering that is driven primarily by thresholding values produced in earlier stages of processing.
It does not *require* campaign management to halt processing between steps to build an external data ID set that can then be validated in its own right - which could be very inconvenient, and at least poses scaling challenges for campaign management systems.
But it still permits campaign management to halt processing for validation when desired - the selection lists are regular butler datasets that can be loaded and analyzed in the usual way.
It is also quite possible for selections lists to be manually inspected, adjusted, and rewritten, even interactively (via `Butler.put`) as new datasets that are then used to drive processing, though this obviously would not scale.

Runtime filtering of this sort is not visible to the `QuantumGraph`, which would predict all visits as inputs to all coadds, modified only by the spatial overlap information it is aware of.
There is a combination of conditions under which this could be problematic for performance: if we perform batch processing by staging (in advance) all input datasets to local work-node storage, and if the number of filtered inputs is much smaller than the set of predicted inputs.

Data ID Relationships and Grouping
----------------------------------

The biggest limitation of runtime filtering of inputs is that it can't be extended to runtime definition of data ID relationships.
More precisely, we can (and do) use this runtime filtering to build different *types* of coadd, such as "deep" and "best-seeing", and this works because these correspond to different dataset types, produced by different tasks (or different configurations of the same task); this is a form of grouping, but the number of groups (types of coadd) is small and fully enumerated well in advance of processing.
We *could* also use it when building master calibrations, to remove bad or otherwise unsuitable frames from combination steps dynamically; as far as we know, this does not currently happen, but it would work because we never generate more than one master calibration for a particular detector (and filter, where appropriate) in a single `RUN` collection.
When multiple master calibrations for different validity ranges must be used as inputs together, they must first then be "certified" into a `CALIBRATION` collection.
What these supported cases have in common is that the group identifiers are not encoded in the output data ID; they are in some other term that we use to identify the dataset (i.e. the dataset type or collection).

For relationships where group identity is included in the data ID, the current middleware's only option is an extremely rigid one, in which each new kind of group must added to the dimensions configuration, triggering a `Registry` schema change and necessitating a migration.
As migrations go, these will be very simple and straightforward - they are entirely additions - but we do not have a process or tooling to automate those kinds of changes, and we still track them in our schema versioning system.
After the schema is updated, dimension records can be inserted via `Registry` Python APIs to define both the set of allowable output data ID keys and their relations.
This is the system currently used (to at least hypothetically) relate ``exposure`` snaps to the ``visits`` they belong to.
It works well for this because we want rigidity here: these relationships are should be constant across processing runs, because we really don't want the definition of a visit to change across different collections.
We can populate the tables that define the snap/visit relationships very early, using raw header metadata that we already ingest into the `Registry's` ``exposure``, and then essentially never touch it again (once raw header metadata and its translation settles down, that is).

A similar approach seems like it would work tolerably well for yearly or other short-period coadds: define a "year" dimension in advance, and use the butler's existing temporal-join capabilities to relate that timespan directly to the visits that overlap it (with a bit of extra filtering in the task to deal with edge cases).

The data ID relationship use cases where none of these approaches work well are a bit harder to find, which may just mean that they aren't fully described anywhere (and aren't in our running pipelines precisely because we don't support them well yet).
As one example, we certainly don't have a good way to handle pair-of-observations image differences, though it's still unclear whether we will need those (clearly it would be nice to have the option); note that a ``visit``-like approach is a poor fit there because the pairs we might want to difference are as likely to change between runs as stay the same.
Out-of-focus image processing for the active optics system or PSF modeling may also have use cases that aren't well supported by current middleware.
At present the wavefront-processing tasks take the same approach as our master-calibration combination tasks, and use collections to separate groups, but I don't know whether this is a good, a pragmatic choice given the lack of alternatives, or a lack of awareness of the alternatives that do exist (or might exist).
Building coadds from random or systematically distributed subsets of the available input visits (e.g. "even visits only" or seeing percentiles) using in-task filtering and different dataset types for different parameter values requires embedding those parameters into the dataset type name, which is a bit ugly, but hardly sufficient reason on its own to implement substantial new middleware functionality.

Databases for Campaign Definition
---------------------------------

The `Registry` guarantees that all of the tables and other database entities it produces can be confined to a single schema (in the namespace sense), allowing external tables in other schemas to safely coexist within the same database.  This theoretically allows those external tables and `Registry` tables to be used together in queries, and in many cases the `Registry` tables have straightforward, easy-to-interpret columns that would work well for this (especially for dimension tables, which are the ones that would probably be of most interest to campaign definition).
This would probably work reasonably well right now, but it is not documented and formally not supported, and hence currently inadvisable for anything other than throwaway prototypes - while we originally intended to make the SQL interface public, this became very difficult to implement for a number of reasons, and it has been explicitly private for a few years now.
Changing that is discussed in :ref:`feature-public-registry-sql-interface`.

It is also possible to use `Registry` interfaces to define custom "opaque" tables within the same schema as its main tables.
This could make it easier to manage external tables across multiple similar data repositories, and it allows those tables to make use custom field types like `sphgeom` regions, timespans, and UUIDs that require cross-DBMS support beyond what is provided by SQLAlchemy alone.
This is the preferred mechanism for "plugin" code built on top of the `Registry` that needs its own tables, and it is already in use by some of our own `Datastore` classes to store their internal per-file records.
At present, however, the `Registry` query system cannot use these tables at all; they are truly opaque to it.
Changing this is discussed in :ref:`feature-queryable-opaque-tables`.

Without either of the two changes discussed above, the best way to add new tables and metadata columns to the `Registry` schema is thus to change the `Registry` schema itself, by modifying its "dimensions" configuration.
This is already a very flexible system that allows arbitrary new tables with typical column types to be added (and later populated using existing `Registry` public methods), and it includes support for foreign keys between dimensions, allowing new tables tables to define relationships, not just metadata.
Such tables are automatically included in `Registry` queries as needed; using a configuration system to define these tables (rather than e.g. SQL ``CREATE TABLE`` statements) allows us to also obtain the information necessary to automatically join them together.
This naturally meshes well with a model in which Campaign Definition workflows explicitly provide data ID relationships as inputs to campaigns, especially if it is considered a feature rather than a bug if those data ID relationship tables are persistent in the database rather than transient (deletion from dimension tables is not currently supported at all).

The main drawback of editing the dimensions configuration is that it is tracked by the butler's schema versioning and migration system, so any edits (even trivial ones, like new tables and columns) require a new version and migration scripts.
This may be a blessing in disguise - any production-ready Campaign Definition system backed by a SQL database *should* be thinking rigorously about schema versioning and migration, and it may be easier to use the butler's system than build a new one.
One the other hand, the butler's migration system does have to account for kinds of complexity (e.g. dynamically-defined tables) that a more independent Campaign Definition database might not have, which has forced us to build our own layer on top of Alembic rather than using it or some other third-party tool more directly or naturally.

Finally, we should point out that it is always possible to use the butler to store Campaign Definition data (tabular or otherwise) in regular butler datasets.
This naturally associates them with the dimensions schema via their data ID, and it should be the first choice for anything produced or consumed by a `PipelineTask` (such as metric measurements or explicit input-visit lists of the sort we use in coaddition today).
There are a few limitations that should be taken into account when considering using butler datasets for Campaign Definition storage, however:

- Datasets may not be updated in place - they are written atomically for each data ID.
- We don't currently have a good solution for rolling up small datasets (e.g. metric measurements or even per-detector catalogs) into larger files that can be *much* more efficient to read (see :ref:`feature-opaque-table-datastore` for a potential solution).
- Dataset content cannot be used to directly drive `QuantumGraph` generation (which could be addressed by a combination of :ref:`feature-queryable-opaque-tables` and :ref:`feature-opaque-table-datastore`).

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
