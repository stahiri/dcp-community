### DCP PR:

[dcp-community/rfc13](https://github.com/HumanCellAtlas/dcp-community/pull/88)


# RFC: Data Groups: Processing Datasets that Span Multiple Bundles

## Summary
This RFC proposes a solution to allow processing of datasets that span multiple bundles. It addresses the general problem that the current DCP data model contains no representation of data set grouping and version-completeness beyond that of individual data bundles.  The current DCP model is restricted in that all data processing can be a one-to-one linear sequence of steps deriving one bundle from another.

However, there are use cases where a given processing step can require input from multiple input bundles, resulting in a directed acyclic graph (DAG) rather than a linear graph. The relevant DAG sub-graph must be defined and all of its associated data available to correctly initiate processing.  A notion and implemented representation of data completeness are required to support DAG processing.

This RFC defines a general mechanism for grouping bundles, here called *data groups*. By providing a general method for grouping bundles, this proposal provides a mechanism for addressing various tasks that cross bundle boundaries.  This has an impact on analysis, pipelines, algorithmic complexity of data processing, data consistency, and data set quality control.  The current bundle grouping, described in detail here, is a submission to a project from external producers or analysis .  However, the concept generalizes in a manner that other groupings can be defined to accommodate future needs.

## Author(s)
- [Nick Barkas](mailto:barkasn@broadinstitute.org)
- [Mark Diekhans](mailto:markd@ucsc.edu)

## Shepherd
- [Nick Barkas](mailto:barkasn@broadinstitute.org)

## Motivation
The current DCP model of processing has the scope of a single data bundle, which contains a single assay or analysis.  There is no mechanism to know when a group of bundles that need to be processed together is complete and available for further processing.  This results in the limitation that data processing is linear and is entirely based on ingestion packaging resulting in multiple problems, as outlined below.

#### Multi-input analysis 
Analysis pipelines may require input from multiple different assays. To perform such analyses, they must be triggered only when all input data is available. The current mechanism of bundle notifications is not effective when data spans multiple bundles because there is no clear set of rules to identify the bundles that need to be co-processed.  This limits the downstream process to the scope of a single bundle and requires Ingest to define the scope of all subsequent processing without foreknowledge of future needs.

An immediate need for analysis pipelines to process 10X V2 scRNA-seq datasets that span multiple bundles exists. The need for co-processing multiple bundles is not however limited to this scenario and extends to any other assay modality that involves repeated sequencing of the same library. Furthermore, the need to co-process data may extend to other data modalities that the DCP will accept in the future.

In the future, a need to run analysis processes where input sets span multiple projects may arise. A mechanism to specify processing data collections that are not restricted to a single project will be required to support this functionality.

#### Algorithmic complexity and performance
Producing combined metadata or data from multiple assay bundles using the bundle event system results in O(N^2) algorithmic complexity, where N is the number of data bundles in some grouping. As each bundle event arrives, it must be combined with accumulated metadata from previous bundles. Additionally, there is a race condition on reading and writing the accumulated results that must be handled as the order of event receipts is not guaranteed.

One approach to addressing this is to merely queue all data bundle events. This, however, requires knowledge of completion so that process can be triggered, which is not addressed by the use of a queue. The algorithmic complexity of combining incoming notifications impacts the data browser when creating project metadata TSVs and normalized JSON files, as well as the matrix service when generating pre-built project matrices.

#### Data consistency and completeness
Consistency and completeness across a data set is an important attribute of an atlas. An incomplete data set may result in misinterpretation of data or missed discoveries. While a data set can be updated in the future, there is a need to know when the current version of a data set is fully available. This is referred to as *version-completeness*. Currently, the DCP has no way to indicate version-completeness beyond the bundle level. Neither the data browser nor any consumer API and has the information to indicate that a project submission to ingest is complete. This information is only available within Ingest, with no way to communicate the status to other components.  Ensuring consistency between bundles in a group requires modifications to be completed before they are made available. For example, an update to metadata that is duplicated between bundle must all finish before the update is published.

#### Data set quality control
An approach to quality control (QC) is to run a partial or full analysis on a subset of the data before doing a full analysis. This requires defining an analyzable subset of the data to pass on to the processing pipelines. A QC data subset must meet the criteria that allow the QC analysis to run.

## User Stories

As a data contributor, I want to maintain flexibility in the laboratory approaches I can use to perform top-up sequencing, library multiplexing and arrange replicates on flowcells in a way that minimizes overall cost, while being assured that the processing will be correct.

As a data-wrangler, I want to be able to succinctly enter the metadata I receive from the data contributor and have a clear understanding of what I need to do to ensure correct processing.

As a schema developer, I want to ensure we are capturing and requiring, the metadata from a contributor to ensuring the pipelines can run appropriately.

As a member of the pipelines team, I want to ensure that the pipelines process the correct data sets and I can easily access the metadata required to trigger the pipelines.  I do not want to be restricted in designing analysis based on how data was submitted.

As a data consumer, I want to be able to find all data files that need to be processed together so that I can run my data processing pipeline. I do not want to be restricted in designing analysis based on how data was submitted.

As a data consumer, I want to be confident that the HCA DCP has correctly processed all data files that belong together so that I know the matrices I receive have been generated correctly.

As a data consumer, I need to know that a data set is done so that I don't download and use it until it is version-complete.

## Detailed Design
A new concept of *data group* is added to the DCP data model to address these issues. A data group is defined as a set of specific versions of metadata and data that is complete and consistent by a pre-specified set of criteria. A *data group* is not created until all its contents are complete and uploaded to the DSS.  As with any bundle, events are generated when a *data group* is created, updated, or deleted.

A data group has a scope that specifies what level of the data hierarchy is contained in the group.  Scope is the breadth of the type of data represented in the group, not the step in the life-cycle that the group references.  The per-bundle notification model has a scope of a single bundle.  The above use cases will require a *project* scope that indicates submission to a project is complete. There is a discussion for units of data different than *project*, currently referred to as *data sets*.  These could form another data group scope if required.

While the life-cycle step can be inferred by interrogating the bundles referenced by a *data group*, this will most likely very expensive and non-obvious.  Instead, we add the attribute *step* that defines where it the DCP pipeline the *data group* was produced.  The exact values will be defined by the producer components.  Once possible approach is to use the bundle type of the primary data types, for example *hca/analysis-output; hca-pipeline:snap-atac*.

Data groups are implemented as a new bundle type that contains a JSON file
listing the FQIDs (UUIDs with versions) of all bundles in the data group. The
existing DSS subscription mechanism is used for notifications.

A new schema *data_group* will be created that contains the fields:
- *scope* - Symbolic name of the scope, for example, *PROJECT_SCOPE*.
- *step* - Symbolic name of the life-cycle step, for example *hca/primary-data; hca-data-type:sptx*.
- *bundle_fqids* - List of FQIDs of bundles that are in the group.



See also:
- [The Ingest Submission Data Model](https://docs.google.com/document/d/1qlgIROPK4qESy9-Lwvva1ZWJ3S5EdZ0HWJiGGvt2JRM/edit#heading=h.8r5egyz5ly82) for a description of submissions.
- [Bundling in ingest](https://docs.google.com/document/d/1cGfolHMGEe1IKkXSBS75SAdSri9Np_oW6Z4S9-Ax1QY/edit#heading=h.xv0ag3c16x15) for details on how bundles are created.

Initially, only the *PROJECT_SCOPE* scope will be implemented. *PROJECT_SCOPE* will indicate that a single submission of data to a project is version-complete and will be generated by Ingest only after all data bundles are committed to the DSS. The generation of the *PROJECT_SCOPE* data group indicates that data in the submission to the project should be processed by downstream components.  Ingest is responsible for creating *PROJECT_SCOPE* *data groups* after the relevant bundles have been created.  Figure 1 shows a diagram of a project with multiple *data groups*.

![Figure 1](../images/0013-multi-data-collection-data-processing-images/project-scope.png)


## Application to analysis pipelines

While this RFC proposes a general mechanism, it was developed in response to the needs of analysis to co-process data that are in multiple assay bundles.  This section uses co-processing of multiple sequencing run from a given library as an example of the use of data groups to identify data that should be processed together.  A key concept behind this proposal is that upstream submission should not be defining how downstream analysis groups inputs for processing.  The analysis pipelines need to be able to group input based on ways that may not be defined at submission time.  New analysis pipelines should be able to group inputs in arbitrary ways without requiring changes to the way data is packaged.

### Identifying data to co-process
Pipelines that require grouping data from multiple sequencing assays (co-processing) subscribe to *PROJECT_SCOPES* events instead of per-bundle events. Unlike with the assay bundle per pipeline run, this makes the pipeline framework responsible for collecting assay bundles into pipeline runs.  Pipelines that do not require co-processing continue to operate on a per-bundle basis, however pipeline services needs to subscribe to the project data group so they can notify ingest when the input *data group* process is complete for use by downstream consumers, such as Azul.

For full background and a detailed proposal on how these libraries are represented in the metadata see the RFC [Representing sequencing library preparations in the HCA DCP metadata standard](https://github.com/HumanCellAtlas/dcp-community/blob/68f0c42fc9bd55297be8c241663721271c90b694/rfcs/text/0010-rfc-library-preparation.md).

The *data group* submission event does not define bundles to be co-processed, it indicates that data to be co-processed is contained within the *data group* and that that group fulfills version-completeness requirements. Co-processing will be initiated by pipelines based on a well-defined set of rules that is data modality-specific.  The partition of a submission *data group* into co-processing units may require multiple parameters. For instance, with 10X V2 3’ scRNA-seq is grouped by library, however, a submission may contain non-10X data modalities that need to be processed separately.

An analysis pipeline event handler component will be created that is responsible for partitioning a *data group* into co-processing units, dispatching the pipelines, and tracking completion. This is the responsibility of Analysis.  When analysis results are submitted to ingest, this creates a secondary analysis *data group*.

The creation and update of *PROJECT_SCOPES* *data groups*, both primary and secondary, is the responsibility of the Ingest system.

![Figure 2](../images/0013-multi-data-collection-data-processing-images/flow-of-information.png)
Figure 2: Flow of information in the proposed data model

#### Updates to *data groups*
Updates to *PROJECT_SCOPES* must result in updating of the associated *data groups* to trigger reprocessing. When part or the entirety of a project submission is updated the associated data group must be identified and updated with the new bundle versions if bundles have been replaced and/or with additional new bundles if bundles have been added. This is the responsibility of Ingest.  Analysis must handle the update to the *data group* by initiating only the pipelines that have been affected by the update and ensure that the output bundles of this analysis are updates to existing bundles.

*Data group* updates may result from creating new bundles to add additional types of data to a project, or updating bundles to change metadata or append data to data files (e.g. FASTQ top-up).

#### Implementation feasibility

Both Analysis and Ingest confirm that implementation of the data_groups bundle proposed here is feasible in principle. Analysis indicates that significant changes need to be made to the existing infrastructure to handle the new type of notification. The generation of secondary data_groups of type PROJECT_SUBMISSION will be generated by analysis submitting to submission and Ingest will identify the bundles included in the secondary bundle.

The following order of implementation is proposed, accounting for the dependencies between components to ensure uninterupted system functionality:

1) Ingest adds functionality for supporting data_groups and starts using them for singalling submission completion
2) Analysis adds handler for data_groups but continues to use bundle events during a transition period
3) Analysis switches to using data_group events
4) Ingests adds support for secondary data_groups
5) Analyis starts notifying Ingest for the existence of data_groups
6) Downstream components (Matrix Service, Portal) start listening for secondary data_groups

### Unresolved Questions

####  Aspects of the design to be resolved during RFC review
* Is *step* the best terminology for the type of a *data group*

#### Aspects of the design to be resolved during implementation:
* Where does the *data group* JSON schema belong?  It is not part of the experimental data model, so doesn't fit in the current hierarchy.

* How is *step* defined?  By reusing bundle type definitions or another mechanism?

#### Aspects of the design to be resolved in the future
* *Data group* concept relates directly to the UX discussion on *experiment data sets*.  This RFC and that work could be unified with a direct mapping between *experiment* and *data groups*.

## Dependencies

Implementation of this RFC is partly dependent on [RFC: HCA DCP Bundle Types and Definitions](https://github.com/HumanCellAtlas/dcp-community/pull/93), however, introspection of bundles could be used as a workaround.

### Alternatives

- DSS collections were considered as a possible implementation, however, they don't support subscriptions and are not indexed, so they are not a sufficient mechanism for implementing *data groups*.
