# Support for ZIP and other container formats

- Start Date: 2025-10-01
- RFC PR: `[TBD]`
- Authors: Mirek Simek
- State: DRAFT

## Summary

This RFC proposes adding support for ZIP files and other container file formats. It includes the ability to list archive contents via API and UI, to preview files inside a ZIP archive, and to extract individual files or directories.

## Motivation

ZIP archives and other container formats (e.g., NetCDF) are commonly used to bundle multiple files for distribution and storage. Currently, Invenio does not support listing or extracting the contents of these files. Adding this functionality will enhance the user experience by allowing users to browse and access archive contents directly within the Invenio platform.

This is a feature that is commonly requested by our users.

### User stories

#### Border stones dataset

A user submitted a dataset with thousands of images of border stones packaged as a single ZIP file. The archive contains a hierarchy of directories named after the regions where the stones were found. The user wants to preview images in the UI and download individual images or directories.

#### Multidimensional data

A user submitted a multidimensional dataset packaged as a NetCDF file. The user wants to browse its logical parts and preview some of them as plots/maps.

## User interface

### Preferred approach

Ideally, the UI should show the contents of the ZIP (and other container) file in a hierarchical tree. Users can expand/collapse directories to see contained files. Each file should provide a direct download link.

Example of user interface showing the contents of a ZIP file:

![Screenshot of ZIP contents tree preview](./0099/zip_list_preview.png)

### Interim approach

As a quicker interim step, we will extend the ZIP previewer to add links for downloading files inside the ZIP. The links will point to new API endpoints for extracting files from the archive. Previewing files inside the ZIP will not be supported in this approach.

## Detailed design

### REST API

The API will be extended to support the following operations (HTTP method: GET unless otherwise noted):

- List contents: List the contents of an archive (ZIP or other supported formats). Returns a hierarchical structure of files and directories.

- Extract files: Retrieve specific files or directories from the archive. Enables downloading individual files or groups of files without downloading the entire archive.

| Endpoint | Description |
| --- | --- |
| `<record>/files/<key>/container` | List archive contents |
| `<record>/files/<key>/container/<path>` | Retrieve a file, or a directory as a ZIP stream |

### File metadata

Existing file metadata will be extended with a boolean attribute `container` to indicate if the file is a container format that can be extracted. This attribute is `true` for supported formats (e.g., ZIP, NetCDF) and `false` otherwise.

#### List operation

The list operation is a GET request to `/api/records/<pid_value>/files/<key>/container`. The user must have the `can_get_content_files` permission to call the API. The API returns a JSON response with the following structure:

```json5
{
    "entries": [
        {
            "key": "directory1",
            "title": "Directory 1", // optional
            "type": "directory",
            // other metadata fields can be added here and should be ignored by clients
            // if not recognized
            "entries": [ 
                {
                    "key": "file1.txt",
                    "title": "My text file", // optional
                    "type": "file",
                    "size": 1234, // optional
                    "checksum": "md5:abcd1234", // optional
                    "mime_type": "text/plain", // optional
                    // other metadata fields can be added here and should be ignored by clients
                    // if not recognized
                },
                // ...
            ]
        },
        // ...
    ],
    "total": 42, // total number of entries (files and directories)
    "truncated": false // true if the listing was truncated (e.g., too many entries)
}
```

Note: we do not plan to support pagination for the listing operation, as pagination over a hierarchical structure is difficult to implement. The entire structure will be returned in a single response. The extractor should return the total number of entries and a flag indicating whether the listing was truncated.

#### Extract operation

The extract operation is a GET request to `/api/records/<pid_value>/files/<key>/container/<path>`. The user must have the `can_get_content_files` permission. If `<path>` points to a file, the API streams the file content. If `<path>` points to a directory, the API streams a ZIP archive of that directory.

### Internal API

Most of the backend logic will be implemented inside the `invenio_records_resources` package.

#### `invenio_records_resources.services.files.extractors`

A new module `invenio_records_resources.services.files.extractors` will be created to handle the extraction.

#### `invenio_records_resources.services.files.extractors.base`

```python

from abc import ABC, abstractmethod
import io
from invenio_records_resources.files.api import FileRecord

class FileExtractor(ABC):
    """Base class for file extractors."""

    @abstractmethod
    def can_process(self, file_record: FileRecord) -> bool:
        """Determine if this extractor can process a given file record."""

    @abstractmethod
    def list(self, file_record: FileRecord) -> dict:
        """Return a listing of the container file contents.

        Returns a dictionary with 'entries', 'directories', 'total', and 'truncated' keys.
        """

    @abstractmethod
    def open(self, file_record: FileRecord, path: str) -> io.IOBase:
        """Open a specific file from the container file.

        Returns a file-like object for the requested entry.
        """
```

#### `invenio_records_resources.services.files.extractors.zip`

This module implements the `FileExtractor` interface for ZIP files using the `zipfile` module from the Python standard library and `zipstream` for efficient streaming.

There are two storage scenarios for the ZIP file, and each requires a different processing strategy:

1. **Locally stored files** — files stored on the local filesystem, where random access to any part of the file is cheap.

2. **Remotely stored files** — files stored on remote storage backends (e.g., S3, Google Cloud Storage) where random access is expensive because seeks require new connections.

To address both cases, a pre-processing step was added. After a file is uploaded, the `ZipProcessor` validates the ZIP file and stores the position of the table of contents (TOC) in the file metadata. This enables efficient extraction by allowing the `zipfile` module to seek directly to the correct position.

The `ZipProxy` class wraps the ZIP file access and optionally uses a `ReplyStream` to cache the TOC region in memory, reducing I/O for repeated operations.

#### Key classes and components

- **`FileExtractor`**: Base abstract class for container format extractors
- **`ZipExtractor`**: Implements `FileExtractor` for ZIP files
- **`ZipProcessor`**: Validates ZIP files during upload and caches TOC position
- **`ZipProxy`**: Manages ZIP archive stream with TOC-aware optimization
- **`ReplyStream`**: Streams data while caching specific byte ranges
- **`ContainerListResult`**: Result wrapper for container listings
- **`ContainerItemResult`**: Result wrapper for extracted container items with `send_file()` method
- **`NoExtractorFoundError`**: Exception raised when no extractor can handle a file
- **`InvalidZipException`**: Exception raised for invalid or malicious ZIP files

#### Configuration variables

The following configuration variables control ZIP processing:

- `RECORDS_RESOURCES_ZIP_FORMATS`: File extensions treated as ZIP archives (default: `[".zip"]`)
- `RECORDS_RESOURCES_ZIP_MAX_LISTING_ENTRIES`: Maximum entries returned in a single listing (default: `1000`)
- `RECORDS_RESOURCES_ZIP_MAX_ENTRIES`: Maximum total entries allowed in a ZIP (default: `10000`)
- `RECORDS_RESOURCES_ZIP_MAX_TOTAL_UNCOMPRESSED`: Maximum total uncompressed size in bytes (default: `500 * 1024 * 1024` = 500 MB)
- `RECORDS_RESOURCES_ZIP_MAX_HEADER_SIZE`: Maximum size of ZIP central directory preloaded into memory in bytes (default: `64 * 1024` = 64 KB)
- `RECORDS_RESOURCES_ZIP_MAX_RATIO`: Maximum allowed compression ratio (default: `200.0`)
- `RECORDS_RESOURCES_EXTRACTED_STREAM_CHUNK_SIZE`: Chunk size for streaming extracted content in bytes (default: `64 * 1024` = 64 KB)
- `RECORDS_RESOURCES_ARCHIVE_DOWNLOAD_MAX_SIZE`: Maximum total file size for archive download (default: `None` = no limit)
- `PREVIEWER_PREFERENCE`: List of previewers; include `"previewable_zip"` to enable ZIP container browsing
- `PREVIEWER_CONTAINER_ITEM_PREFERENCE`: List of previewers for items inside containers (excludes IIIF)

## Previewers

Currently, previewers use a low-level API to access file content. To enable previewing files inside ZIP archives in a backward-compatible way, we will add a compatibility layer that exposes archive entries as regular file-like resources.

## Unresolved questions

- As support for extractors might require both a `FileExtractor` and a `FileProcessor`, would it make sense to have a contrib module that provides both (similar to `invenio_vocabularies.contrib`)?
  - `invenio_records_resources.contrib.extractors.zip`
  - `invenio_records_resources.contrib.extractors.netcdf`
