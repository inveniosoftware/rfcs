- Start Date: 2024-03-22
- RFC PR: [#<PR>](https://github.com/inveniosoftware/rfcs/pull/<PR>)
- Authors: Mirek Simek <miroslav.simek@cesnet.cz>, Miroslav Bauer <bauer@cesnet.cz>

# Pluggable Transfer Types for Record Files

## Summary

Currently, the system only supports three fixed transfer types—**Local**, **Fetch**,
and **Remote**—and does not allow adding new ones. By making transfer types pluggable,
users could implement and integrate their own custom transfers.

To fully support pluggable transfer types on the front end, the file uploader UI must also be customizable.
We propose integrating a custom file uploader UI based on the Uppy.io library, utilizing the AWS S3 Multipart uploader
plugin.

## Motivation

We plan to introduce a built-in **Multipart** transfer type that enables
uploading large files in separate chunks, improving performance and
reliability.

In addition, one could, for example, develop an **S3Copy** transfer type to
copy files directly between S3 buckets—bypassing the Invenio server—to reduce
bandwidth usageand improve efficiency.

As a record depositor, I want to:
- Select one or multiple local files to upload and assign to a record.
- Select one or multiple remote files (e.g., import from a parent record or cloud storage services).
- View the upload progress of each ongoing file upload.
- Abort an ongoing upload.
- Retry any failed uploads.
- View and manage already uploaded files (e.g., remove files, update associated metadata such as image captions and descriptions).
- Verify file integrity using checksums when uploading (to ensure the file was received and stored without errors) or downloading (to confirm I received an exact copy as stored on the server).
- Efficiently and reliably upload large files (>10 GiB) even over unreliable networks.

As a developer, I want to:
- Implement or integrate custom file uploader UIs (including custom upload logic) into the deposit form (e.g., using library like Uppy.io), allowing the front-end to support any configured transfer types.

As an admin, I want to:
- Configure the default upload mechanism (TransferType) for new uploads.
- Configure which uploader UI should be used in the deposit form.
- Define upload restrictions (e.g., allowed MIME types, file size limits).

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

## Front-end layer

### **Current implementation**  

Currently, most of the file upload UI is part of the [`invenio-rdm-records` deposit](https://github.com/inveniosoftware/invenio-rdm-records/tree/master/invenio_rdm_records/assets/semantic-ui/js/invenio_rdm_records/src/deposit) package.  

Record depositors interact directly with the [FileUploader](https://github.com/inveniosoftware/invenio-rdm-records/blob/master/invenio_rdm_records/assets/semantic-ui/js/invenio_rdm_records/src/deposit/fields/FileUploader/index.js) field, which relies heavily on the **Files Redux store** to manage file upload state. Currently, the only way for developers to customize this field is by overriding components using `react-overridable` IDs, but they cannot modify the logic governing how files are uploaded.

The [Files Redux store actions](https://github.com/inveniosoftware/invenio-rdm-records/blob/master/invenio_rdm_records/assets/semantic-ui/js/invenio_rdm_records/src/deposit/state/actions/files.js) depend on an instance of the [DepositFilesService](https://github.com/inveniosoftware/invenio-rdm-records/blob/master/invenio_rdm_records/assets/semantic-ui/js/invenio_rdm_records/src/deposit/api/DepositFilesService.js) service interface. This service is passed as a prop to the [DepositFormApp](https://github.com/inveniosoftware/invenio-rdm-records/blob/master/invenio_rdm_records/assets/semantic-ui/js/invenio_rdm_records/src/deposit/api/DepositFormApp.js#L117) component.  

For `invenio-app-rdm`, [there is no way to change](https://github.com/inveniosoftware/invenio-app-rdm/blob/master/invenio_app_rdm/theme/assets/semantic-ui/js/invenio_app_rdm/deposit/RDMDepositForm.js#L114-L120) the default service, which is [`RDMDepositFilesService`](https://github.com/inveniosoftware/invenio-rdm-records/blob/master/invenio_rdm_records/assets/semantic-ui/js/invenio_rdm_records/src/deposit/api/DepositFormApp.js#L62).  

The [DepositFilesService](https://github.com/inveniosoftware/invenio-rdm-records/blob/master/invenio_rdm_records/assets/semantic-ui/js/invenio_rdm_records/src/deposit/api/DepositFilesService.js) depends on the [DepositApiClient](https://github.com/inveniosoftware/invenio-rdm-records/blob/master/invenio_rdm_records/assets/semantic-ui/js/invenio_rdm_records/src/deposit/api/DepositApiClient.js) to interact with Invenio’s file REST APIs. This API client is configured similarly to `DepositFilesService`, and in `invenio-app-rdm`, it always defaults to [`RDMDepositApiClient`](https://github.com/inveniosoftware/invenio-rdm-records/blob/master/invenio_rdm_records/assets/semantic-ui/js/invenio_rdm_records/src/deposit/api/DepositApiClient.js), which also cannot be changed.  

To switch to a different file upload form field implementation, a mechanism is needed to modify all of the above dependencies. We decided to make the new uploader UI a feature toggle, replacing the current FileUploader UI.
Site admins could enable it via app config (e.g. `invenio.cfg`).

### Proposed changes

#### Feature flag

We introduce the folowing feature flag to replace the current deposit form file uploader UI with the new one:
```python
# invenio.cfg
APP_RDM_DEPOSIT_NG_FILES_UI = True
```

#### Presentation/resources layer

To let the uploader UI implementation know about the TransferTypes available on the back-end, we need to
pass the information down to the `deposits-config` HTML input by extending the deposit's [get_form_config](https://github.com/inveniosoftware/invenio-app-rdm/blob/master/invenio_app_rdm/records_ui/views/deposits.py#L388) with:
```python
def get_form_config(**kwargs):
    """Get the react form configuration."""
   return dict(
    #    ...
+       default_transfer_type=current_transfer_registry.default_transfer_type,
+       transfer_types=list(current_transfer_registry.get_transfer_types()),
        **kwargs,
    )
```

Additionally, we pass the feature flag value to the deposit app template, using hidden input:
```jinja
<!-- deposit.html -->
  <input type="hidden" name="deposits-use-uppy-ui"
          value='{{ config.APP_RDM_DEPOSIT_NG_FILES_UI | tojson }}'>
```

It is then passed down to RDM deposit form app as the `useUppy` prop.

#### UppyDepositFilesService

Current RDM's files service has a concept of managing the upload queue and tracking upload progress.
For Uppy, this is all managed by its internal state and not needed here. We propose additional
implementation overriding the `RDMDepositFilesService` where needed.

### Files API client extension

To support new transfer types and new uploader UI, `RDMDepositFileApiClient` interface needs to be extended
by following methods:

- `initializeFileUpload(initializeUploadUrl, filename, **transferOptions**) {` here we added `transferOptions` to properly initialize upload with transferType
- `getUploadParams(fileContentUrl, file, options)` this is needed for cases when the uploader UI needs to make the upload request on its own (e.g. using XHR) not using
file API client, thus its Axios instance. For Uppy, this is used by `AWSS3Multipart` plugin code for making a single-part file upload.

### Files Redux store changes

We propose to:

- extract `saveAndFetchDraft` from `uploadFiles` as a separate Redux action (so that uploader UI could dispatch it independently where needed)
- introduce `initializeFileUpload`, `finalizeUpload` and `getUploadParams` actions to match the new workflow introduced to RDM files service

#### Deposit form apps

- In `DepositFormApp` component we [toggle between files service implementations](https://github.com/oarepo/invenio-rdm-records/pull/1/files)
  depending on the `useUppy` prop.

- Wherever we rendered the original `FileUploader` React component in the `RDMDepositForm` component, we now
[toggle between implementations](https://github.com/inveniosoftware/invenio-app-rdm/pull/2994/files#diff-b5e1b966b44dd2d7d0fc9688d8243e176d2a3c1242200e495b2bb4598d60be4bR173-R191)
 depending on the `useUppy` prop. Overriding mechanism using `react-overriable` isn't affected at this part.

### Custom Files UI with Uppy

Here we describe integration of [Uppy.io](https://uppy.io/) as a custom Files UI implementation addressing
use cases regarding reliable multi-part uploads.

<!-- TODO: move under inveniosoftware org? -->
Uppy integration provides `UppyFileUploader` field component that acts as a drop-in replacement of original `FileUploader` component.
It is based based on [Uppy Dashboard](https://uppy.io/docs/dashboard/) UI. For the multipart uploading mechanism, it uses and extends
[AWS S3 Multipart](https://www.npmjs.com/package/@uppy/aws-s3-multipart) package, adapted to match REST APIs designed by this RFC and already present in InvenioRDM.

Drag & drop file input with auto-thumbnails, upload progress tracking, upload abortion & retry and status bar is provided out of the box. Files Redux store actions is still being used to manage record files state.

More features & integrations like the remote file sources, image or file metadata editor, could be potentially added to the Dashboard UI later.

#### UI workflow

Default dashboard state:
![Screenshot 2025-02-13 at 8 57 20](https://github.com/user-attachments/assets/85319625-7510-4482-b314-64eac835fc73)

Files chosen for upload:
![Screenshot 2025-02-13 at 9 07 20](https://github.com/user-attachments/assets/eb1810d5-fd27-4612-9de7-ad028c78c1ca)

Multipart upload in progress:
![Screenshot 2025-02-13 at 9 07 38](https://github.com/user-attachments/assets/c7be5987-7848-4322-9baf-af8cd9a28e9b)

Upload complete:
![Screenshot 2025-02-13 at 9 07 55](https://github.com/user-attachments/assets/a5820012-a16a-4270-97ee-82d1392ad948)

Add one more file that fails to upload:
![Screenshot 2025-02-13 at 9 08 58](https://github.com/user-attachments/assets/252826f3-fb66-44b2-a211-4a72bb0d2564)

## Other Examples

Another potential use case is a specialized transport for handling
sensitive data in the Crypt4GH file format before storing it in the
repository (managing encryption keys, etc.).

## How we teach this

A [file storage documentation](https://github.com/inveniosoftware/docs-invenio-rdm/blob/master/docs/reference/file_storage.md)
page will be updated to reflect these API changes.

Back-end part is primarily
an internal feature, detailed implementation notes will remain in this RFC
and in the API documentation.

On front-end:
- For site admins, we need to document the `APP_RDM_DEPOSIT_NG_FILES_UI` config option.

## Drawbacks

- A slightly more complicated code.

## Alternatives

Having separate API endpoints for handling the use cases above. This RFC provides a cleaner solution.

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
