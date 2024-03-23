- Start Date: 2024-03-22
- RFC PR: [#<PR>](https://github.com/inveniosoftware/rfcs/pull/<PR>)
- Authors: Mirek Simek <miroslav.simek@cesnet.cz>

# Allow pluggable transfers for record files

## Summary

Currently the transfer types are fixed to Local, Fetch and Remote and can not be extended.
Users might benefit from having this system extensible so that they can write/use
their own transfer types.

## Motivation

We would like to implement at least one additional transfer type - Multipart. This
transfer type will allow users to upload big files split into multiple parts and
will be contained as a core functionality.

Another example might be a S3Copy transfer type, which would copy the file between
two S3 buckets directly on S3 cluster, without passing through invenio server.

## Detailed design

All the current functionality is in `transfer.py` and related modules. The `TransferType`
is a non-extendable enum, transfers extensions of `BaseTransfer` and not pluggable
(if/then/else inside the `Transfer` class).

Proposal:

* Change the `TransferType` from enum to a normal class. Create constants for the built-in
  types - turn `TransferType.LOCAL` to `TRANSFER_TYPE_LOCAL`
* Create a `TransferRegistry` class modelled after ServiceRegistry. Add `current_transfer_registry`
  proxy.
* Modify the `Transfer` class to use the registry
* Modify the sources to use the updated class and constants

* Fix the `dump_status` method on schema. The current implementation says:
```
    def dump_status(self, obj):
        """Dump file status."""
        # due to time constraints the status check is done here
        # however, ideally this class should not need knowledge of
        # the TransferType class, it should be encapsulated at File
        # wrapper class or lower.
```

Approaches to fix this (to be decided which one to take):

1. Create a `status=ModelField()` on the FileRecord class, with values `pending`, `completed`, `aborted`, `failed` .
   The value of the field would be stored inside the database in the file record
2. Create a `status=StatusSystemField()` that would fetch/store the status from the file's metadata (.model.json)
3. Create a `status=StatusSystemField()` that would get the file's transfer and delegate the status to it.

The first two options add a value stored to the database but are cleaner, #3 would create a reference
from invenio_records_resources.records to invenio_records_resources.services.files.

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
