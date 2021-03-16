- Start Date: 2021-03-05
- RFC PR: [#42](https://github.com/inveniosoftware/rfcs/pull/42)
- Authors: Lars Holm Nielsen, Nicola Tarocco, Alex Ioannidis, Jose Benito Gonzalez Lopez
- State: DRAFT

# JSONSchema resolution and registry

## Summary

The RFC proposes to extend Invenio-Records with an optional feature to inject a custom RefResolver and to extend the existing registry in Invenio-JSONSchema with an function to provide a schema store/registry to the RefResolver.

## Motivation

Invenio stores records (JSON documents). The structure of these JSON documents is validated with JSONSchemas. In a nuthsell, it works like below.

We have a JSONSchema that is named with an ``$id`` keyword (the keyword is required and the value [must be an URI](https://json-schema.org/draft/2020-12/json-schema-core.html#rfc.section.8.2.1)):

```python
# JSONSchema
{
    "$id": "...",
    ...
}
```

Next, we have a record API class, that sets the ``$schema`` keyword in the record, to have the record validated by a JSONSchema:

```python
# Record API
class MyRecord(Record):
    schema = ConstantField("...")
```

Thus the JSON of a record looks like this:

```python
# Record (JSON document to be validated)
{
    "$schema": "...",
    ...
}
```

**What's the issue?**

1. The record's ``$schema`` value does not match the JSONSchemas ``$id``.

2. The record's ``$schema`` value is dynamic and based on the ``JSONSCHEMA_HOST`` configuration variable.

3. The record's ``$schema`` can be an external URL, in which case an HTTP request will be made to fetch the external URL.

4. The internals of how ``$schema`` is matched against a local schema is difficult to follow and understand.

Above issues leads to developers and used being confused on how to correctly set ``JSONSCHEMA_HOST`` and in general not understanding how a core feature of Invenio (JSONSchema validation) works. Last, requests to remote resources opens up for potential security issues as well as bad performance.

## Background

Invenio-Records provides an API for validating records according their specified JSONSchema:

```python
from invenio_records.api import Record
data = {'$schema': 'http://adomain.com/myschema.json', ... }
Record(data).validate()
```

Invenio-Records delegates the validation to the [jsonschema library](https://python-jsonschema.readthedocs.io/en/latest/#) which does something like this:

```python
from jsonschema import validate, RefResolver

resolver = RefResolver(...)

validate(
    data,
    {'$ref': data['$schema']},
    resolver=resolver,
)
```

What happens:
- ``validate()`` takes the schema as a dictionary.
- Instead of the schema dictionary, we pass in a JSONRef to the schema.
- The ``RefResolver()`` is used to resolve the JSONRef to the schema.

By default, the RefResolver will make an HTTP requests to resolve the URL (i.e. ``http://adomain.com/myschema.json``). In order to avoid the HTTP request, the modules [jsonresolver](https://pypi.org/project/jsonresolver/) and [Invenio-JSONSchemas](https://github.com/inveniosoftware/invenio-jsonschemas) is used to catch the HTTP requests and create a registry of JSONSchemas that the HTTP request is matched against. If no local schema is found, the HTTP request escapes and will make the request.

Note that the RefResolver is also used to dereference internal JSONRefs inside the schema (e.g. often used for definitions).

**Undesired behaviours**

1. The current JSONRref resolution behaviour implemented by jsonresolver and Invenio-JSONSchemas is very hard to follow through and understand how it actually works.

2. The resolution behavior relies on the ``JSONSCHEMA_HOST`` config variable, which may conflict with the ``id`` attribute defined in the schema, making it non-intutive why a schema is not being picked up, and it makes it hard to set ``JSONSCHEMA_HOST`` to a correct value.


## Detailed design

### Requirements

- All changes must be backward compatible and opt-in to avoid breaking existing code.

- The JSONRef resolution mechanism must never make HTTP reqeusts or otherwise access external services to resolve a schema.

- The ``$id`` keyword of a JSONSchema must be the URI used in the record's ``$schema`` keyword and must not be dynamic.

### JSONSchema validation of records

We propose to introduce a new URI scheme ``local://`` (i.e. not an official IANA-registered scheme). We use these URI scheme to name and reference local JSONSchemas.

For instance a record, would reference a JSONSchema local to Invenio like this:

```python
# A record
{
    "$schema": "local://records/record-v1.0.0.json",
    ...
}
```

The JSONSchema itself would use the same URI inside the ``$id`` keyword:

```python
# A JSONSchema
{
    "$id": "local://records/record-v1.0.0.json",
    ...
}
```

This would also allow to build nested schemas with references to local schemas:

```python
# A JSONSchema
{
    "$id": "local://records/record-v1.0.0.json",
    "type": "object",
    "properties": {
        "access": {
            "$ref": "local://records/access-v1.0.0.json"
        }
    }
}
```

In order to get the above JSONSchema validation to work, we need to customize the RefResolver

1. Ability to resolve the ``local://`` URI scheme.
2. Adapt our JSONSchema registry to work with the resolver.

### Enable custommization of RefResolver and the resolver store

We propose to extend Invenio-Records with:

- Allow overwriting the extension's default ``ref_resolver_cls`` (in ``ext.py``).
- Allow injecting a store/registry to the ``RefResolver``.

Both options would be set via configuration (making them opt-in):

```python
RECORDS_REFRESOLVERCLS = 'invenio_jsonschemas.resolver:LocalRefResolver'
RECORDS_RESOLVER_STORE = 'invenio_jsonschemas.proxies.current_refresolver_store'
```

This enables to hook in the new RefResolver to work with the ``local://`` URI scheme, and it allows us to inject our custom JSONShema registry with usuing the ``local://`` URI scheme.

### Build a custom RefResovler

We propose to extend Invenio-Records with new ``InvenioRefResolver`` capable of

1. Preventing requests to remote resources.
2. Resolving a ``local://`` URI scheme

**Preventing HTTP requests**

The new RefResolver should prevent making any remote requests by overwriting the [``RefResolver.resolve_remote()``](https://github.com/Julian/jsonschema/blob/ec29d5c37953727d0a91f51393e8b40d2f7f194c/jsonschema/validators.py#L806) method:

```python
class InvenioRefResolver(RefResolver):
    def resolve_remote(self, uri):
        raise exceptions.RefResolutionError(...)
```

An alternatively to this, would be to provide handlers to the existing RefResolver, but you would need to know all possible URI schemes to be fully sure that no external requests were made.

**Resolving a ``local://`` URI scheme**

Resolving the ``local://`` URI scheme can likely be done simply by providing the registry store since a local schema is always looked up there first. Alternatively, we need to overwrite the ``RefResolver.resolve_remote()`` with this capability.

### Build a RefResolver store from the existing registry

Invenio-JSONSchema already have a registry of schemas as well as the associated loading mechanisms. We propose to simply extend this registry with a method to build a **store** compatible with the RefResolver:

```python
class InvenioJSONSchemasState(...):
    def refresolver_store(self):
        # ...
```

**Schema identifier**

The current registry uses an identifier for registering a schema based on the ``JSONSCHEMA_HOST`` configuration variable and the path of the schema. Instead of using the existing registry, we propose to simply strip the ``JSONSHEMA_HOST`` part.

For instance instead of:

```
{JSONSCHEMA_HOST}/schemas/records/record-v1.0.0.json
```

we would use:

```
local://records/record-v1.0.0.json
```

Alos, we should validate that the schema's ``$id`` keyword matches the URI (e.g. ``local://records-record-v1.0.0.json``) under which we register it.

**Injection of the store**

The store itself could either be provided directly by the ``InvenioRefResolver`` or simply allow Invenio-Records to inject any RefResolver store.


## How we teach this

This feature should first be launched under the hood as part of Invenio-Records/JSONSchemas, and be tested out first with InvenioRDM.

Once ready, we can make it the default in a future Invenio v4, but we would not start advertising until fully tested for all potential problems, and it goes a bit hand in hand with disabling the JSONRef mechanisms for managing record relations.

## Alternatives

We could do only the Invenio-Records modifications and leave Invenio-JSONSchemas as is. The store could be provided by users themselves. This would however significantly change the existing pattern.

## Drawbacks

The drawback is that we're allow two ways of doing the schema resolution. We however don't see that we can remove the existing method as it will break all existing code, and the existing method is too complex to understand. We think that this is a way to progressively create a better product.

## Resources/Timeline

This is right now causing issues for InvenioRDM uses, so we would implement it in InvenioRDM to be released by end-April latest.

### Tasks

- Invenio-Records:
    - [ ] Allow overwriting the RefResolver and inject the store.
    - [ ] Build the InvenioRefResolver
    - [ ] Add tests (specifically show that nested schema's work).
- Invenio-JSONSchemas:
    - [ ] Add support for exporting the registry with ``local://`` schema.
    - [ ] Add validation of if ``$id`` matches the registry key.
