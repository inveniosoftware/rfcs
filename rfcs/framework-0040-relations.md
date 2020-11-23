- Start Date: 2020-11-16
- RFC PR: [#40](https://github.com/inveniosoftware/rfcs/pull/40)
- Authors: Lars Holm Nielsen, Nicola Tarocco, Alex Ioannidis
- State: DRAFT

# Record relations

## Summary

TODO

## Motivation

Record relations is conceptually similar to foreign key relationships in relational databases. Record relations are used to model relationships between different entities such as for instance referencing a grant record from a bibliographic record.

In terms of Invenio the use cases are many. For instance a bibliographic record could be related to:

- Resource type
- Subjects/keywords
- Languages
- Authors
- Organistaions
- Geonames
- ...

## Previous work

Invenio v1 had the concept of authority records. Authority records would be able to store all types of above examples in the same model.

Early on in Invenio v2 and v3, there was discussions on an [Invenio-Authorities](https://github.com/inveniosoftware-attic/invenio-authorities/issues/1) module.

Invenio v3 today solves part of the record relations via JSONRef support. For instance, a record can include a reference:

```json
{
    "grant": {"$ref": "https://localhost/api/grants/..."},
}
```

During e.g. indexing, the ``{"$ref": "..."}`` part is replaced with the related record. See also the [training exercise](https://github.com/inveniosoftware/training/tree/v3.1/11-linking-records).

The issues with this approach are:

- Technically diffcult to understand and debug because of the resolving magic happening behind the scenes. A special library, catches requests to special URLs, and translates them into database queries. This is non-intuitive and very difficult to trace when it's not working as expected.
- Lack of fine grained control. It is not easily possible to control which fields from the referenced record is included. For instance, a biblographic record relates to a grant record that relates to a funder record. All three records will be dereferenced, causing the index JSON to become very big.
- Security/performance - because JSONRef is based on making HTTP requests, there's a risk that badly formed URLs, will cause HTTP requests to be made. At best this will only cause performance issues, at worst it opens a backdoor for injecting content in the JSON document. While this could also be considered a feature, ability to relate to externally hosted records, we think the performance/security issues makes this technically impossible.


## Design

### Challenges

The primary challenges of record relationships includes:

- **Syntax**: How should a relation be made between two records?
- **Integrity**: How is integrity checked and validated?
- **Dereferencing**: When and how is a record dereferenced?
- **Denormalization**: When a dereferenced record is updated, how is the related index updated?

Each of above challenges is addressed in the following design.

#### Syntax

We propose the following syntax:

```
{
    "<field>": {"id": "<persistent id>"},
    "<field>": [{"id": "<persistent id>"}, ...]
}
```

A relation is an object with a single key ``id`` with a string value for a persistent identifier for the related record.

A relation is considered dereferenced if it contains more than one key.

The syntax was chosen for a number of reasons:

- We would like to avoid confusion with JSONRef as it is already implemented in Invenio, and will work differently than JSONRef.
- We would like a syntax that can be used both in the internal JSON document stored in the database, as well as in the external JSON used in the REST API.
- By using an object we can easily dereference it. I.e. ``{"id": "..."}`` is dereferenced to ``{"id": "...", "key1": "val1", ...}``. The key ``id`` always exists (unlike e.g. when using JSONRef).
- Relations should be to a persistent identifier for the related record to make the relation as stable as possible. Also, we wanted to avoid having mulit-key objects as possible persistent identifiers to ensure APIs could be as standardized as much as possible.
- We did not want to include e.g. type information for the related record, to keep the JSON small as well as keep the type information on the field definition itself, instead of inside the referenced object.

#### Persistent identifiers

We imagine that the persistent identifiers depends on the related record. If the related record uses Invenio-PIDStore, it would be the persistent identifer stored in PIDStore. If the related record just uses the record's table UUID as persistent identifier, this value could be used.

Persistent identifiers should not be renamed to avoid having to go update the relations in database records (i.e the persistent identifier is actually persistent!).

If renaming is needed, it could be done by:

- Adding a new persistent identifier.
- Redirect the old persistent identifier to the new.
- Change all records to use the new persistent identifier
- Delete the old persistent identifier.

#### Integrity

The checking of integrity (i.e. that a related record exists), can be done via a simple primary key look up for the persistent identifier.

#### Dereferencing

The dereferencing mechanism should allow for fine-grained control of which fields are dereferenced. This is because the reference object may be big, and only a subset of fields is relevant to dereference.

For instance, take a record which should be related:

```json
{
    "id": "1234",
    "a": "",
    "b": ""
}
```

The relation in another record would look like below:

```json
{"id": "1234"}
```

When dereferenced, it should be possible to only include the key ``a`` and not ``b``:

```json
{"id": "1234", "a": ""}
```

#### Denormalization

Dereferencing a relation can be compared to denormalizing the data or caching the data. This means, that once the related record changes, the places where it was dereferenced/cached must be updated.

The issues with that is:

- **Cardinality**: If a related record is referenced by 1 million records, you have to go and dereference 1 million records again.

- **Concurrency**: With high cardinality, dereferencing will take time, and thus during this time the system can operate with stale data. For instance a new entry may be added, that must be picked up for dereferencing to avoid having it around forever with stale data.

##### Database store relations only

Due to the cardinality and concurrency issues, the database should always only store the relation and never a dereferenced version. This avoid that records in the database has to be updated.

##### Version counters

All records have a version counter for optimistic concurrency control. Thus, if everytime we dereference a relation we also store the version counter of the referenced record, we will be able to reliably detect which records must be dereferenced again.

As searches a likely to happen in Elasticsearch, we suggest to make a single compound key

We suggest adding a ``@v`` key in the *dereferenced data* (not in the database) that includes the version counter of the referenced record:

```json
{"id": "1234", "@v": 18, "a": ""}
```

It should be investigated if this if sufficient in order to proper queries in Elasticsearch. Alternatively, we will need to create a compound key concatenating the id and the version - for instance:

```json
{"id": "1234", "@v": "v18:1234", "a": ""}
```

Note, we cannot replace the ``id`` key, as it must be stable between the normalised and denormalised version.


### Programmatic API


#### Defining relations

Relations could be defined by a new system field named ``RelationsField``. The reason for using the name "relations" is to avoid confusion with REST API HATEOS links in the responses.

```pythton
class Record(RecordBase):
    relations = RelationsField(
        resource_type=PIDRelation('metadata.resource_type', ResourceTypeRecord.pid),
        subjects=PKListRelation('metadata.subjects', SubjectsRecord),
    )
```

Note that the class property name (``relations``) can be named whatever you like (e.g. ``vocabularies``).

We proposed four relationship types:

- ``PIDRelation`` - a relation using PIDStore for lookup.
- ``PKRelation`` - a relation using record UUIDs for lookup (i.e. a record's primary key).
- ``PIDListRelation`` - same as ``PIDRelation`` but for list of referenced objects.
- ``PKListRelation`` - same as ``PKRelation`` but for list of objects.

An alternative, would be to specify relations individually:

```python
class Record(RecordBase):
    resource_type = PIDRelation('metadata.resource_type', ResourceTypeRecord.pid)
```

However, this will make it a bit more diffcult to operate on all relations at once (e.g. ``record.relations.dereference()``).

##### Limiting attributes

Limting the attributes could be done as:

```python
PIDRelation('metadata.resource_type', ResourceTypeRecord.pid, attrs=[
    'type', 'subtype', 'some.nested.key'])
```

#### Dereferencing / retrieving the record

```python
# No relation defined
record.relations.resource_type() is None

# Getting via metadata
record['metadata']['resource_type']
# - return only id if  not dereferenced
# - return id, type and subtype

# Get the related record:
# - if not dereferenced: query db and store dereferenced
# - if dereferenced: use dereferenced
rt = record.relations.resource_type()
# Get the related record (force a DB query)
rt = record.relations.resource_type(force=True)

# Get many records:
for s in record.relations.subjects():
    # ...

# Dereference all records at once
record.relations.dereference()
```

#### Setting related records


```python
# Via metadata
Record({
    "metadata": {
        "resource_type": {"id": "text-article"}
    }
})

# Via system field
record.relations.resource_type = ResourceType.get_record('text-article')
record.relations.resource_type = 'text-article'


record.relations.subjects.append('3c1eb018-0031-4488-b11d-ad6831997e89')
```


#### Integrity checking and validation


```python
# Checking if a specific ID exists
Record.relations.resource_type.exists('text-article')
    # -> ResourceTypeRecord.pid.resolve('text-article')

# Checking if many ids in one go.
Record.relations.subjects.exists_many(['3c1eb018-0031-4488-b11d-ad6831997e89', ...])
    # -> SubjectsRecord.get_record('3c1eb018-0031-4488-b11d-ad6831997e89'))

# Validate all relations
record.relations.validate()
# Above could be hooked directly into "record.validate()"
```

#### Updating outdated dereferenced relations

Updating the outdated references is not doable only via the data access layer.

```python
# API to scan for records
service.scan(relations={'resource_type': [('text-article', 18), ...]})

# API to using above scan to reindex
service.reindex(relations={'resource_type': [('text-article', 18), ...]}, bulk=False)
```

### Additions/Changes

The above design would require changes to the modules as described below.


#### Invenio-Records

- Create a ``RelationField`` system field.
    - Support direct integration with the Record.validate() function via record hooks.
    - Write a dumper extension that can dereference the fields on indexing time.
- Create a ``PKRelation`` and ``PKListRelation`` relation type.

#### Invenio-Records-Resources

- Records:
    - Create a ``PIDRelation`` and ``PIDListRelation`` relation types.
    - Complete the ``UUIDResolver``.
- Service:
    - Add a ``scan()`` method similar to search, that can be used to iterate over all records given a specific search query.
    - Add a ``reindex()`` method that given a query will send matching corresponding records for direct or bulk reindexing.

- Add a ``suggest()`` or expand ``search()`` to allow for search as you type completion.

## How we teach this

TODO

## Drawbacks

TODO

## Alternatives

TODO

## Unresolved questions

TODO

## Resources/Timeline

The RFC will be implemented as part of the upcoming Invenio sprint.
