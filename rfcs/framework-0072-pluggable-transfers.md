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

**Alex' suggestion**
For the decision on the status field, I have a slight preference for #3 (i.e. computing/querying dynamically instead of storing), since I could imagine the BaseTransfer sub-classes, implementing the logic... Though I see that this introduces a dependency in the wrong direction... Would it make sense to maybe remove status from the record class, and just compute/dump it on the service marshmallow schema? Maybe a 3rd person without all the past implementations baggage could comment on a cleaner solution 🙂

- [ ] Alex: Then another less critical concern:
 
Maybe there should be a way to configure different transfer types per file service... 
I could imagine e.g. not wanting to have remote files for community logos. 
But TBH, given how hidden the feature is, we could roll with having everything available 
by default across all services, and then later on adding the config if we see weird 
cases emerging

- [ ] Clarify the "storage_class" vs "transfer_type"

It seems that inside invenio_files the storage_class was meant to specify
the details of the storage backend (as the tests suggest, at least) - for example
'A' as 'Archive'.

In invenio_records_resources, the storage_class on the initial schema means the type of 
transfer that will be used (Local, Fetch, Remote). This is independent of the storage
backend and is a bit confusing.

- [ ] FileSchema.uri vs. links section

For the initialization of files, 'uri' can be used to specify the location of the file
for REMOTE/FETCH transfers. Later on, clients can use this URI for these transfers to get
a direct access to the referenced data.

During "dumping" the file record, a links section with `content` uri is created. 
This is a bit confusing as clients do not know which URL should be used.

Proposal: keep the uri only inside the initial file record. Do not serialize it
afterwards. Clients should use the links section for all the URIs. The content
url can return a HTTP 302 redirect to the actual file location. This way counters
will work as well.

- [ ] FETCH transfer type and private URIs

The FETCH transfer type is used to fetch the file from a remote location. The uri
of the external location is visible to anyone who has a read access to the record,
not just the uploader. As the uri might contain credentials (e.g. S3 signed URL),
this might be a security issue.

Proposal: do not serialize the uri of the FETCH transfer type. the fetch_file task
would have to be changed to not to use the service but fetch the uri directly.

- [ ] IfFileIsLocal generators

The permissions in https://github.com/inveniosoftware/invenio-records-resources/blob/master/invenio_records_resources/services/files/generators.py#L27
take into account only the files with storage class (== transfer type) "L". This means that
the permissions will fail for "M" (Multipart transport) files. Is this intended?

The same problem is in invenio-rdm-records/services/generators.py

Proposal: either change the meaning of "Locality" and keep the permissions as they are
or change the permissions to take into account all the files. Example from RDM that I do 
not understand (can_draft_set_content_files - why there is the can_review permission and 
not record owner?):

```python
    can_draft_create_files = can_review
    can_draft_set_content_files = [
        # review is the same as create_files
        IfFileIsLocal(then_=can_review, else_=[SystemProcess()])
    ]
    can_draft_get_content_files = [
        # preview is same as read_files
        IfFileIsLocal(then_=can_draft_read_files, else_=[SystemProcess()])
    ]
    can_draft_commit_files = [
        # review is the same as create_files
        IfFileIsLocal(then_=can_review, else_=[SystemProcess()])
    ]
```

- [ ] MultipartStorageExt

In order to make direct multipart upload to remote storage working we need a PyFS storage
to implement multipart aware methods. These are currently summarized in the MultipartStorageExt
interface that lives in invenio-records-resources - 
[MultipartStorageExt class in pull request](https://github.com/mesemus/invenio-records-resources/pull/3/files#diff-ab7d7ef448eaae0f109622e167d9168d7f0ad56dbba383d6d681b9cc17b62bcfR14)


We can not directly implement this interface in storage backends (for example invenio-s3) as
it would introduce a dependency in the wrong direction.

Proposal: either move the MultipartStorageExt to invenio-files-rest/storage/pyfs or do not check
for the interface but just check for the implemented methods (currently implemented this way in POC
implementation - see for example 
[check for multipart_set_content method](https://github.com/mesemus/invenio-records-resources/pull/3/files#diff-ab7d7ef448eaae0f109622e167d9168d7f0ad56dbba383d6d681b9cc17b62bcfR177) )



## Resources/Timeline

Both CESNET and Munster need multipart upload that depends on this feature in a short time (May 2024). 
Both CESNET and Munster are willing to put their resources into the implementation and testing.
