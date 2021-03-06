### DCP PR:

[dcp-community/rfc14](https://github.com/HumanCellAtlas/dcp-community/pull/103)


# HCA DCP Data Citation Plan

## Summary

Data Contributors need to be able to cite data sets that they have contributed to the DCP.
Data Consumers need to be able to cite the DCP data that they have used in their research projects.
Providing citable records for the DCP will enable all contributors and consumers to reference the data stored in the DCP that they have used in their scientific publications.

## Author(s)

 [Trevor Heathorn](mailto:theathor@ucsc.edu)

## Shepherd

 [Trevor Heathorn](mailto:theathor@ucsc.edu)

## Motivation

There is currently no clear and agreed upon definition of the requirements for Data Citation in the DCP.
Key issues that this RFC seeks to resolve are:
  - What is the minimum feature set required for a first release (Phase 1)?
  - What are the discrete set of features that make sense for subsequent releases (e.g. Phase 2, Phase 3, etc.)
  - In which phase, if any, is support for a formal digital object identifier ([DOI](https://en.wikipedia.org/wiki/Digital_object_identifier)) required?

### User Stories

1. As a data contributor (e.g. researcher with a pipette), it is essential to have a unique way for my project (consisting of primary and secondary analysis data files and associated metadata files) to be identified so that others can properly cite my work when using my data, and that this citation identifier is available in the DCP Data Portal. Note: This will discourage but not prevent others from using my data without attribution.
2. As a data consumer (e.g. researcher with a keyboard), I want to be able to view and share a unique citation identifier so that a reader of my manuscript can obtain the data needed to reproduce my results. Anyone can use the citation identifier to view and download all the original cited data and metadata files for a project from the DCP.
3. As a data consumer, I need a simple way to reference a project in the DCP so that I can fulfill the requirements of a Creative Commons attribution license (CC-BY).
4. As a data contributor or consumer, I need a way to use the citation identifier to access the output produced by the DCP Matrix Service for the data being cited.
5. As a member of the Data Operations team I want to ensure that a versioned citation identifier is created for each discrete version of a project in the DCP.
6. As an authorized data consumer (a user who has been granted Data Portal login credentials) I want to be able to update my research project (consisting of an arbitrary selection of HCA data that I have selected) and have a versioned citation identifier that can be used to reference discrete point-in-time versions of my research project.

## Detailed Design

It is proposed to split the initial implementation into three phases:

### Phase 1 - Stable non-versioned citable project URLs
This is designed to satisfy the minimal set of requirements for User Stories #1 through #4 by providing only "per-project" citations.
The Data Browser project details page will add a "To cite this project please copy this link" item.
This "stable non-versioned project URL" will link back to the production site project page using the project's UUID (e.g. https://data.humancellatlas.org/explore/projects/cc95ff89-2e68-4a08-a234-480eca21ce79).
The URL refers to the “live" view of the project and is therefore subject to additions and updates (e.g. corrections) of data and metadata. However, these are expected to be infrequent and will rarely affect primary data.
A stable URL applies only to a project as a whole; there is no facility to provide separate citations for individual bundles or files within a project.
If an existing project is deleted and re-ingested then the cited project UUID would become invalid. If such re-ingestion occurs then a means will be provided to redirect the original "stable project URL" to the new version of the project (e.g. by providing a landing page which states something similar to "This project has been updated with corrected data and/or analyses. For the current version of this project click *here*", where *here* is a link to the new project page).
Note: Scientists are *already* citing such project based URLs in publications.
A formal DOI is *not* required for Phase 1.

### Phase 2 - Versioned Data Citations
This is designed to satisfy the requirements for User Story #5.
This allows the Data Operations team to ensure that each discrete version of a project is citable.
The Data Browser must provide users with access to each immutable version of data in addition to the “live/latest” view.
The Data Browser provides a means of downloading the cited data associated with a specific version of a project.
The per-project output files from the Matrix Service *must* be stored as part of the immutable data set since these files, along with the primary data and secondary analysis files, are all part of a versioned project.
Note: The Data Store "collections" API provides one suggested means for recording specific versions of data.
A formal DOI is required for this phase as is provides a standardized method for specifying specific versions of data objects.

### Phase 3 - Enabling citation of user defined collections of data
This is designed to satisfy the requirements for User Story #6.
This phase allows a data contributor or data consumer to cite an arbitrary set of data which may span multiple projects.
The user must be able to create a discrete collection of their selected data with the ability to update that collection (i.e. create a new version of the collection).
The Data Browser must provide users access to each version of their collections.
A data consumer must be able to share a citable DOI link to any of the collection versions that they have created.

### Implementation notes for DOI support
A DOI provides a link of the form https://doi.org/xxxx which resolves via the hosting [Registration Agency](https://www.doi.org/registration_agencies.html) or [open-access repository](https://en.wikipedia.org/wiki/Open-access_repository).
A DOI is the most commonly accepted means for citing documents and data in scientific publications.

It is an Unresolved Question as to whether a DOI may resolve to an external website (e.g. BioStudies, Zenodo, etc). Such an external web page could then provide a URL to the project in the Data Browser as well as the ability to store point-in-time copies (i.e. versions) of relevant files such as the download manifest, metadata tsv, matrix output file, etc.
The stored manifest file could then be used in the HCA CLI to download the exact versions of the data and metadata for that version of the project. It is a requirement that the DCP never deletes older versions of cited data files, except for the special case of retraction of unconsented data.

#### Possible DOI Registration Agencies and open-access repositories that provide the ability to assign DOIs:
  - [BioStudies](https://www.ebi.ac.uk/biostudies/)
  - [Zenodo](https://en.wikipedia.org/wiki/Zenodo)
  - [Figshare](https://en.wikipedia.org/wiki/Figshare)
  - [Crossref](https://en.wikipedia.org/wiki/Crossref)
  - HCA DCP assigns its own DOIs by becoming a member of the International DOI Foundation (IDF)

#### DOI Versioning
Most open access repositories that provide DOI minting services appear to provide support for versioning (i.e. multiple versions of a single citation available e.g. via a drop-down selection), the ability to store accompanying files in the repository (this would be useful for storing versions of manifest and metadata tsv files, etc.), and the ability to store URL references (useful for linking back to a page in the Data Browser).
Examining how both Figshare and Zenodo implement DOIs, as the group minting DOIs controls everything after the prefix, it is possible to have a main DOI which points to the most recent version and then versioned URLs which point to specific versions.

#### Data Browser support for DOIs
The Data Browser could contain a reference to the base DOI for each project. Clicking on the DOI link would then redirect the user to the (external) DOI repository where the user could see all the versions of that project and download the associated manifest and metadata.tsv files for a specific version. 

#### DOI Creation/Update Process
Embedding a DOI in the metadata (e.g. biostudies_doi) during ingest would provide per-project citability.
Provided the DOI repository supports versioning the metadata DOI field could be updated automatically by Ingest to a new version whenever a project update is processed.
Using a DOI that links to an external repository that can store a manifest for each version of a project may be the simplest way to provide access to versions of the data for a project.

The creation/update process would perform the following steps:
  - Ingest creates a new DOI (new project) or a new version of an existing DOI (project has been updated) when a project data submission has been determined by the Data Operations team as ready for public consumption
  - Upload the corresponding metadata tsv and manifest file for this version of the project to the DOI repository
  - Upload the matrix output file(s) for this version of the project to the DOI repository
  - Update a project description to the DOI repository (e.g. what’s in this version?)
  - Create a link in the DOI repository back to the project details page in the Data Browser

### Acceptance Criteria

#### Phase 1
  - A citation link is provided in the Data Browser for each project
  - The citation link remains valid for the lifetime of the DCP
  - The citation link is a URL which resolves to the latest version of the project details page for the specified project
#### Phase 2 (in addition to Phase 1 criteria)
  - The Data Operations team is able to create an immutable citation reference for each project
  - The citation reference consists of a versioned DOI
  - The Data Operations team can update the version of a project's DOI, for example when updates have been made to that project's data since the previous version of the DOI was assigned
  - The Data Browser provides a means of downloading the data associated with a specific version of a project
#### Phase 3 (in addition to Phase 1 and 2 criteria)
  - An authorized user may create a citation for an arbitrary set of versioned data selected via the Data Browser
  - An authorized user may update a previously versioned set of data to a new version and update the version of the corresponding citation
   - An authorized user may share a citation which they have created with others users who can then access the specific version of the data referenced by the citation

### Unresolved Questions

Must a data citation that provides a DOI provide an *immutable* view of all the data and metadata associated with a project? Or is acceptable for any of the following to undergo additions, updates or deletions over time?
  - a project's primary data
  - a project's secondary analysis outputs
  - a project's expression matrix outputs, generated by the Matrix Service
  - a project's metadata
  
For Phase 2 & 3 does the Data Browser need to provide a view of the cited data on which further faceted searches can be performed?
Currently the Data Browser provides only a view of the latest data and metadata. It may be desirable to add another "facet" such as "Version" which would enable the user to view a particular point-in-time version of the DCP data and then perform further faceted searches within that view.

Can a DOI for the DCP resolve to an external website? The author(s) will work with the UX team to come up with pros and cons of using an external authority vs setting up the DCP as a DOI assigning entity. If the results of the UX and Browser team research does not provide sufficient clarity we will present our preferred option and evidence to the Oversight Committee and ask for their feedback.
