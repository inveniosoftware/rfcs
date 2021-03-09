- Start Date: 2020-03-03
- RFC PR: [#45](https://github.com/inveniosoftware/rfcs/pull/45)
- Authors: Lars Holm Nielsen
- State: DRAFT

# Versioning support in InvenioRDM

## Summary

The RFC describes how Zenodo-style versioning support is going to be integrated into InvenioRDM. A certain knowledge of Zenodo's versioning support is required to understand this RFC.

## Motivation

Versioning is a core feature in order to support users in evolving their records, while keeping previous versions online for record-keeping (e.g. citation to previous versions).

Please also see the Zenodo [FAQ](https://help.zenodo.org/#versioning) on versioning.

## Prior work

Zenodo already has record versioning support implemented, that allow users to create new version of records. The best way to try out the existing versioning support is to test it out on https://sandbox.zenodo.org. The existing code you code find by grepping for  ``pidrelations`` in Zenodo source code, and read the [FAQ](https://help.zenodo.org/#versioning)

From an end-users perspective Zenodo currently implements versioning this way:

- Landing page:
    - Newer version warning
    - List new/previous versions on the landing page (with some associated metadata, like e.g. a publication date).
    - Manage: "New version" button to create a new version.
- Deposit form:
    - Discard a new version (similar to deleting an unpublished draft)
    - Manage: "New version" button new the files section.
- Search:
    - Facet to switch between showing the latest versions only (the default) or all versions.
    - Sort records by their version index.
    - Filter to show only records for a given record.
    - Result items links to a page to see all versions

In addition the backend code in Zenodo deals with:

- Community inclusion requests over multiple versions (accepting a record in a community accepts all versions).
- DataCite DOI registration for concept and specific versions (specifically the metadata of the latest version is used as basis for the concept metadata).
- Files management (to avoid duplicating files).

What's not currently in Zenodo:

- Landing page for the concept version.

## Design overview

### Using prior work

The core of the versioning support was implemented in Invenio-PIDRelations. Invenio-PIDRelations was however designed for the old way of storing all records/deposits in the same table that we have gone away from. Additionally a lot of code in Invenio-PIDRelations deals with the fact that records and deposits had separate PIDs (recids and depids). Additionally, a lot of code deals with registration of new relation types and similar. The actual code that deals with the versioning support is in the end quite small. For these reasons, we think that we should not use Invenio-PIDRelations, as it will make the versioning support overly complex.

In addition, we'd like to separate by design the versioning support from the community integration as well as the external PID registration (yet to be done). This is to ensure that each feature works as independently as possible.

### Versions and revisions disambiguation

The versions and revisions terms are used several places in Invenio and thus it's useful to first disambiguate what is meant with each term.

- A record **version** is an individual record with it's own metadata, files, PIDs etc.
- A record **revision** is a change made to the JSON document of a record (version).

A record revision is created every time a record is published. E.g. when an user updates the metadata of a record, a publish the change, then a record revision is created. This means that we use revisions to track changes made to the JSON document of a record over time. Currently, these changes are not exposed publicly, but used solely for tracking changes to the JSON document for record keeping.

A new record version is created when a user decides so - this is usually when the user wishes to update the files. The record version represents the entire record.

Last, you may see that the record metadata table has a ``version_id``. This is confusingly actually the revision id, and the Record API class also exposes this as the ``revision_id``. The value is used for optimistic concurrency control.

### High-level principles

A record version is by itself a record with its own associated objects. This entails:

- Persistent identifiers:
    - Each version has its own persistent identifier.
    - Each version has its own external persistent identifiers.
- Files:
    - Each version has its own set of files (and thus its own file bucket) - i.e. files can be managed independently for each version.
- Usage statistics:
    - We track views/downloads for both individual versions and aggregated numbers for the concept.

At a technical level, we want the version relationship to be a strong relationship between records, instead of between internal persistent identifiers.

A record can have potentially have 1000s of versions (e.g. in a snapshotting use case).

### Access control

The main thorny issues for the versioning support is access control.

**Visibility and embargo date per version**

We want to be able to control record and files restrictions (public vs restricted), and the embargo date at a record version level. E.g. one record version can be restricted, and another version can be public (this feature has e.g. been used a lot in Zenodo).

**Ownership and permissions per concept**

Ownership and permissions must be managed at the concept record level (i.e. all versions). This is contrary to how the visibility and embargo date are defined on single versions instead.

Having ownership/permissions managed at a single version level will lead to strange issues, like an owner of a record version not being able to manage prior versions (this issue has e.g. been seen in the GitHub integration on Zenodo). Also, it becomes very complex to assign permissions to other users. For instance, an owner that want to remove access for a user has to go to all versions (potentially several hundreds).

The community integration works in a similar way to owners, in that all versions or none belongs to a community.

**Ownership and permissions takes effect immediately**

A key design decision is if ownership and permissions is managed differently on records vs drafts. Drafts are being edited, and then published to make the changes available.

Thus, the question is when is ownership and permissions taking effect. For instance, an owner gives view permission on an already published record to a user via the deposit form. What happens?

- Immediate effect:  If the changes take effect immediately, the user can now view the record.
- On publish:  If the changes take effect on publish, then the user cannot view the record until the new record is published.

Having permissions take effect immediately opens other use cases of directly sharing a record from the landing page instead of going through the deposit form. Also the model seems simpler to understand for end-users.

### Exclusion: Version relationships across repositories

You could imagine modelling version relationships across repositories. For instance a dataset is first uploaded to repository X, then later a new version is published in repository Y.

We will not complicate the versioning support with this style of relationships. Relationships across repositories should be modelled in metadata via the related identifiers field. In addition, another workaround is to transfer the full dataset from repository X into repository Y.

## Backend Design

The backend consist of the data access, service and presentation layers.

### Data access layer

#### Version relationships

We propose that the information previously held by Invenio-PIDRelations is moved directly to the record database table. The current attributes of the ``RDMRecordMetadata`` model includes:

- ``id `` - UUID of the record
- ``json`` - Record JSON document
- ``version_id`` - Version counter used for optimistic concurrency control.
- ``created `` - Creation timestamp.
- ``updated`` - Modification timestamp.
- ``bucket_id`` - Foreign key (UUID) to a Files-REST bucket.

We propose to add the following two attributes:

- ``parent_id`` - Foreign key (UUID) to a Concept Record (a new entity).
- ``parent_order`` - Integer (4 bytes) defining the order of siblings.
- ``parent_latest`` - Boolean to indicate if the record is the latest version.

The name ``parent`` could alternatively be ``concept``, but we would like to avoid naming that conflicts with the ``version_id``. Technically ``version_id`` is the **revision identifier** however, due to history reasons and API this is not an easy rename to perform.

In addition a database index should be defined to allow for easily querying all records related to a concept record in the correct order (thus likely a combined index on ``parent_id`` and ``parent_order``):

```SQL
SELECT * FROM rdm_records_metadata WHERE parent_id=... ORDER BY parent_order
```

#### Concept Record

We also propose to add a new entity, a ``ConceptRecord``, which represents the concept record and are able to hold properties for all records such as ownership and permissions.

The model (table name ``rdm_records_concept``) would be like a usual record with the following database columns:

- ``id `` - UUID of the concept record
- ``json`` - Record JSON document
- ``version_id`` - Version counter used for optimistic concurrency control.
- ``created `` - Creation timestamp.
- ``updated`` - Modification timestamp.

The concept record would hold JSON documents similar to below:

```json
{
    "id": "1234-abcde",
    "pid": {
        ...
    },
    "access": {
        "owned_by": [{"user": 1}],
        "grants: [
            ...
        ]
    },
}
```

Essentially the concept persistent identifier, the access ownership and access grants would be moved from the existing record to the new concept record.

**Why a new entity?**

The new model allows keeping the ownership and permissions separate from the single version records. Alternatively, we would have to duplicate (and update) access information in multiple specific version records.

Also, having just a single concept record for both records and drafts, means that access information can be updated not only from the deposit form, but also e.g. share button on the landing page record.

In addition, it opens future use cases where we may want to associate more data with the concept record. On possible upcoming use case involves storing shared links with the record.

We will also know exactly when we have to reindex siblings and when it's not needed.

**Why not in the JSON?**

An alternative is to store the ``parent_id`` and ``parent_order`` in the record instead of at the database table, however would like to have the database-level constraints enforced.

**SQLAlchemy-Continuum revisions**

The Concept Record should not have the SQLAlchemy-Continuum revision system enabled, as any change to the access would create a new revision. While, we may want to log changes to the access control, we prefer to have a specific audit logging module for this purpose.

#### Record API

The record API should be extended with

**Accessing the concept**

```python
{
    "concept": {
        "id": "<uuid>",
        "access": {
            "owned_by
        }
    },
}
```

#### Indexing of concept records

We will not index concept records.

#### Use Case: GitHub Integration (system process)


#### Reindexing / denormalization

- Reindex siblings.


### Service layer

#### Create a new record

```python
service.create(id_, identity, data, links_config=...)
```

#### Read a record

```python
service.read(id_, identity, data, links_config=...)
service.read_draft(id_, identity, data, links_config=...)
```

#### Edit a record

```python
service.edit(id_, identity, data, links_config=...)
```

#### Create a new version

```python
service.new_version(id_, identity, data, links_config=...)
```

#### Updating a draft

The update of a draft is unchanged, except that you can no longer change the ownership and permissions via the draft (which anyway was not possible in the February release).

**Visibility and embargo information**

Specifically, you will still update the record/files restrictions and embargo date/reason via the is method. The only difference is that it requires manage permissions to change this information, where as the metadata can be changed with edit permission.

#### Updating ownership and permissions

We propose adding a new method specifically update the permissions:

```python
service.update_permissions(id_, identity, data, links_config=...)
service.update_owners(id_, identity, data, links_config=...)
```

- Resolve the id
    - Should transparently handle the concept pid / pid issue (explicit - always use concept - not logical for APIs)
- Check revision id
- Check permissions
- Load the data
- Run components
- Commit concept record
- Index
    - Reindex all records and drafts (bulk index)

#### Search versions

```python
service.search_versions()
```

#### Reading a record/draft (dereferencing)

(not needed? managed through record service)

#### The service schema

- A new schema for dumping permissions.

### Presentation layer

#### REST API

The existing REST API, already define endpoints for the main operations discussed in the service:

- ``GET /records`` - Search records.
- ``GET /user/records`` - Search drafts.
- ``POST /records`` - Create a new draft.
- ``POST /records/:id/draft`` - Edit a record.
- ``PUT /records/:id/draft`` - Update draft.
- ``POST /records/:id/actions/publish`` - Publish a record.
- ``GET /record/:id/versions`` - Search versions for a record.
- ``POST /record/:id/versions`` - Create a new version.

We will add a new endpoint for updating the permissions:

- ``PUT /records/:id/access`` - Update permissions.

We use ``PUT`` instead of e.g. ``PATCH /records/:id`` because we want to make use of the concept records version counter in the ETag for optimistic concurrency control, and we want to model the endpoint as a new REST resource. Some notes:

- Identifier: We're using the ``:id`` of single version records instead of the concept record. This is to make it easier for API developers to deal only with a single identifier for a version, and to not confusing by using a different id.
- Resource: We model an "access resource" instead of a "concept record resource" to make it easy to use and understand for API developers, and because different parts of the concept record may have different permissions as well as service methods.

**Record/Draft Serialization**

Both record and drafts serializations should include a link to the access resource if the user has manage permission:

```json
{
    "id": "...",
    "links": {
        "access": "https://localhost:5000/api/records/:id/access",
        "latest": "https://localhost:5000/api/records/:id/versions/latest/",
        "latest_html": "https://localhost:5000/records/:id/latest/",
    }
}
```

**Access Serialization**

The access serialization

```json
{
    "id": "...",
    "access": {
        "owned_by": []
        "grants": []
    }
    "links": {
        "access": "https://localhost:5000/api/records/:id/access"
    }
}
```

## Frontend design

#### Redirect endpoint

In order make getting to the latest version easier and not make expensive queries when rendering the landing page, we suggest adding a UI endpoint, ``/records/:id/latest/`` which redirects to the latest version of a record.

```
HTTP/1.1 302 Found

Location: /records/:id
```

#### Landing page

The record landing page should add the following elements:

- Newer version alert
- Previous versions
- Manage section:
    - A "New version" button.

**Newer version alert**

A record that has a newer version should clear display a warning message (same as on Zenodo), with a link to the new version.

The condition for the new version is essentially: ``record.parent_latest is False``.

The latest version link should use the link defined by the REST API.

**Previous versions**

The list of previous versions can be quite expensive to compute and requires information from each of the records (e.g. title/version string/DOI). We suggest adding a small React-Searckit application that calls the ``/api/records/:id/versions`` endpoint.

The request should include the following parameters:

- ``size=5`` - To show the latest 5 versions.
- ``sort=version`` - To sort by the version order.

The result item rendering should:

- Clear mark the currently displayed version. If the currently displayed version is not in the top 5 results, then it should be added to the end of the list. See Zenodo behaviour for this.
- Link to a page where all versions can be searched ("View all X versions")

**Manage section**

The landing page should in the manage section display a "New version" button if the use has manage permission.

#### Search/List uploads

The two search pages needs to add the following:

**Version facet**

A flip-switch which essentially

- Version facet (flip switch): ``?versions=all|latest``
- Sort by version number
- Filter by parent version
- Result item: display link to concept landing page

## Plan

- Make service.new_version() work fast
    - Perhaps without files?
    - Forget about reindexing etc
- Landing page: new version button
- Start list uploads/search pages filters.
-




- New draft:
    - ConceptPID and PID is created
    - One bucket for both drafts and records (records/draft files).
- Publish:
    - both pids are registered.

- How do you navigate from deposit form to other versions

- List uploads:
    - Should show both records being   edited


- Difficult cases:
    - Create a draft of latest record, and create new version draft.
    - How are we dealing with files?



### Design principles

- Reindexing should be done in the background - to avoid long publishing time - any impact?
- Internal/external PIDs should not be mixed up
- Community inclusion requests?

### Data model


- Draft:
    - status: new? is_published?


### Queries




### Landing page for concept

- Show what? metadata from latest version?
- List search over all versions

### Tasks
- REST API Docs
- Files:
    - ???
- Deposit form:
    - [ ] Discard button to discard the a new version
- Search page + list uploads
    - [ ] Flip switch on search result page
    - [ ] Sort by version number
    - [ ] Filter by versions
    - [ ] Result template updates (display link to other versions)
        - should this be a concept landing page? showing all versions?
        - concept record could have metadata and pids -> would it make serialization easier? pid lookiup etc?
- Landing page:
    - [ ] Right hand column display of versions.
    - [ ] Version alerts on landing page and link to new version
    - [ ] Button + view for creation of new verison
    - [ ] Check UI flow for new version creation.
- REST API support
    - [ ] ``/api/records?all_versions`` (include/exclude previous versions)
    - [ ] ``/api/records?sort=versions`` (sort in version order)
    - [ ] ``/api/records?parent=`` (filter to versions of a record)
    - [ ] New version creation endpoint ``POST /records/:id/versions``.
- Backend
    - [ ] New system field: relations (responsible for seralizing relations into the record).
    - [ ] ESDumper: correct dumping of relations in index
    - [ ] PID relations: Redirect of latest record
    - [ ] Custom check to see if publishing is allowed (e.g. are the files the same)
- No community integration (but shouldn't be necessary)

- Excluded:
    - Checking if files are the same
    - Registering external PIDs


## Example

> Show a concrete example of what the RFC implies. This will make the consequences of adopting the RFC much clearer.

> As with other sections, use it if it makes sense for your RFC.

## How we teach this

> What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing Invenio patterns, or as a wholly new one?

> Would the acceptance of this proposal mean the Invenio documentation must be re-organized or altered? Does it change how Invenio is taught to new users at any level?

> How should this feature be introduced and taught to existing Invenio users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Invenio, on the integration of this feature with other existing and planned features, on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still TBD? Use it as a todo list for the RFC.

## Resources/Timeline

> Which resources do you have available to implement this RFC and what is the overall timeline?
