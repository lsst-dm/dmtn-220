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

DMTN-181 :cite:`DMTN-181` describes at a high level a Campaign Management system that provides tooling for organizing and tracking the multiple workflows that comprise a pipeline processing campaign.
RTN-023 :cite:`RTN-023` describes in more detail a prototype for at least some of such a system, focused on tooling and procedures for managing many batch submission workflows.
This document addresses design questions about the Campaign Management boundary with the lower-level aspects of the middleware - the `Butler` and `PipelineTask` systems in particular.
It has relatively little overlap with the side of Campaign Management discussed by :cite:`RTN-023`, for which the relevant middleware interface is primarily the Batch Processing Service, though it does provide some alternatives for how state and provenance for workflow management could be stored.

Instead, the primary focus here is we will here call (for lack of a better term, so far) *Campaign Definition*: the tools and processes for determining the datasets, software, and configuration that will go into a campaign.
The datasets are our primary concern here, and for these :cite:`DMTN-181` focuses on the concept of exposure *exclusion lists* represented as *data ID sets*.
A data ID set is a table (at least conceptually) whose columns are data ID keys (butler data dimensions), with each data ID a row; :cite:`DMTN-181` proposes using ``git``-controlled JSON files as the source of truth for these, and uploading them to a table in the `Registry` database for use in `QuantumGraph` generation.
It also proposes that these be permanent database tables for provenance purposes.

The processes and tools for creating and maintaining exclusion lists are considered out of scope for campaign management by :cite:`DMTN-181`, but it correctly notes that these must be assembled from many disparate inputs that cannot be assumed to be present in any single database (let along the `Registry` database), so generating exclusion lists from these inputs implicitly as part of the queries that back `QuantumGraph` generation is not viable.
Defining those processes and tools fully is out of scope for this technote, too, but we will try to anticipate their demands on the middleware.
That includes support for using the data repository as a place to store and query metrics (see also DMTN-203 :cite:`DMTN-203`), dataset annotations (DMTN-204 :cite:`DMTN-204`), and other criteria that play a role in building exclusion lists, as well as making the `Registry` database the source of truth for curated data ID sets (such as the exclusion lists themselves).
It should be emphasized that this support has not been requested by all of those involved in pushing for and designing the Campaign Management system, and some participants have expressed concerns that relying on the `Registry` database for these is over-centralization and poor separation of concerns.
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
So we must consider not just global filtering, but data ID *groups* and *relationships* as well.
Whether this is part of the definition of a campaign is, we believe, the single biggest point of contention about the scope of Campaign Management/Definition and its relationship to middleware, in large part because resolving it involves both technical considerations about how to represent data ID relationships and management/process considerations for how to assign responsibilities and schedule work.

As a concrete example, we'll consider building different kinds of coadds: while some coadds may attempt to include as many input visits as possible, others may include only include visits within certain date ranges, that meet certain processing-generated criteria (e.g. PSF model quality), or that are simply random subsets (e.g. for cross-validation).
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

- The `TAGGED` collection would naturally be persistent, rather than ephemeral as data ID set uploads would be (until :ref:`feature-queryable-extension-tables`), as requested for provenance reasons by :cite:`DMTN-181`.
  Making a new `TAGGED` collection for each campaign and updating it within that campaign as necessary, seems a reasonable use of the collection system, as does maintaining one `TAGGED` collection representing our best current exclusion list.
  Neither of these provides strict reproducibility, as a `TAGGED` collection would still be subject to change after being used to drive processing, but we maintain that this is better handled by :ref:`feature-quantum-provenance` anyway.

Intermediate/Output Filtering
-----------------------------

We don't currently have any way to provide data ID sets in bulk to `QuantumGraph` generation that correspond to intermediate or output datasets.
That isn't seen as a significant limitation - in all cases at present, the output data IDs are either a direct or dimension-driven mapping from the exclusion list (e.g. ``exposure`` or ``visit`` dimensions constrained by ``raw`` existence), derived directly from what is possible given input collections, or are skymap tracts for which the number per workflow is limited by other constraints to be small enough to easily fit in the query expression.

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
When multiple master calibrations for different validity ranges must be used as inputs together, they must first be "certified" into a `CALIBRATION` collection.
What these supported cases have in common is that the group identifiers are not encoded in the output data ID; they are in some other term that we use to identify the dataset (i.e. the dataset type or collection).

For relationships where group identity is included in the data ID, the current middleware's only option is an extremely rigid one, in which each new kind of group must added to the dimensions configuration, triggering a `Registry` schema change and necessitating a migration.
As migrations go, these will be very simple and straightforward - they are entirely additions - but we do not have a process or tooling to automate those kinds of changes, and we still track them in our schema versioning system.
After the schema is updated, dimension records can be inserted via `Registry` Python APIs to define both the set of allowable output data ID keys and their relations.
This is the system currently used (to at least hypothetically) relate ``exposure`` snaps to the ``visits`` they belong to.
It works well for this because we want rigidity here: these relationships should be constant across processing runs, because we really don't want the definition of a visit to change across different collections.
We can populate the tables that define the snap/visit relationships very early, using raw header metadata that we already ingest into the `Registry's` ``exposure``, and then essentially never touch it again (once raw header metadata and its translation settle down, that is).

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

It is also possible to use `Registry` interfaces to define custom "opaque" tables within the same schema as its main tables.
This could make it easier to manage external tables across multiple similar data repositories, and it allows those tables to make use of custom field types like `sphgeom` regions, timespans, and UUIDs that require cross-DBMS support beyond what is provided by SQLAlchemy alone.
This is the preferred mechanism for "plugin" code built on top of the `Registry` that needs its own tables, and it is already in use by some of our own `Datastore` classes to store their internal per-file records.
At present, however, the `Registry` query system cannot use these tables at all; they are truly opaque to it.
Changing this is discussed in :ref:`feature-queryable-extension-tables`.

Without a public schema or queryable extension tables, the best way to add new tables and metadata columns to the `Registry` schema is thus to change the `Registry` schema itself, by modifying its "dimensions" configuration.
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
- We don't currently have a good solution for rolling up small datasets (e.g. metric measurements or even per-detector catalogs) into larger files that can be *much* more efficient to read (see :ref:`feature-table-backed-datastore` for a potential solution).
- Dataset content cannot be used to directly drive `QuantumGraph` generation (which could be addressed by a combination of :ref:`feature-queryable-extension-tables` and :ref:`feature-table-backed-datastore`).

.. _middleware-feature-requests:

Middleware Feature Requests
===========================

This section describes in detail various planned or in-progress middleware features that we expect to be of interested to Campaign Management/Definition.
All of them are things we'd like to do eventually, and many have other drivers (see :ref:`other-drivers`).
None are trivial, however, and the needs of Campaign Management/Definition should be considered in their prioritization.

This section assumes more knowledge about middleware concepts and terminology than the rest of the document.
Non-expert readers may want to skip it and go directly to :ref:`summary-and-recommendations`, especially on a first read.

.. _feature-data-id-set-upload:

Data ID Set Upload
------------------

.. note::
   This feature is tracked as `DM-30438 <https://jira.lsstcorp.org/browse/DM-30438>`__  and `DM-33621 <https://jira.lsstcorp.org/browse/DM-33621>`__, which depend on `DM-31725 <https://jira.lsstcorp.org/browse/DM-31725>`__.

This feature gives the butler query system the ability to accept data ID sets from external Python objects and files, uploading them to temporary tables for the duration of a single query or small set of queries (within a single context-manager block).
This will be integrated into `QuantumGraph` generation, allowing external data ID sets to directly constrain that process.

As a temporary upload, this feature does not fully provide the minimal middleware functionality requested by :cite:`DMTN-181`, but as noted earlier,  ``TAGGED`` collections are probably a better tool anyway for using exclusion lists or otherwise providing fine-grained control over input datasets, especially if the lists should remain persistent in the `Registry`.
Making data ID set uploads persistent will require both this feature and :ref:`feature-queryable-extension-tables`.

Temporary data ID set uploads do provide key functionality that ``TAGGED`` collections do not, however, in that they allow explicit external filtering or grouping for intermediate and output datasets and quanta, not just input datasets.
Even this is fairly limited unless other features are implemented as well, however:

- Without :ref:`feature-dynamic-dimensions`, data ID set upload can only be used to filter or define relationships between existing dimensions, and since in practice all dimension combinations that could plausibly be related are already related (usually via spatial overlaps), any external data ID sets must be subsets of those that would be produced by the `Registry`'s default joins between those dimensions.
  New long-lived dimensions could be added to the configuration (with a single up-front schema migration) that could be designed to always require a data ID set upload to set relationships, however, and it *may* make sense to redefine the ``physical_filter`` - ``band`` relationship this way after data ID set upload lands - the current identification of each ``physical_filter`` with exactly one ``band`` seems like a "usually true" convenience that we should back away from enforcing as soon as our data model can reasonably support that.

- Without :ref:`feature-per-task-quantum-graph-generation`, each data ID set constrains quanta and datasets for all tasks and dataset types in the QuantumGraph that involve its dimensions.
  For example, it does not provide a way to use different data ID sets for e.g. different types of coadds, unless each type of coadd is produced via a different QuantumGraph.

This feature is difficult to implement only in the sense that it involves a piece of the codebase (the `lsst.daf.butler.registry.queries` subpackage) that requires a lot of work more generally; we have slowly added more and more functionality there "pragmatically" over the past several months to the point where class roles and encapsulation are quite tangled, and some of those ill-fitting additions are pieces we would like to build upon when implementing data ID set upload.
`DM-31725 <https://jira.lsstcorp.org/browse/DM-31725>`__ captures at least the initial prototyping work for the necessary refactor, and once it's done adding data ID upload itself should be quite easy.
As a result, it's safe to say that we will deliver this functionality eventually, even if it isn't needed for Campaign Management/Definition, but making it a high priority for such usage won't necessarily make it something we can deliver quickly.

.. _feature-dynamic-dimensions:

Dynamic Dimensions
------------------

.. note::
   This feature is tracked as `DM-33751 <https://jira.lsstcorp.org/browse/DM-33751>`__ and depends on `DM-31725 <https://jira.lsstcorp.org/browse/DM-31725>`__ in Jira.

In its minimal form, this feature allows butler dataset types to be defined  and datasets of those types read and written with data ID keys that are not part of the static dimensions configuration that defines much of the `Registry` schema.
Unlike static dimensions, these dynamic dimensions would not be expected to have values that could be iterated over or enumerated independently of the datasets they identify, and hence there are no guarantees that those values take on the same meaning in different collections.
They also would not be associated with metadata or have foreign keys or natural relationships to other dimensions.

That makes it hard to use these dimensions in `QuantumGraph` generation, at least in any role other than pure input datasets.
A slightly less minimal form of the feature could permit custom data ID keys in intermediate, output, and quantum data IDs if they had the same, constant value over the entire `QuantumGraph`.
Where this functionality really shines is in combination with :ref:`feature-data-id-set-upload`, which could allow external data ID sets to relate dynamic dimensions to each other and existing static dimensions, providing fine-grained external control over the grouping done by `QuantumGraph` generation.

It's hard to guess right now how difficult this feature would be to implement; generally speaking, registering dataset types with dynamic dimensions seems easy, but making those queryable later in the usual way seems hard, as we'd need to use subqueries on dataset-collection join tables in parts of the query system where we can usually rely on pure dimension tables existing, and this both inverts our usual process for query-building (start with dimensions, then look up datasets) and forces us to remember more about what we've already joined in to avoid unnecessarily including the same table in a query multiple times.
It also seems that we'll need to modify the static schema a bit to remember the new dynamic dimensions - they can't be purely ephemeral, after all, if they are used to identify persistent datasets.
Certainly we'll want to at least tackle `DM-31725 <https://jira.lsstcorp.org/browse/DM-31725>`__ first, to get the query system in a state where we could contemplate an extension like this.

.. _feature-quantum-provenance:

Quantum Provenance
------------------

.. note::
   This feature is fully described in DMTN-205 :cite:`DMTN-205`.

Quantum Provenance here refers to storing the as-run `QuantumGraph` in the data repository (and in particular new `Registry` tables), as well as providing tools to traverse that graph in order to (among other things) reproduce previous processing runs.
This is functionality that we have intended to include in the middleware since its inception, and while fully implementing it is still a major project, there is no question that it will ultimately get done.

This is relevant for Campaign Management/Definition primarily because it draws a clear boundary between the provenance information and use cases that will be handled by the middleware provenance system and use cases that must be handled by Campaign Management/Definition.
In particular, middleware provenance is aimed at rigorously solving the problem of exactly reproducibility, starting from a saved and queryable `QuantumGraph`, but it largely punts on providing any reproducibility for `QuantumGraph` generation, as (from its perspective, at least) the inputs to `QuantumGraph` generation are mutable.
Campaign Management could extend reproducibility earlier only by similarly taking care to depend only on immutable entities (such as git-controlled data ID lists) or limit via policy how other entities are modified in practice (e.g. "freezing" per-submission ``RUN`` collections after a batch job completes, and not using ``CHAINED`` collections as inputs to `QuantumGraph` generation).
And it may be better to make no such attempt (at least not at *rigorous* reproducibility), since reproducibility starting from the `QuantumGraph` is already quite powerful.

One subtlety of quantum provenance is that while it will not save the exact data ID sets passed in to `QuantumGraph` generation (when passing in data ID sets is implemented), it essentially will save the subsets of those sets that are consistent with each other and the other inputs to `QuantumGraph` generation.
More precisely, if the dimensions of the data ID sets are recorded externally, one can obtain from quantum provenance data ID sets with those dimensions that will produce the same `QuantumGraph`, provided other constraints (such as input collection contents) have not changed.
This *may* make it unnecessary for Campaign Management/Definition to store the data ID sets it uses directly.

.. _feature-queryable-extension-tables:

Queryable Extension Tables
--------------------------

.. note::
   This feature depends on `DM-31725 <https://jira.lsstcorp.org/browse/DM-31725>`__ in Jira.
   It does not have a tracking ticket of its own yet.

As discussed in :ref:`current-middleware`, the butler registry already has an interface that allows external code to create custom tables in the same database.
This is used by `Datastore` implementations to save information about each file, and it could be used by Campaign Management/Definition as a tabular storage mechanism.
At present these tables are completely opaque to the registry, however, and hence they can't be used to constrain registry queries or `QuantumGraph` generation.

Allowing these extension tables to participate in those queries could be extremely powerful:

- with :ref:`feature-data-id-set-upload`, it would allow data ID set uploads to be persistent, not just temporary;

- Campaign Management/Definition could use these tables to save observational metadata, quality flags, campaign/workflow provenance, etc. within the `Registry`, and then include constraints on that information during `QuantumGraph` generation or when using `Registry` queries to create data ID sets.

- with :ref:`feature-table-backed-datastore`, metric datasets produced by `PipelineTasks` could also be used to similarly constrain queries and `QuantumGraph` generation.

To include an extension table in a registry query, the extension code would need to declare one or more special columns that the query system already knows how to include in its joins, such as dimension values, dataset UUIDs, spatial regions, and timespans.

In addition to the general query-system work (`DM-31725 <https://jira.lsstcorp.org/browse/DM-31725>`__), the main challenge in implementing this ticket is figuring out how the column metadata provided by the extension code should be persisted in the data repository.
There are two main options:

- We could add new static tables whose rows record the schemas of extension tables, and populate them when those extensions are first registered.

- We could require extension code that conforms to a specific schema-introspection interface to be referenced in the data repository or butler client configuration as an importable type string.

To select between these we probably need to think about how we want to handle changes to extension table schemas.
We probably don't want extension tables to participate in the butler's internal data repository versioning or migration system, except in a very limited way when an internal butler column used as a foreign key (e.g. dataset UUID or dimension) is changed in a backwards-incompatible way.
That's something we can hopefully avoid ever doing, because it's extremely painful no matter how the migration is managed.
But we do want extensions to be able to change the schemas of their own tables and give them the tools they need to do this in managed, backwards-compatibility-focused way.

.. _feature-table-backed-datastore:

Table-Backed Datastore
----------------------

.. note::
   This feature is tracked as `DM-13362 <https://jira.lsstcorp.org/browse/DM-13362>`__ in Jira.
   It is also closely related to the subject of DMTN-203 :cite:`DMTN-203` (which hasn't yet been written).

This feature implements a new concrete `Datastore`, which would use the registry's "opaque table" mechanism to store dataset contents entirely within the registry database.
During batch execution, these records would be exported to the `QuantumGraph` when their datasets are needed as inputs, and they would initially be written to per-quantum files that would need to be merged prior to upload into the registry database.

This storage makes sense only for very small datasets, and if it's only a small-dataset optimization, using the SQL registry database instead of direct multi-dataset file storage (e.g. Parquet) is unlikely to be ideal.
But it might be a lot easier to implement if we need help avoiding a proliferation of tiny files in a hurry.

What's more relevant for Campaign Management/Definition is the combination of this feature with :ref:`feature-queryable-extension-tables`, in which the records that back these datasets become queryable, and this `Datastore` becomes ideal for metric datasets, which are essentially single values that we want to be usable as query constraints that are joined into queries according to the dimensions the metric measurements are associated with.

.. _feature-dataset-annotations:

Dataset Annotations
-------------------

.. note::
   This feature is fully described in DMTN-204 :cite:`DMTN-204`.

This collection of features involves ways to add annotations to a number of butler entities, such as datasets, dimension records, and collections.
Many of these annotations are intended to provide information to Campaign Management/Definition processes, and it is an important question whether having them in the butler makes sense for Campaign Management/Definition workflows.

It is generally true that the butler provides a good organizational structure for these annotations, and in the absence of arguments against, we probably should put them in the butler instead of setting up a similar organizational structure elsewhere.

As discussed in :cite:`DMTN-204`, one option for implementing these is to use the existing opaque table storage system; when combined with :ref:`feature-queryable-extension-tables`, these annotations would also be usable in registry queries and `QuantumGraph` generation.
With :ref:`feature-table-backed-datastore` as well, it may even be possible to implement them fully as regular butler datasets (which, when viable, is a much better-understood and low-risk extension point than using the opaque table interface directly).

.. _feature-per-task-quantum-graph-generation:

Per-Task QuantumGraph Generation
--------------------------------

.. note::
   This feature is tracked as `DM-21904 <https://jira.lsstcorp.org/browse/DM-21904>`__ and depends on `DM-31725 <https://jira.lsstcorp.org/browse/DM-31725>`__ in Jira.

We have long had a pseudocode algorithm in hand for `QuantumGraph` that addresses a number of current limitations, on `DM-21904 <https://jira.lsstcorp.org/browse/DM-21904>`__.
Its implementation has been blocked by a lack of butler query-system functionality (essentially :ref:`feature-data-id-set-upload`) that we have been unable to prioritize.

This algorithm is relevant for Campaign Management/Definition because it allows filters on data IDs - whether provided by data ID sets or boolean contraint expressions - to be specific to certain tasks or dataset types, for example allowing one data ID set to be used for one kind of coadd, and another data ID set to be used for a different type of coadd.
Without this, the only way to have different tasks operate on different sets of input data IDs is via task code in either `PipelineTaskConnections.adjustQuantum` or `PipelineTask.runQuantum`.

.. _other-drivers:

Other Drivers for Middleware Features
=====================================

Many of the features described here are important for other DM needs, and when considering the total development cost of a new Campaign Management/Definition, it makes sense to consider these "discounted" (or at least easier to prioritize) at some level.

- `DM-31725 <https://jira.lsstcorp.org/browse/DM-31725>`__ has come up repeatedly here as a blocker for other features.
  It also blocks vectorized calibration-dataset lookup (the absence of which is frequently the bottleneck in `QuantumGraph` generation) and support for `PipelineTasks` whose dimensions include HEALPix or HTM (such as a task to produce HiPS maps).

- Without :ref:`feature-per-task-quantum-graph-generation` / `DM-21904 <https://jira.lsstcorp.org/browse/DM-21904>`__, some tasks in the DRP pipeline cannot safely be run as part of the same submission, forcing the pipeline to be split up into more steps than we would like.

- :ref:`feature-dynamic-dimensions` / `DM-33751 <https://jira.lsstcorp.org/browse/DM-33751>`__ is one of two possible solutions to the problem of how the image cutout service should identify its output datasets (the other is allowing some dataset types to have non-unique data IDs within a run, which is less generally useful).
  We expect this to be useful more broadly in Science Platform services and user-defined processing, which is likely to want data ID keys other than those needed for our own processes.

- :ref:`feature-quantum-provenance` plays a key role in satisfying fundamental middleware requirements.

.. _summary-and-recommendations:

Summary and Recommendations
===========================

Recommendations
---------------

Based on current middleware capabilities, the perceived difficulty of extending those capabilities in the ways described above, and our guesses at how difficult it would be to stand up and maintain non-middleware (or less-middleware-based) solutions to various problems, we make the following recommendations for Campaign Management/Definition:

#. We should mandate a one-to-one relationship between ``RUN`` collections and workflows.
   This allows per-workflow provenance and metadata to be saved as butler datasets without new dimensions, and it automatically associates that per-workflow information with the datasets produced by the workflow.
   If this leads too problems with query performance due to too many ``RUN`` collections in certain ``CHAINED`` collections, we have a few avenues of optimization we can explore to address it.

#. We should use ``TAGGED`` collections referenced through per-instrument ``CHAINED`` collection pointers to maintain the official, current-best ``raw`` exclusion list (as an inclusion list).
   This is analogous to the approach taken for the official, current-best suite of calibrations.
   Exclusion lists used for particularly important campaigns may be saved by creating a new ``TAGGED`` collection snapshot with a new name, making it much easier to reproduce QuantumGraphs generated for that campaign later (but note that this is unnecessary to reproduce processing, once the original QuantumGraphs are also saved in the data repository).
   Similarly, major updates to the exclusion list could be performed by creating a new exclusion list and updating the ``CHAINED`` collection pointer, rather than modifying the current one in-place.
   If maintaining a more fine-grained history of the exclusion list's evolution is important, this history should probably be stored outside the data repository in a git repository or perhaps in butler datasets.

#. Metric values produced by PipelineTasks should be obtained by Campaign Management/Definition directly from the butler, either as the datasets via `Butler.get`, or via the registry query system (with :ref:`feature-queryable-extension-tables`, and :ref:`feature-table-backed-datastore`).
   In particular, it does not make sense to export metrics into a different relational database in order to make them queryable according to the data ID dimensions they correspond to.
   While querying the SQuaSH InfluxDB probably makes more sense for history-of-the-pipelines temporal queries (and many metric values will be uploaded to it anyway), these are not the kinds of queries on metrics we expect Campaign Management/Definition to be performing for the most part.

#. We should not attempt to include all observational metadata in butler data repositories.
   There is considerable value in keeping simple external systems (such as exposure logging) at most loosely coupled to the butler, and the EFD is of course much better for querying the full breadth of time-organized observatory state information.
   Whenever Campaign Definition processes consume observational metadata from a non-butler data source, however, we should at least consider whether it or some summary of it should be included in the data repository, either as a dataset or a dimension column, especially when the production resembles a major operations one, like data release or alert production (as opposed to ad-hoc hardware commissioning activities).
   Campaign Definition queries will often be good predictor of things science users or pipelines developers will want to do, and we don't want that to involve too many heterogeneous sources of data.

#. Similarly, when standing up new systems to track and store observational metadata or Campaign Management state (or deciding whether to stand up a new one or piggyback on something else), we should prefer *not* to use a butler data repository when the organizational structure is extremely simple (e.g. strictly per exposure or per workflow) or organized along dimensions not in the butler data model (continuous variables like time, or processing units such as campaigns), in order to limit requirements on the butler and keep coupling loose.
   When the organizational structure is more relational and it involves dimensions already included in the butler data model, we should lean towards building this new system in or on top of the butler data repository.
   This is especially true for any system that describes or annotates processing outputs, as the butler's relational model for these is not something we should attempt to replicate in some other database.

#. We should prefer in-task mechanisms or static dimensions over explicit input data ID sets for grouping the input datasets to tasks.
   Requiring input data ID sets to define groups for a pipeline takes the work of defining groups out of a system that has been designed for automated, at-scale operation (task execution or `QuantumGraph` generation) and puts it in one that typically involves much more human intervention (Campaign Management/Definition), creating an artificial scaling challenge.
   It would also force workflows to be split between tasks that produce inputs to group-definition and tasks that operate on those groups, rather than allow them to be run together in a single workflow.
   This recommendation explicitly extends to all kinds of coadds and other many-visit processing of science images.
   Known exceptions to this recommendation include:

   - Pairwise processing of visits or exposures *other* than the grouping of snaps into visits (which is handled by static dimensions): if the pair definitions are as likely to change between runs as stay the same, and writing each pairwise output to a separate dataset type/collection is not viable, we should should use :ref:`feature-data-id-set-upload` and :ref:`feature-dynamic-dimensions`.

   - Small-group tasks that are not pairwise, such as focus sweeps.
     We do not understand these use cases in enough detail to make a recommendation for them, but note that most similar ones we are aware of - traditional master calibration productions like flats, biases, and darks - operate with one output per detector (and filter, where appropriate) per collection, and hence can be considered a filtering problem rather than a grouping problem.

   - ``physical_filter`` - ``band`` associations are currently statically defined in the dimension tables, but it would be better in the long term - after :ref:`feature-data-id-set-upload` lands - to use a data ID set upload to define these relationships (and indeed define what ``band`` values mean) on a per-campaign or per-production basis.

Prioritization
--------------

All of the middleware features described in :ref:`middleware-feature-requests` are things we'd like to get done eventually, and `DM-31725 <https://jira.lsstcorp.org/browse/DM-31725>`__ (a blocker for nearly all of them) is a high priority, albeit a large one that we've long struggled to make progress on, even before that ticket existed in its current form.
:ref:`feature-quantum-provenance` is a notable exception: it is not blocked by that ticket, and is instead something of a competitor with most of the rest of the features described here for prioritization.
It is not currently a high priority, though we are taking some steps toward it as a side effect of ongoing work towards replacing our "execution butler" approach to batch scaling (see DMTN-177 :cite:`DMTN-177` and `DM-33500 <https://jira.lsstcorp.org/browse/DM-33500>`__).

Other major middleware work packages that will compete with these for prioritization include:

- Making usage of ``day_obs`` and ``seq_num`` more natural and broadly supported across butler interfaces, and in ``visit`` as well as ``exposure`` (`RFC-836 <https://jira.lsstcorp.org/browse/RFC-836>`__, `DM-30439 <https://jira.lsstcorp.org/browse/DM-30439>`__).

- Developing an https client/server registry that could be used by science users (`DM-27569 <https://jira.lsstcorp.org/browse/DM-27569>`__).

- General polish and documentation work, especially in the pipeline definition, `QuantumGraph` generation, and small-scale execution system.

Once `DM-31725 <https://jira.lsstcorp.org/browse/DM-31725>`__ and the general query system overhaul is complete, understanding the relative importance of queryable metrics and annotations:

- :ref:`feature-queryable-extension-tables`
- :ref:`feature-table-backed-datastore`
- :ref:`feature-dataset-annotations`

vs flexibility in controlling `QuantumGraph` generation:

- :ref:`feature-data-id-set-upload`
- :ref:`feature-dynamic-dimensions`, - :ref:`feature-per-task-quantum-graph-generation`

will be the next question for prioritizing of Campaign Management/Definition support work.
The answer may depend largely on how well the current middleware supports running out-of-focus image processing tasks; these seems to be the place where data ID set upload and dynamic dimensions are most likely to be critical, and where the middleware team's understanding of what is needed is weakest.

Most of the other Campaign Management/Definition and commissioning tasks we anticipate actually seem well-enough supported by current middleware to begin commissioning and even perform some large-scale processing (as we are already doing with DP0.2), even if there is room for improvement on virtually every front.

.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
