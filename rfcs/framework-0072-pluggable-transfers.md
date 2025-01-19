- Start Date: 2024-03-22
- RFC PR: [#<PR>](https://github.com/inveniosoftware/rfcs/pull/<PR>)
- Authors: Mirek Simek <miroslav.simek@cesnet.cz>

# Pluggable Transfer Types for Record Files

## Summary

Currently, the system only supports three fixed transfer types—**Local**, **Fetch**,
and **Remote**—and does not allow adding new ones. By making transfer types pluggable,
users could implement and integrate their own custom transfers.

## Motivation

We plan to introduce a built-in **Multipart** transfer type that enables
uploading large files in separate chunks, improving performance and
reliability.

In addition, one could, for example, develop an **S3Copy** transfer type to
copy files directly between S3 buckets—bypassing the Invenio server—to reduce
bandwidth usageand improve efficiency.

## REST API changes

To make file transfers more consistent, we are introducing a new `transfer`
field to replace the current inconsistent usage of `storage_class`.

- In **invenio-files-rest**, `storage_class` indicates the file’s intended location or usage (e.g., `archive`, `standard`).  
- In **invenio-records-resources**, it has been (mis)used to denote the transfer type (e.g., `local`, `fetch`, `remote`).

Going forward, the new `transfer` field will hold all transfer-related
information, while `storage_class` will continue to serve its original purpose
from `invenio-files-rest`.

The `transfer` field will be a dictionary with the following structure:

### Local transfer

Equivalent to current `storage_class == "L"`. The `transfer` section can be omitted.

```json
[{
    "key": "file.txt",
    "transfer": {
      "type": "L",
    }
}]
```

### Fetch transfer

Equivalent to current storage_class == "F"

```json
[{
    "key": "file.txt",
    "transfer": {
        "type": "F",
        "url": "https://example.com/file-to-fetch.txt",
    }
}]
```

### Remote transfer

Currently not supported on REST API level.

```json
[{
    "key": "file.txt",
    "transfer": {
        "type": "R",
        "url": "https://example.com/linked-file.txt",
    }
}]
```

### Multipart transfer

Multipart transfer is a new feature that allows uploading large files in
multiple chunks. The `transfer` field will contain additional information
about the transfer, such as the number of chunks and their size (needed
for the generic implementation for PyFS backend).

```json
[{
    "key": "file.txt",
    "size": 3000,
    "transfer": {
        "type": "M",
        "url": "https://example.com/multipart-upload",
        "chunks": 3,
        "chunk_size": 1024,
    }
}]
```

### Backward compatibility

Although this change technically breaks backward compatibility,
it should not cause significant issues. The **Fetch** transfer—introduced
in **RDM 11**—was marked as experimental, and there have been no other known
uses of `storage_class` for transfer-related purposes.

## Permissions

Permissions will be extended to support defining rules for each transfer type.
This is achieved through a new `IfTransferType(type, then_, else_)` permission generator,
which allows you to specify custom permissions based on the transfer type.

Example:

```python
# invenio.cfg

from invenio_records_resources.services.files.generators import IfTransferType
from invenio_records_resources.services.files.transfer import REMOTE_TRANSFER_TYPE
from invenio_administration.generators import Administration

class MyRepositoryPermissionPolicy(RDMRecordPermissionPolicy):
  can_draft_create_files = RDMRecordPermissionPolicy.can_draft_transfer_files + [
        IfTransferType(REMOTE_TRANSFER_TYPE, Administration())
  ]

RDM_PERMISSION_POLICY = MyRepositoryPermissionPolicy
```

## Detailed Design

All current transfer functionality resides in `transfer.py` and related modules.
At present, `TransferType` is a non-extendable enum, and transfers based on
`BaseTransfer` are not pluggable (they rely on `if`/`then`/`else` logic within
the `Transfer` class).

### Transfer types

We will convert `TransferType` from an enum into a regular class.
Will have string constants for built-in types, e.g., the `TransferType.LOCAL` will be replaced with `TRANSFER_TYPE_LOCAL`.

### Transfer implementation

A new `Transfer` base class will be introduced with the following API:

```python
class Transfer:
    transfer_type: str
    Schema: ma.Schema

    def __init__(self, record: Record, key: str, file_service: FileService, uow=None): ...
    # file upload operations
    def init_file(self, record, file_metadata): ...
    def set_file_content(self, stream, content_length): ...
    def commit_file(self): ...
    def delete_file(self): ...

    # upload status
    def status(self): ...

    # send file to client
    def send_file(self, *, restricted, as_attachment): ...

    # get extra links that the transfer may provide
    def expand_links(self, identity, self_url): ...
```

Then for each built-in transfer, a dedicated implementation will be created.

### TransferRegistry

A `TransferRegistry` class will be introduced, modeled after the
`ServiceRegistry`. A new proxy with pre-filled transfers will be provided as
`current_transfer_registry`.

The transfer registry will be configured through the following options:

```python
RECORDS_RESOURCES_TRANSFERS = [
    "invenio_records_resources.services.files.transfer.LocalTransfer",
    "invenio_records_resources.services.files.transfer.FetchTransfer",
    "invenio_records_resources.services.files.transfer.RemoteTransfer",
    "invenio_records_resources.services.files.transfer.MultipartTransfer",
]
"""List of transfer classes to register."""


RECORDS_RESOURCES_DEFAULT_TRANSFER_TYPE = "L"
"""Default transfer class to use. 
One of 'L' (local), 'F' (fetch), 'R' (point to remote), 'M' (multipart)."""
```

### Initial vs. File schema

The distinction between `InitialFileSchema` and `FileSchema` has been blurred by
recent changes in **invenio-records-resources**, where `FileSchema` is now used
for `init_files` as well.

To clarify, we will turn `InitialFileSchema` into a mixin that is applied to
`FileSchema` automatically when a new file is created.

This approach lets us mark all “read only after creation” fields, such as
`key`, `storage_class`, `mimetype`, `size`, and `checksum`, as `dump_only` in
`FileSchema`. In the `InitialFileSchema`, we will re-enable these fields for
writing by marking them as `load_only`.

This ensures that users can provide their own file schemas for different use
cases (e.g., media files vs. regular files) while still preserving the correct
handling of initialization parameters.

We will also introduce a `TransferSchema` to handle the new `transfer` field.
It will allow using different schemas for each transfer type. To achieve this,
we will leverage `marshmallow-oneofschema`, which is already included in the
project dependencies.

### Permissions

To be able to use the new `IfTransferType` permission generator, we need to
change the way permissions are handled inside the `init_files` method.

#### Current implementation

The current implementation calls:

```python
    record = self._get_record(id_, identity, "create_files")
```

which is defined as (abbreviated):

```python
record = self.record_cls.pid.resolve(id_, registered_only=False)
self.require_permission(identity, action, record=record, file_key=file_key)
```

Then a validation of the input data is performed and components called.

#### New implementation

As a user may specify multiple files in a single `init_files` call, we need to
iterate over each file and check permissions individually. To achieve this in a
clean manner, we must first validate the input data, then iterate over the files
and apply permission checks for each of them.

We will accomplish this by introducing a new parameter, `file_metadata`, to the
`require_permission` method. The `IfTransferType` permission generator will then
use this parameter to determine the permissions.

This change is backward compatible because the permission system accepts
`**kwargs` and ignores any unknown parameters.

### Internal fixes

- Fix the `dump_status` method in the schema. Currently, the implementation
  states:

    ```python
        def dump_status(self, obj):
            """Dump file status."""
            # due to time constraints the status check is done here
            # however, ideally this class should not need knowledge of
            # the TransferType class, it should be encapsulated at File
            # wrapper class or lower.
    ```

    To address this, we'll call the transfer class to obtain its status, then
    serialize it. In order to do this, we'll need to send a file service
    instance to the schema's context.

### Multipart Transfer Type

The pluggable transfer system will enable a new **Multipart** transfer type,
designed for uploading large files in multiple chunks.

A dedicated `Transfer` class will handle multipart uploads and will be
registered in the `TransferRegistry`.

The built-in implementation relies on the PyFS storage interface. If the
storage class is not multipart-aware, the transfer class will use the `seek`
method of the underlying PyFS to store the chunks to the appropriate place
within the uploaded file.

If the PyFS storage class is multipart-aware, it must implement the following
methods that the `Transfer` class will call:

- `def multipart_initialize_upload(self, parts, size, part_size)`
- `def multipart_set_content(self, part, stream, content_length)`
- `def multipart_commit_upload(self)`
- `def multipart_abort_upload(self)`
- `def multipart_links(self)`

## Other Examples

Another potential use case is a specialized transport for handling
sensitive data in the Crypt4GH file format before storing it in the
repository (managing encryption keys, etc.).

## How we teach this

A [file storage documentation](https://github.com/inveniosoftware/docs-invenio-rdm/blob/master/docs/reference/file_storage.md)
page will be updated to reflect these API changes.

Since this is primarily
an internal feature, detailed implementation notes will remain in this RFC
and in the API documentation.

## Drawbacks

A slightly more complicated code.

## Alternatives

The only alternative is having separate API endpoints for handling the use cases above. This RFC provides a cleaner solution.

## Unresolved questions

- [x] *Alex*: Then another less critical concern:

    Maybe there should be a way to configure different transfer types per file service...
    I could imagine e.g. not wanting to have remote files for community logos.
    But TBH, given how hidden the feature is, we could roll with having everything available
    by default across all services, and then later on adding the config if we see weird
    cases emerging

    *Mirek*: This is covered by the `IfTransferType` permission generator - each service can have its own permissions.

## Resources/Timeline

Both CESNET and Munster are willing to put their resources into the implementation and testing.
