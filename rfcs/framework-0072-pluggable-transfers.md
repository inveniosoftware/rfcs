- Start Date: 2024-03-22
- RFC PR: [#<PR>](https://github.com/inveniosoftware/rfcs/pull/<PR>)
- Authors: Mirek Simek <miroslav.simek@cesnet.cz>

# Allow pluggable transfers for record files

## Summary

Currently the transfer types are fixed to Local, Fetch and Remote and can not be extended. Users might benefit from having this system extensible so that they can write/use their own transfer types.

## Motivation

We would like to implement at least one additional transfer type - Multipart. This transfer type will allow users to upload big files split into multiple parts and will be contained as a core functionality.

Another example might be a S3Copy transfer type, which would copy the file between two S3 buckets directly on S3 cluster, without passing through invenio server.

## Detailed design

All the current functionality is in `transfer.py` and related modules. The `TransferType` is a non-extendable enum, transfers extensions of `BaseTransfer` and not pluggable (if/then/else inside the `Transfer` class).

Proposal:

- Change the `TransferType` from enum to a normal class. Create constants for the built-in types - turn `TransferType.LOCAL` to `TRANSFER_TYPE_LOCAL`
- Create a `TransferRegistry` class modelled after ServiceRegistry. Add `current_transfer_registry` proxy.
- Modify the `Transfer` class to use the registry
- Modify the sources to use the updated class and constants

- Fix the `dump_status` method on schema. The current implementation says:
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
   The value of the field would be stored inside the database in the file record as a new database column.
2. Create a `status=DictField("status")` that would take the status from file's metadata (.model.json)
3. Create a `status=StatusSystemField()` that would get the file transfer and delegate the status query to it.

The first two options add a value stored to the database but are cleaner, #3 would create a reference from `invenio_records_resources.records` to `invenio_records_resources.services.files`.

## Example

Another example might be a specialized transport for handling sensitive data in Crypt4GH file format before they are stored in the repository (handling encryption keys etc.)

## How we teach this

This is rather an internal feature (the same level as services, service configs etc.) We might add a section to [developer topics](https://inveniordm.docs.cern.ch/develop/topics/) or just keep the API docs.

## Drawbacks

A slightly more complicated code.

## Alternatives

The only alternative is having separate API endpoints for handling the use cases above. This RFC provides a cleaner solution.

## Unresolved questions

- [ ] Decide how the `status` field should be implemented

**Alex:**
For the decision on the status field, I have a slight preference for #3 (i.e. computing/querying dynamically instead of storing), since I could imagine the BaseTransfer sub-classes, implementing the logic... Though I see that this introduces a dependency in the wrong direction... Would it make sense to maybe remove status from the record class, and just compute/dump it on the service marshmallow schema? Maybe a 3rd person without all the past implementations baggage could comment on a cleaner solution 🙂

Then another less critical concern:
 
Maybe there should be a way to configure different transfer types per file service... I could imagine e.g. not wanting to have remote files for community logos. But TBH, given how hidden the feature is, we could roll with having everything available by default across all services, and then later on adding the config if we see weird cases emerging

## Resources/Timeline

Both CESNET and Munster need multipart upload that depends on this feature in a short time (May 2024). Both CESNET and Munster are willing to put their resources into the implementation and testing.
