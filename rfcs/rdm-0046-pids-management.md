- Start Date: 2021-04-01
- RFC PR: [#46](https://github.com/inveniosoftware/rfcs/pull/46)
- Authors: Lars Holm Nielsen, Nicola Tarocco, Pablo Panero
- State: IMPLEMENTED

# PIDs management in InvenioRDM

## Summary

This RFCs describes how identifiers management is handled in InvenioRDM. Users will be able to input or register external identifiers for a record. For example, register a new DOIs (DataCite, Crossref, etc.), input a DOI previously registered in another system and input any other alternate identifier.

## Motivation

See also [RDM Persistent identifiers RFC](https://github.com/inveniosoftware/rfcs/blob/9b7e5cc513981a2711edc5f7cbf3baafa5d6be27/rdm/0003-persistent-identifiers.md#external-persistent-identifiers).

### User use cases

- As a user, I want to create a record without any external identifier.
- As a user, I want to create a record with an optional external identifier.
- As a user, I want to obtain a new DOI for a newly submitted draft.
- As a user, I want to copy/paste my previously obtained DOI for a newly submitted draft.
- As a user, I want my DOI to be updated when I publish an updated version of the DOI.
- As a user, I do not want to have a DOI registered for restricted records


### Admin use cases

- As a admin, I want to manage multiple DOI prefixes, so that I don't need a repository installation per prefix.
- As a admin, I want to take over management of a dataset with a DOI from another repository, so that records can be migrated between repositories.
- As a admin, I want that a concept DOI pointing always to the latest version is also registered when registering a DOI for a version.
- As a admin, I want to customize the creation of PID value by resource type (e.g. publication DOI, something else a ISBN, etc.)

## Detailed design

Identifiers are stored in the `pids` field. They can be categorized in 2 groups:

1. **Unmanaged**: the user provides the identifier obtained in another system and it is treated as a simple string value.
2. **Managed**: the user delegates to Invenio the ability to reserve, register, unregister or more actions an identifier.

In both cases, such identifier must be unique in the system: it will be registered in `PIDStore` and its uniqueness will be checked.

Each version of a record could have multiple identifiers. Some of them can be managed and some unmanaged.

It is **not** a valid use case to mix managed and unmanaged identifiers in different versions of the same record.
For a given identifier, e.g. a DOI from `datacite`, the user will input the identifier (*unmanaged*) or it will automatically obtain it from the system (*managed*). The following versions of this records **must** have the same type of identifier as the first initial version (managed or unmanaged).

A record can have only of instance of a given identifier: for example, it cannot have 2 DOIs. If multiple instances of the same identifier are needed, all except the main one must be set as alternate/other identifiers.

Alternate/Other identifiers are simple scheme/identifier tuples recorded in the field `identifiers` of the record's metadata. The only validation check on these values is the compliance of its value with its scheme, no duplication or more complex validation is performed.

### Data model

#### Identifiers

Managed and unmanaged identifiers are stored in the `pids` field (see more in the [metadata reference guide](https://inveniordm.docs.cern.ch/reference/metadata/#external-pids)) with the following attributes:

```json
{
    ...
    "pids": {
        "<scheme>": {
            "identifier": "string",
            "provider": "string",
            "client": "string"
        }
    },
    ...
}
```

Description of the fields:

* The key of each dictionary item is the identifier's scheme e.g. doi, isbn, etc.
* `identifier`: the value of the identifier.
* `provider`: the provider of the identifier (e.g. DataCite)
* `client`: the client is who is in charge of managing the identifier (e.g. the Zenodo account/credentials for DataCite)

For example:

```json
{
    ...
    "pids": {
        "doi": {
            "identifier": "10.5281/zenodo.1234",
            "provider": "datacite",
            "client": "zenodo",
        }
    },
    ...
}
```

A system field is added to the Record (and Draft) API, to allow them to be access via `record.pids...`

NOTE: This field is currently a mere `DictField`, however it might need to be enhanced to allow for nested access `record.pids.doi`
```
pids = DictField("pids")
```

#### Alternate/other identifiers

Unmanaged identifiers are stored in the `identifiers` field (more in the [metadata reference guide](https://inveniordm.docs.cern.ch/reference/metadata/#identifiers-0-n)) with the following attributes:

```json
{
    ...
    "identifiers": [
        {
            "scheme": "string",
            "identifier": "string",
        },
    ],
    ...
}
```

For example:

```json
{
    ...
    "identifiers": [
        {
            "identifier": "10.7612/abcd",
            "scheme": "doi"
        }
    ],
    ...
}
```

The possible scheme values are stored in a vocabulary.

### Service layer

The validation and the management of PIDs are implemented as a new component in the service layer.
The `RDMRecordSchema` should have 2 new fields: `pids` and `identifiers`.

#### PIDs

The marshmallow schema for PIDs should retrieve the available provider and validate the input to avoid to store a non-existing provider. The PIDs component will then delegate to the provider the validation of the given PID, for example if the given identifier is valid for the provider.

Managed providers can enforce the creation of a PID or make it optional: in the first case, when the identifier is not part of the record (not previously reserved), it will be automatically created and added to the record.

After having published a draft, PIDs cannot be modified anymore (for example when editing again the record).

#### Identifiers

The Marshmallow schema reflects the structure defined above, a list of dictionaries with `scheme` and `identifier` (value).
The list is done using the [IdentifierSet](https://marshmallow-utils.readthedocs.io/en/latest/api.html#marshmallow_utils.fields.IdentifierSet) field of Marshmallow-Utils. This field allows to de-duplicate (i.e. not allow multiple identifiers with the same scheme). However, in the case of the *alternate* `identifiers` field **scheme duplication is allowed**.

Each item of the list (set) uses the [IdentifierSchema](https://marshmallow-utils.readthedocs.io/en/latest/api.html#marshmallow_utils.schemas.IdentifierSchema) of Marshmallow-Utils. This scheme will validate the identifier's value against the schema if it is supported by IDUtils, otherwise it will simply accept it as is.

#### ExternalPIDsComponent

The component hooks into the different service methods, and therefore into the records lifecycle. Note that it is named `ExternalPIDsComponent`, whereas the *external* has been removed from the rest of the objects/classes. This is due to the existence of another very similar component named `PIDComponent`. used to manage the *internal* pids (e.g. RECID).

The implementation will respect the following:

* when a draft is **published**, each `pids` field value that is in the draft is validated by each provider. Required PIDs which are not provided will be created and registered. The final version of the `pids` field will be copied to the published record.
* when a published record is **edited**, there is only validation happening.
* when a **new version** of a published record is created, the `pids` field is cleared out to allow new PIDs to be provided.

### Providers configuration

Managed identifiers are stored in `PIDStore` and managed by providers.
The available providers can be configured in the records service:

```python
class RDMRecordServiceConfig:

    pids_providers = {
        "doi": {
            "datacite": DataCiteProvider,
            ...
        },
        ...
    }

```

A provider will have the following model/actions:

```python
class DataCiteProvider():
    name = "datacite"
    client = None

    def reserve(pid, record) - blocking, return identifier
    def register(pid, record) - blocking or async, return None
    def update(pid, record) - blocking or async, return None
    def delete(pid, record) - blocking or async, return None
    def get_status(pid)- blocking, return status
    ...
```

Injecting the record (can be the draft or the published record depending on the method) as param allows to be more flexible and, for example, customize the identifier value based on some fields values or handle identifier's registration in case of restricted records.

#### Reserve/status PID REST APIs

The new endpoint `/api/records/<pid_value>/draft/pids/<pid_type>` allows to reserve a new PID on `POST` and to delete a previously reserved PID on `DELETE` actions.
The endpoint will return the updated draft with or without the PID.

### Versioning

PIDs are different in each version of a record: for example, an unique DOI is registered for the version 1 of a record, and a different unique DOI is registered for the version 2 of a record.
In the specific case of DOI, a `Concept DOI` is also registered after the first publishing of the record: this role of the concept DOI is to point always to the latest version of the record.

Example:

* published record with recid `k34Nd` - version 1:
    * DOI: `10.1234/zenodo.LAOP.v1`, resolving to recid `k34Nd`
    * Concept DOI: `10.1234/zenodo.ABCD`, resolving to recid `k34Nd`
* published record with recid `m1L2u` - version 2:
    * DOI: `10.1234/zenodo.MDJY.v1`, resolving to recid `m1L2u`
    * Concept DOI: `10.1234/zenodo.ABCD`, resolving to recid `m1L2u`

Note that the concept DOI did not change value, it only change to which record it will resolve to.

### UI deposit form

The deposit form will have a new component to manage PIDs and another component to manage alternate identifiers.
The PIDs component will use the providers configuration injected via Jinja template to allow the user to choose between unmanaged and managed PID, and in the latter to reserve a new PID.

## Example

A typical workflow of a research data digital repository providing DOIs could be the following.

When the user starts a new upload, the form will display the PID component. The user will then choose between providing its own DOI, obtained somewhere else, so that it can be copy/pasted in the input field, or choose to obtain a new DOI by clicking on a dedicated UI button. The DOI is then created in the backend and the draft record is returned with the new DOI, which appears in the form.

At this stage, the DOI is only created and it is not yet registered with the provider (e.g. DataCite/CrossRef).

When the user publishes the new upload, the system will then register the DOI with the provider. The DOI should now resolve to the published record.

When editing such record, the UI form for PIDs will appear as disabled for managed PIDs, given that it cannot be changed.
When creating a new version of the record, the UI form for PIDs will appear as when creating a new upload, to allow the input of new PIDs.

## How we teach this

A good way to teach might be:

1. Introduce the concepts of Persistent Identifiers: what are they, why are they useful in particular in the research world.
2. Clarify what the difference between managed, unmanaged PIDs and alternate identifiers. The explanation should mention that the PIDStore database table will enforce uniqueness for the formers.
3. Show a demo: show the operation of DOI creation via the UI and explain what is going to happen when publishing, edition or creating a new version.
4. Explain how to configure (for what is possible to configure) providers, extend existing or create new ones.

## Drawbacks

At the time of writing, the only implemented PID support is for DOIs with DataCite.
Extending and allowing creation of new providers is currently cumbersome and code should be refactor to allow developers to easily implement new functionalities.

## Alternatives

Alternatives were considered mainly when designing the `Provider` interface: at the end it has been decided to keep the `is_required` and `system_managed` config options outside the provider class, but it might be confusing and probably not be the ideal solution.

```python
pids_providers = {
    "doi": {
        "datacite": {
            "provider": DOIDataCitePIDProvider,
            "required": True,
            "system_managed": True,
        },
    ...
    }
}
```

The technical challenge here was to allow the configuration of such params: DOIs with DataCite might be required for some digital repositories, e.g. Zenodo, but they might be not for others. One other implementation option evaluated was to take advantage of the `partial` function, and for example define `partial(DOIDataCiteProvider, required=True)`, which has been discarded: class attributes cannot be accessed (`DOIDataCiteProvider.name`) when using `partial`.
A better implementation should rethink and redesign this configuration, while keeping existing implementation.

## Unresolved questions

At the time of writing, the following features have not been implemented yet:

* take into account record restrictions when registering PIDs
* concept DOIs
* ensure that a given PID do not change from managed to unmanaged (or viceversa) between versions
* allow easy configuration of PIDs providers

See the related GitHub issue [here](https://github.com/inveniosoftware/product-rdm/issues/9).
