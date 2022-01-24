- Start Date: 2021-10-05
- RFC PR: [#54](https://github.com/inveniosoftware/rfcs/pull/54)
- Authors: Alex Ioannidis, Pablo Panero
- State: DRAFT

# (Author) names vocabulary

## Summary

Implementation of an auto-complete-style input field for Creators and Contributors in the upload form, powered by a new **Names** vocabulary.

## Motivation

Manually filling in author lists for a new upload is a tedious and error-prone process, which can causes issues regarding:

- Author disambiguation, e.g. ["Which John Smith am I referring to?"](https://orcid.org/orcid-search/search?searchQuery=John%20Smith) 
- Poor metadata quality, e.g. missing or invalid ORCiD, affiliations, etc.

Simple user input format-validation (i.e. "is this a structurally valid ORCiD") helps addressing input errors, but doesn't make the input process easier/quicker or leads to better metadata quality.

For that reason, allowing the user to select from an auto-complete-like search field Creators and Contributors allows for:

- Quickly finding the name/author
- Automatically filling in metadata that's already known for the author
- Making sure that the input is by default valid (since it comes from a pre-existing database)

## Detailed design

### Datamodel

Since the aim of this feature is to provide easy means to fill-in the upload form fields for Creators and Contributors, the datamodel of the `Names` vocabulary has the same fields as they accept (for `type: personal`):
    
```python
{
  # PIDs with `names` type
  "id": "vsbay-2rk51",
    
  # Basic metadata
  "name": "Smith, John",
  "given_name": "John",
  "family_name": "Smith",
  "identifiers": [
    {"scheme": "orcid", "identifier": "0000-0001-8135-1234"},
    {"scheme": "gnd", "identifier": "7749153-1"},
  ],
  "affiliations": [
    {"name": "CERN"},
    {"name": "TU Wien"},
  ],
}
```

#### PID Type

**_Using `recidv2`, why?_**
    
The idea is to provide a transparent way of managing name records and protect them against changes from the source e.g. change on ORCiD records.
    
**Drawbacks**
    
The corresponding identifiers (ORCiD, GND, etc.) are only stored as metadata. This has the implication that before performing any action (update, create, etc.) it will be needed to perform a `resolve` query, since the identifiers will not cause a `PIDAlreadyExists` error. This also means, there is no implicit deduplication (i.e. two _names_ can have the same orcid).

One potential solution would be to create a `PIDs` system field. See more in the _unresolved problems_ section.

### Service layer
    
It get the default behaviour given by the `invenio-records-resources` Record factory. Meaning: create, delete, update, read, read_many, read_all, search, scan.
    
In addition, it implements a `resolve` method to be able to retrieve a name based on one of its identifiers (e.g. orcid). This method performes a specific query search on ES, because it is not possible to do a DB query since the identifiers are metadata only.
    
### REST API

`/api/names`
`/api/names/<id_types>/<id_value>` e.g. `/api/names/orcid/0000-0001-8135-3489`

### UI/UX

Mockups at: https://balsamiq.cloud/ssg4nq2/p68p1b0/r0520

## Example

[ZenodoRDM Upload form](https://zenodo-rdm.web.cern.ch/uploads/new)

![](https://codimd.web.cern.ch/uploads/upload_b4c8a136770f755c35bb4f2b55dbc4bd.png)

![](https://codimd.web.cern.ch/uploads/upload_8782a03b93fed9fa8eea4e17fee232a7.png)

    
## How we teach this

N/A: It is a feature.
    
## Drawbacks

- Vocabulary entries are not linked through a `relation`. Users might expect that changing a entry's name would change the record metadata too.
- Users cannot notify that an entry is stale/old/wrong.
    
## Alternatives

- No auto-completion
- Auto-completion with a relation at record level
- Auto-complete from an external API (e.g. query ORCiD.org public API)

## Unresolved questions

- Should we have an "auto fetch"? i.e. a user searches for a non-existing ORCiD, should we trigger a datastream task to import it?
- Implementing a `PIDs` system field, in a similar fashion than the exising `PID` one, would solve the problems coming from duplicates. In addition, it would help performance. For example, eliminating the duplicate permission check, which is a heavy operation (i.e. calling `resolve` requires a permission check, which will be also done on `create` after).
    
## Resources/Timeline

- Based on the current InvenioRDM roadmap, ORCiD support is planned for the v8.0 release which is scheduled for 2021-12-02
- From the Zenodo team, there are 10-14PW allocated for this feature, which is also part of an [OpenAIRE-NEXUS](https://cordis.europa.eu/project/id/101017452) deliverable for January 2022.