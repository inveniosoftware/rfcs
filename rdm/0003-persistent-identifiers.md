- Start Date: 2019-09-30
- RFC PR: [#3](https://github.com/inveniosoftware/rfcs/pull/3)
- Authors: Lars Holm Nielsen

# Persistent identifiers

## Summary

Overall design for management of persistent identifiers in Invenio RDM.

## Motivation

- As a user, I want to have a DOI for my upload, so that I I can cite my research.
- As a admin, I want to manage multiple DOI prefixes, so that I I don’t need a repository installation per prefix.
- As a admin, I want to assign both DOIs and Handles for uploads to my repository, so that I I can fulfill business requirements.
- As a admin, I want to have a Handle for one upload and a DOI for another upload, so that I I don’t need a repository installation per PID scheme.
- As a user, I want to share a dataset that already have a DOI, so that I I can collect all my research one place.
- As a admin, I want to have tombstone pages, so that I I can remove content.
- As a admin, I want to take over management of a dataset with a DOI from another repository, so that I records can be migrated between repositories..
- As a admin, I want to customize which internal PID scheme is used, so that I I can continue my existing identifier scheme..
- As a admin, I want to ensure all records have a DOI, so that I all records have an DOI.
- As a user, I want to reserve a DOI prior to uploading my report, so that I I can include the DOI in the report.

## Detailed design

### Internal persistent identifier

#### One internal PID per record
All records must have an internal persistent identifier that is unique for the given data model. Assigning an internal persistent identifier ensures that all records have an identifier from the same identifier scheme. This is useful e.g. for identifier resolution in the REST API as well as for managing relationships internally.

InvenioRDM should further allow administrators to easily configure which internal identifier scheme should be used. The only two identifier schemes provided by default are:

- Record identifier (randomly generated 8 character string)
- Recid (Invenio v1 style auto-incrementing integers)

#### Internal PID: Record identifier (random number encoded as 8 characters)
The internal persistent identifier is named the record identifier. The *record identifier* is a randomly generated number encoded using base32. The result is an 8 character string where the last character is a checksum. Following is an example of a record identifier:

```
55e5-t5c0
```

The size of the record identifier namespace is 34 billion and ensures that the identifier can be conveniently and accurately transmitted by both humans and machines.

#### Auto-incrementing integers

InvenioRDM also supports Invenio v1 style recid’s (auto-incrementing integers). They are however problematic because creating new recids require knowledge of the global state (i.e. what is the next number in the sequence). When dealing with concurrency, this can quickly become a bottleneck, whereas randomly generated numbers locally have low chance of conflicting.

### External persistent identifiers

#### Multiple external PIDs per record

A record can have one or more external persistent identifiers (e.g. DOI, Handle, …), however a record can only have one external identifier per identifier scheme. For instance, a record can be identified by one DOI and one Handle, but cannot be identified by two DOIs. In case two DOIs exists for the same record, the problem can be resolved in one of the following manners:

- Create two records, each with one DOI (will allow the DOIs to be managed).
- Include one of the two DOIs in the record metadata as an alternative identifier for the record (means that the DOI cannot be managed by the system).

#### Multiple prefixes/credentials

Many globally unique identifier schemes support a notion of prefixes, and registering new identifiers in different prefixes may require different credentials. It is important that InvenioRDM is capable of handling DOIs (and other PID schemes) that have been registered with different providers. Following are examples of this situation:

- One record having a DOI from DataCite, another record having a DOI from CrossRef.
- Two records having DOIs from DataCite, but each DOI is in a different prefix requiring different credentials.

#### Supported identifier schemes: DOI

InvenioRDM will initially come with built-in support for DOIs as external identifiers. Further identifier schemes may be provided later. The architecture however should from the beginning allow multiple PID schemes.

#### Managed vs unmanaged PIDs

The external persistent identifiers can either be *managed* or *unmanaged*.

- **Managed PIDs** means that it is the repository’s job to register, update and maintain the PID in an external registry.
- **Unmanaged PIDs** means that another repository takes care of registering, updating and maintaining the PID in an external registry.

This allows e.g. dataset with an existing DOI to be shared in the repository, with the repository having take control and manage the DOI.

InvenioRDM must know for each external PID if it’s a managed or unmanaged DOI, since a single DOI from one prefix may be migrated to another repository (i.e. you may or may not manage all DOIs in a prefix).

#### Assignment of identifiers
A key issue in the management of persistent identifiers is when to assign which external identifier(s) for a given record.

- Internal PIDs are reserved as soon as a deposit record is created.
- External PIDs are only registered when the deposit record is published.

A record must be able to explicitly defined which external PIDs should be registered with a default behaviour in case nothing is specified.

#### Tombstones

All persistent identifiers must support having a tombstone pages.

### References

- [1] Fenner (2016). Cool DOIs. https://doi.org/10.5438/55e5-t5c0

## How we teach this


## Drawbacks


## Alternatives


## Unresolved questions
