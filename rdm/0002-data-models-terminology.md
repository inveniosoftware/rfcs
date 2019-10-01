- Start Date: 2019-09-30
- RFC PR: [#2](https://github.com/inveniosoftware/rfcs/pull/2)
- Authors: Lars Holm Nielsen

# Data models terminology

## Summary

Definition of terms used when speaking about records and data models.

## Motivation

The purpose of this RFC is to disambiguate terms like records, record type, schema and data model that are often used interchangeably in the context of Invenio.

## Detailed design

The central part of InvenioRDM is the primary data model for storing a bibliographic record type describing the digital resources (datasets, software, articles, ...). In addition to the primary data model, InvenioRDM can be extended with one or more secondary data models for storing authority record types (grants, licenses, ...).

You may often see the words like records, record type, schema and data model used interchangeably in the context of Invenio, therefore it’s useful to distinguish what is meant with each term:

- **Record:** a JSON document describing an object.
- **Record type:** a category of JSON documents describing a specific type of objects (examples: grants, licenses, authors, research outputs).
- **Record schema:** a JSONSchema describing the JSON structure of a record belonging to a specific record type. A record type may have one or more record schemas (examples: authors schema v1.0, authors schema v1.1, ...).
- **Deposit record:** a temporary record (e.g. copy of an existing record) used for supporting long running edit sessions that allow users to save their partial changes to a record. A deposit record belongs to the same record type as the source record.
- **Data model:** a full implementation in Invenio of a specific record type which includes e.g.:
    - Record schemas describing the internal JSON structure of the records and deposit records.
    - Search mappings describing how to index records in the search engine.
    - Serializers/loaders used for transforming the internal record to external representations (such as Dublin Core XML, JSON-LD, ...) and vice-versa.
    - Persistent identifier management for records.
    - Exposure of the records in the REST API and on landing pages.
    - Deposit support and access control.
- **Record type category:** a division of record types having shared characteristics (examples: bibliographic record types, authority record types).
- **Data model category:** a division of data models having shared characteristics (examples: bibliographic data models, authority data models).
- **Metadata standard/schema:** a standardized schema for metadata (examples: DataCite, Dublin Core, MARC21).

## Drawbacks

Librarians are not using the term **record** as defined as above. Normally a librarian would use the term record to mean a bibliographic record. The term record might better be named a document, however the term is already used so many places through out the Invenio codebase that it will be hard to change.
