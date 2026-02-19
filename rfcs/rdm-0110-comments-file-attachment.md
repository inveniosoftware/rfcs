# Request Comment Attachments

- Start Date: 2025-10-30
- RFC PR: [#110](https://github.com/inveniosoftware/rfcs/pull/110)
- Authors: Alex Ioannidis
- State: DRAFT

## Summary

Enable users to attach files to request comments, similar to GitHub issue comments. Users can upload files (PDFs, images, etc.) while composing comments, with images automatically embedded inline for preview. Files are uploaded to a request-scoped storage bucket and referenced via URLs in the rich HTML comment content.

## Motivation

At CERN and across research communities, reviewers and participants in request workflows need to share rich content beyond plain text:

- As a reviewer, I want to attach my annotated PDF with extensive review comments directly to my response, so that the requester can see my detailed feedback without needing external file sharing.
- As a curator, I want to upload corrected figures or diagrams to a request comment, so that the submitter can replace problematic materials with my suggestions.
- As a community member, I want to embed screenshots or images in my comments with inline preview, so that I can illustrate issues or suggestions visually without requiring recipients to download files.
- As a request participant, I want to attach supporting documentation (data files, reports, etc.) to my comments, so that all relevant materials are centralized in the request conversation.

## Detailed design

The file attachment feature adds storage capabilities to requests while keeping files as an internal implementation detail not exposed in request API responses or search indexes.

### Comment-File Association Model

Comments track attached files via an explicit `files` array in the payload. This design supports both embedded files (images in HTML) and non-embedded attachments (PDFs referenced but not linked in text).

**Storage format (database):**

```json
{
  "payload": {
    "content": "<p>See attached <img src='/requests/123/files/uuid-img.png' /></p>",
    "format": "html",
    "files": [
      {"file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"}
    ]
  }
}
```

**Key design decisions:**

1. **Reference by ID, not key**: Files referenced by `RequestFileMetadata.id` (UUID), not storage key
   - Enables future file renaming without breaking references
   - Trade-off: Renaming file changes key in HTML links (would break embedded images)
   - Can add HTML update logic later if needed

2. **URLs still use keys**: File endpoints use keys for compatibility
   - `GET /requests/{id}/files/{key}` (not ID)
   - HTML embeds by key: `<img src="/requests/123/files/uuid-filename.png" />`

3. **Expansion for responses**: `file_id` references expanded to full metadata
   - **OpenSearch indexing**: Expand during index to include filename, size, mimetype
   - **API responses**: Expand on read for single comment or search results

**Expanded format (API/OpenSearch):**
 
```json
{
  "payload": {
    "content": "...",
    "format": "html",
    "files": [
      {
        "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "key": "uuid-filename.png",
        "original_filename": "filename.png",
        "size": 12345,
        "mimetype": "image/png",
        "created": "2024-10-31T10:00:00Z"
      }
    ]
  }
}
```

4. **Explicit attachment tracking**: `files` array is independent of HTML content
   - Supports non-embedded files ("See the two PDFs attached" without links)
   - Embedded images also listed for consistent attachments display
   - No HTML parsing required

5. **Auto-delete on removal**: When file removed from comment's attachments list, delete from storage
   - **Backend deletes file automatically** in same transaction as comment update
   - **Atomic operation** - `FileCleanupComponent` ensures all-or-nothing
   - Note: If multiple comments reference same file via separate uploads, each has independent copy
   - Orphaned files from abandoned drafts remain (cleanup job can be added later)

### Programmatic API

#### Data layer

Requests will gain file storage capabilities similar to Records and Communities, using the existing `FilesField` systemfield pattern. However, unlike records where files are part of the public API, request files are purely internal for comment attachments.

**invenio_requests/requests/records/models.py**

```python
class RequestMetadata(db.Model, RecordMetadataBase):
    """Request metadata model."""

    __tablename__ = 'requests_metadata'

    # ... existing fields ...

    # Files support - nullable for lazy initialization
    bucket_id = db.Column(
        UUIDType,
        db.ForeignKey(Bucket.id, ondelete='RESTRICT'),
        nullable=True  # Only created on first file upload
    )
    bucket = db.relationship(Bucket)


class RequestFileMetadata(db.Model, RecordMetadataBase, FileRecordModelMixin):
    """Request file metadata."""

    __tablename__ = 'request_files'

    # JSON field structure: {"original_filename": "..."}
    # Comments track file associations via payload.files array (by file_id)
```

**invenio_requests/requests/records/api.py**

```python
def get_files_quota(record=None):
    """Get bucket quota configuration for request files."""
    # Returns quota_size and max_file_size from Flask config
    # with defaults (100MB total quota, 10MB max file size)


class RequestFile(FileRecord):
    """Request file API."""

    model_cls = models.RequestFileMetadata
    record_cls = LocalProxy(lambda: Request)


class Request(Record):
    """Request API."""

    # ... existing systemfields ...

    bucket_id = ModelField(dump=False)
    bucket = ModelField(dump=False)

    # Files NOT dumped or stored in JSON - internal only
    files = FilesField(
        store=False,  # Don't serialize to request JSON
        dump=False,   # Don't include in dumps()
        file_cls=RequestFile,
        delete=False,  # Manual management via service
        create=False,  # Lazy initialization
        bucket_args=get_files_quota,  # Quota config
    )
```

#### File naming and metadata

To ensure uniqueness across multiple comments and handle duplicate filenames:

- **Internal storage key**: `{uuid}-{original_filename}` (e.g., `a1b2c3d4-screenshot.png`)
- **File ID**: Each file has a unique `RequestFileMetadata.id` (UUID) for referencing
- **Original filename**: Preserved in request file JSON metadata for preservation and display purposes

```python
{
    "original_filename": "screenshot.png"
}
```

This approach allows:
- Multiple comments to have files with same original name
- Comments reference files by stable ID (survives renaming)
- Simple metadata structure focused on storage
- Files are independent entities not tied to comment lifecycle (but tracked for display)

#### Service layer

The `RequestService` exposes a files service via configuration-based wiring (similar to how permission policies and search configs are wired).

**invenio_requests/services/requests/service.py**

```python
class RequestService(RecordService):
    """Request service with file attachment support."""

    @property
    def files(self):
        """Request files service."""
        return self.config.files_service_config

    # ... existing methods ...


class RequestFilesService(FileService):
    """Service for managing request file attachments."""

    def create_file(self, identity, id_, key, stream, content_length):
        """Upload a file in a single operation (simple endpoint).

        Convenience method that combines init/upload/commit into one operation.
        """
        # Permission check, lazy bucket init, file size validation,
        # unique key generation, upload and commit

    def init_files(self, identity, id_, data):
        """Initialize multi-part file upload."""
        # For large files or exotic transfer workflows

    def set_file_content(self, identity, id_, file_key, stream, content_length):
        """Upload file content in multi-part workflow."""
        # Step 2 of multi-part upload

    def commit_file(self, identity, id_, file_key):
        """Commit file upload in multi-part workflow."""
        # Step 3 of multi-part upload - finalize file

    def read_file_metadata(self, identity, id_, file_key):
        """Retrieve file metadata."""
        # Return file metadata (key, size, mimetype, links, etc.)

    def get_file_content(self, identity, id_, file_key):
        """Retrieve file content for download/display."""
        # Return file stream

    def list_files(self, identity, id_):
        """List all files for a request."""
        # Return list of all files in request bucket

    def delete_file(self, identity, id_, file_key):
        """Delete a specific file."""
        # Called explicitly via API or by frontend when file removed from comment
```

#### File lifecycle and comment associations

**Files are tracked per comment** via the `files` array in comment payload:

**Upload:**
- Files uploaded to request before or during comment composition
- Each comment tracks its own attachments via `file_id` references

**Association:**
- Comment payload includes: `files: [{"file_id": "..."}, ...]`
- Validation on create/update ensures all referenced files exist
- Same file can be referenced by multiple comments (via separate uploads)

**Deletion:**
- **Auto-delete on removal**: When file removed from comment's `files` array (via update), automatically deleted from storage
  - **Backend handles deletion** via `FileCleanupComponent` in same transaction
  - Compares old vs new files arrays during comment update
  - **Atomic operation** - all-or-nothing guarantee
  - Prevents orphaned files when user explicitly removes attachment
- **Multiple comments**: Each comment typically uploads its own file copy, so deletions don't affect other comments
- **Manual deletion**: Files can also be deleted directly via file management API

**Orphaned files:**
- Files uploaded but never included in submitted comment (draft abandoned)
- Remain in storage until manual cleanup or periodic cleanup job
- Accepted trade-off for simplicity

See [Alternatives](#alternatives) section for other approaches considered.

### REST API

#### Simple file upload (recommended)

`PUT /api/requests/{id}/files/upload/{key}`

Uploads a file to the request's file storage in a single operation. This is a convenience endpoint that combines initialization, upload, and commit into one step. Recommended for typical file attachment use cases.

**Parameters**

| Name    | Type   | Location | Description                                |
| ------- | ------ | -------- | ------------------------------------------ |
| `id`    | string | path     | The request's identifier                   |
| `key`   | string | path     | Original filename (will be made unique)    |

**Request**

```http
PUT /api/requests/{id}/files/upload/screenshot.png HTTP/1.1
Content-Type: application/octet-stream
Content-Length: 45678

<binary file data>
```

**Response**

```http
HTTP/1.1 201 CREATED
Content-Type: application/json

{
  "id": "f9e8d7c6-b5a4-3210-9876-543210fedcba",
  "key": "a1b2c3d4-e5f6-7890-abcd-ef1234567890-screenshot.png",
  "metadata": {
    "original_filename": "screenshot.png"
  },
  "checksum": "md5:d41d8cd98f00b204e9800998ecf8427e",
  "size": 45678,
  "mimetype": "image/png",
  "links": {
    "self": "/api/requests/{id}/files/{key}",
    "content": "/api/requests/{id}/files/{key}/content",
    "commit": "/api/requests/{id}/files/{key}/commit",
    "download_html": "/requests/{id}/files/{key}"
  }
}
```

**Important**:
- **`id` field**: Frontend uses this to build `files` array in comment payload
- **URLs are relative** (no scheme+hostname) to support domain changes without breaking embedded links

**URL purposes:**
- `content`: API endpoint for programmatic file access
- `download_html`: Non-API endpoint for HTML embedding and downloads

**Frontend usage**: The `links.download_html` URL is embedded in the rich text editor:

```html
<img src="/requests/{id}/files/a1b2c3d4-...-screenshot.png"
     alt="screenshot.png"
     width="1024"
     height="768" />
```

TinyMCE detects image dimensions client-side and includes width/height attributes in the saved HTML.

**Note**: The `commit` link is used for multi-part uploads but can be ignored for simple uploads (file is already committed).

#### Multi-part file upload (for large files)

For large files or exotic transfer workflows (e.g., S3 uploads, chunked uploads), use the standard 3-step multi-part upload:

**Step 1: Initialize**
```http
POST /api/requests/{id}/files
Content-Type: application/json

[{"key": "large-file.pdf"}]
```

**Step 2: Upload content**
```http
PUT /api/requests/{id}/files/{key}
Content-Type: application/octet-stream

<binary file data>
```

**Step 3: Commit**
```http
POST /api/requests/{id}/files/{key}/commit
```

This follows the same pattern used elsewhere in the InvenioRDM REST API for file uploads.

#### Retrieve file content (HTML/Download endpoint)

`GET /requests/{id}/files/{file_key}`

Returns the file content for download or HTML embedding. This is a **non-API endpoint** (no `/api/` prefix) following the pattern used for record files (`/records/{id}/files/{filename}`).

**Parameters**

| Name       | Type   | Location | Description                   |
| ---------- | ------ | -------- | ----------------------------- |
| `id`       | string | path     | The request's identifier       |
| `file_key` | string | path     | The file's storage key         |

**Request**

```http
GET /requests/{id}/files/a1b2c3d4-...-screenshot.png HTTP/1.1
```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: image/png
Content-Disposition: attachment; filename="screenshot.png"
Content-Length: 45678

<binary file data>
```

**Important**:
- Uses `Content-Disposition: attachment` to trigger download rather than inline display
- Prevents navigation away from request page when clicking file links
- For images embedded via `<img>` tags, browsers will still display inline despite attachment header

#### Retrieve file content (API endpoint)

`GET /api/requests/{id}/files/{file_key}/content`

Programmatic API endpoint for accessing file content. Returns the same file content as the HTML endpoint above.

**Parameters**

| Name       | Type   | Location | Description                   |
| ---------- | ------ | -------- | ----------------------------- |
| `id`       | string | path     | The request's identifier       |
| `file_key` | string | path     | The file's storage key         |

**Request**

```http
GET /api/requests/{id}/files/a1b2c3d4-...-screenshot.png/content HTTP/1.1
```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: image/png
Content-Disposition: attachment; filename="screenshot.png"
Content-Length: 45678

<binary file data>
```

#### List request files

`GET /api/requests/{id}/files`

Lists all files in a request.

**Parameters**

| Name | Type   | Location | Description              |
| ---- | ------ | -------- | ------------------------ |
| `id` | string | path     | The request's identifier |

**Request**

```http
GET /api/requests/{id}/files HTTP/1.1
```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "entries": [
    {
      "key": "a1b2c3d4-...-screenshot.png",
      "metadata": {
        "original_filename": "screenshot.png"
      },
      "size": 45678,
      "mimetype": "image/png",
      "created": "2024-10-30T10:00:00Z"
    }
  ]
}
```

### Important: Files not exposed in request responses

Files are explicitly **not** included in:

- `GET /api/requests/{request_id}` responses
- `GET /api/requests` search results
- ElasticSearch request index documents
- Request JSON dumps

Files are purely internal storage for comment attachments, accessible only via dedicated file endpoints.

### URL structure and domain independence

**File URLs use relative paths** (no scheme or hostname):
- Upload response returns two URLs:
  - `content`: `/api/requests/{id}/files/{key}/content` (API endpoint)
  - `download_html`: `/requests/{id}/files/{key}` (non-API endpoint)
- Comment HTML embeds the `download_html` URL: `<img src="/requests/{id}/files/{key}" />`

This approach:
- Supports domain/hostname changes without breaking embedded links
- Works across different deployment environments (staging, production)
- Prevents hardcoded URLs in stored comment content

**File endpoint patterns:**
- Records files: `/records/{id}/files/{filename}` (non-API)
- Request files: `/requests/{id}/files/{filename}` (non-API, for HTML)
- Request files API: `/api/requests/{id}/files/{filename}/content` (API)

**Content-Disposition behavior:**
- Files served with `Content-Disposition: attachment` to trigger downloads
- Clicking file links downloads instead of navigating away from page
- Images in `<img>` tags display inline despite attachment header

### Frontend User Flows

This section documents the HTTP request sequences for key user scenarios.

#### Scenario 1: New Comment with Attachments

**User actions:**
1. User starts typing "See attached files" in rich editor
2. User selects 2 PDF files to attach
3. Frontend uploads files (both complete successfully)
4. User clicks "Submit comment"

**HTTP requests:**
```http
# Step 1: Upload first file
PUT /api/requests/123/files/upload/report.pdf
Content-Type: application/octet-stream
<binary data>

Response:
{
  "id": "abc-1234-...",
  "key": "uuid-report.pdf",
  "metadata": {"original_filename": "report.pdf"},
  "size": 234567,
  "links": {...}
}

# Step 2: Upload second file
PUT /api/requests/123/files/upload/data.pdf
Content-Type: application/octet-stream
<binary data>

Response:
{
  "id": "def-5678-...",
  "key": "uuid-data.pdf",
  "metadata": {"original_filename": "data.pdf"},
  "size": 123456,
  "links": {...}
}

# Step 3: Submit comment with file references
POST /api/requests/123/comments
Content-Type: application/json

{
  "payload": {
    "content": "See attached files",
    "format": "html",
    "files": [
      {"file_id": "abc-1234-..."},
      {"file_id": "def-5678-..."}
    ]
  }
}

Response:
{
  "id": "comment-456",
  "payload": {
    "content": "See attached files",
    "format": "html",
    "files": [
      {
        "file_id": "abc-1234-...",
        "key": "uuid-report.pdf",
        "original_filename": "report.pdf",
        "size": 234567,
        "mimetype": "application/pdf"
      },
      {
        "file_id": "def-5678-...",
        "key": "uuid-data.pdf",
        "original_filename": "data.pdf",
        "size": 123456,
        "mimetype": "application/pdf"
      }
    ]
  },
  ...
}
```

**Backend validation:** Checks that both file IDs exist in request bucket before accepting comment.

---

#### Scenario 2: Update Comment to Add Attachments

**User actions:**
1. User clicks "Edit" on existing comment (no attachments currently)
2. User modifies text to add "EDIT: forgot to attach PDF files"
3. User selects 2 PDF files to attach
4. User clicks "Update comment"

**HTTP requests:**
```http
# Step 1: Get current comment
GET /api/requests/123/comments/456

Response:
{
  "id": "comment-456",
  "payload": {
    "content": "Original text",
    "format": "html",
    "files": []  // No attachments yet
  }
}

# Step 2: Upload new files (same as Scenario 1)
PUT /api/requests/123/files/upload/file1.pdf
# Response: {id: "abc-...", ...}

PUT /api/requests/123/files/upload/file2.pdf
# Response: {id: "def-...", ...}

# Step 3: Update comment with new content and files
PUT /api/requests/123/comments/456
Content-Type: application/json

{
  "payload": {
    "content": "EDIT: forgot to attach PDF files",
    "format": "html",
    "files": [
      {"file_id": "abc-..."},
      {"file_id": "def-..."}
    ]
  }
}

Response:
{
  "id": "comment-456",
  "payload": {
    "content": "EDIT: forgot to attach PDF files",
    "files": [
      {
        "file_id": "abc-...",
        "original_filename": "file1.pdf",
        ...
      },
      {
        "file_id": "def-...",
        "original_filename": "file2.pdf",
        ...
      }
    ]
  }
}
```

**Critical requirement:** Backend must allow `payload.files` array to be updated (not just `content` and `format`).

---

#### Scenario 3: Delete Attachment from Comment

**User actions:**
1. User clicks "Edit" on comment with 2 attachments
2. User clicks "Remove" on one attachment in the UI
3. User clicks "Update comment"

**HTTP requests:**
```http
# Step 1: Get current comment
GET /api/requests/123/comments/456

Response:
{
  "payload": {
    "files": [
      {"file_id": "abc-...", "key": "uuid-file1.pdf", "original_filename": "file1.pdf", ...},
      {"file_id": "def-...", "key": "uuid-file2.pdf", "original_filename": "file2.pdf", ...}
    ]
  }
}

# Step 2: Update comment with reduced files array
PUT /api/requests/123/comments/456
Content-Type: application/json

{
  "payload": {
    "content": "...",  // Unchanged
    "format": "html",
    "files": [
      {"file_id": "def-..."}  // Only second file remains
    ]
  }
}

```

**Backend behavior (ATOMIC):**
- `FileCleanupComponent` compares old vs new files arrays
- Detects file removed from array (file_id "abc-..." no longer present)
- **Deletes file automatically in same transaction** as comment update
- **All-or-nothing**: If update fails, file NOT deleted
- **Idempotent**: Retry-safe

**Frontend logic:**
- Simply sends updated comment with new files array
- **No separate DELETE request needed!**
- Backend handles file deletion automatically

**Note:** File remains in storage until DELETE is called. If another comment references the same file (via separate upload), it has a different file_id/key, so deletion doesn't affect it.

---

#### Scenario 4: Upload Fails Mid-Composition

**User actions:**
1. User selects 3 files
2. Files 1 and 2 upload successfully, file 3 fails (network error/quota)
3. User sees error, removes failed file from UI
4. User submits comment with 2 successful files

**HTTP requests:**
```http
# File 1: Success
PUT /api/requests/123/files/upload/file1.pdf
Response: {id: "abc-...", ...}

# File 2: Success
PUT /api/requests/123/files/upload/file2.pdf
Response: {id: "def-...", ...}

# File 3: Failure
PUT /api/requests/123/files/upload/file3.pdf
Response: 413 Payload Too Large
{
  "message": "File size exceeds limit",
  "max_size": 10485760,
  "actual_size": 15000000
}

# Submit with only successful uploads
POST /api/requests/123/comments
{
  "payload": {
    "files": [
      {"file_id": "abc-..."},
      {"file_id": "def-..."}
      // file3 not included
    ]
  }
}
```

**Frontend handling:**
- Track upload state (pending/success/failed)
- Only include successful uploads in files array
- Show clear error for failed uploads
- Allow retry or removal

---

#### Scenario 5: File Deleted Between Upload and Submit

**User actions:**
1. User uploads file, gets file_id
2. Admin/other user deletes file before comment is submitted
3. User tries to submit comment

**HTTP requests:**
```http
# Step 1: Upload
PUT /api/requests/123/files/upload/report.pdf
Response: {id: "abc-...", key: "uuid-report.pdf", ...}

# Step 2: (Admin deletes file)
DELETE /api/requests/123/files/uuid-report.pdf

# Step 3: User submits
POST /api/requests/123/comments
{
  "payload": {
    "files": [{"file_id": "abc-..."}]
  }
}

Response: 400 Bad Request
{
  "errors": [{
    "field": "payload.files[0]",
    "messages": ["File abc-... not found"]
  }]
}
```

**Frontend handling:**
- Show error highlighting problematic attachment
- Allow user to re-upload or remove from comment
- Validation component should include file_id in error for identification

---

#### Scenario 6: Image Embedded in HTML

**User actions:**
1. User pastes image into rich editor
2. Frontend uploads and inserts `<img>` tag
3. Image added to files array automatically
4. User submits comment

**HTTP requests:**
```http
# Step 1: Upload image
PUT /api/requests/123/files/upload/screenshot.png
Response: {id: "abc-...", key: "uuid-screenshot.png", ...}

# Step 2: Frontend inserts HTML and tracks file
# HTML: <img src="/requests/123/files/uuid-screenshot.png" width="1024" height="768" />
# Files array: [{"file_id": "abc-..."}]

# Step 3: Submit
POST /api/requests/123/comments
{
  "payload": {
    "content": "<p>Issue screenshot:</p><img src=\"/requests/123/files/uuid-screenshot.png\" width=\"1024\" height=\"768\" />",
    "files": [{"file_id": "abc-..."}]
  }
}
```

**Result:**
- Image displays inline from HTML `<img>` tag
- Image also appears in attachments list from `files` array
- Both views show same file (not redundant - different contexts)

**Frontend logic for embedded images:**
- When image inserted in editor, automatically add to files array
- When image removed from editor, keep in files array unless user explicitly removes from attachments list
- Clear separation: HTML content vs. explicit attachments

---

### Frontend integration

The frontend workflow follows the GitHub-style upload experience:

1. **User initiates upload** (drag/drop, paste image, or file picker in TinyMCE)
2. **Insert placeholder** in editor at cursor position:
   ```html
   <!-- Uploading "screenshot.png"... -->
   ```
3. **Upload file** via `PUT /api/requests/{id}/files/upload/screenshot.png` with bytestream
4. **Receive response** with `download_html` URL
5. **TinyMCE detects image dimensions** client-side
6. **Replace placeholder** with actual content including dimensions:
   ```html
   <img src="/requests/{id}/files/{file_key}"
        alt="screenshot.png"
        width="1024"
        height="768" />
   ```

**Image dimension handling:**
- Backend does NOT compute or return image dimensions
- TinyMCE loads the image client-side to detect actual dimensions
- Width/height attributes are included in the saved HTML for layout stability
- Prevents content reflow when comment is viewed later

7. **User continues editing** or submits comment
8. **On comment submission**:
   - Frontend includes uploaded file_id in `payload.files` array
   - Backend validates all file_id references exist
   - Comment stored with file associations

### File lifecycle

**Storage:**
- Files stored in request-scoped bucket via `PUT /api/requests/{id}/files/upload/{key}`
- Tracked in `request_files` table with unique keys: `{uuid}-{original_filename}`
- Referenced by comments via `payload.files` array (by file_id)

**Association with comments:**
- Comments track attachments via `files: [{"file_id": "..."}, ...]` in payload
- Backend validates file_id references on comment create/update
- Multiple comments can reference files (via separate uploads)

**Deletion:**
- **Auto-delete on removal**: When file removed from comment's `files` array, frontend deletes from storage
  - Frontend compares old/new files arrays after edit
  - Sends `DELETE /api/requests/{id}/files/{key}` for removed files
- **Manual deletion**: Via API endpoint (requires `can_manage_files` permission)

**Orphaned files:**
- Files uploaded but never included in submitted comment (abandoned drafts)
- Remain in storage until manual cleanup or periodic cleanup job
- Future enhancement: automated cleanup for old orphaned files

### Permissions

File access follows request permissions:

- **Upload/Manage files** (`can_manage_files`): Same as creating comments on the request
- **View/Download files** (`can_read_files`): Same as viewing the request and its timeline
- **Delete files**: Via `DELETE /api/requests/{id}/files/{key}` endpoint (requires `can_manage_files`)

## Example

### Complete workflow: Reviewer attaches annotated PDF

1. **Reviewer opens request and starts writing a comment**

2. **Reviewer uploads annotated PDF:**

```http
PUT /api/requests/abc-123/files/upload/reviewed_manuscript.pdf HTTP/1.1
Content-Type: application/octet-stream
Content-Length: 123456

<reviewed_manuscript.pdf binary data>
```

Response:
```json
{
  "id": "f1e2d3c4-e5f6-7890-abcd-ef1234567890",
  "key": "uuid-abc123-reviewed_manuscript.pdf",
  "metadata": {
    "original_filename": "reviewed_manuscript.pdf"
  },
  "links": {
    "content": "/api/requests/abc-123/files/uuid-abc123-reviewed_manuscript.pdf/content",
    "download_html": "/requests/abc-123/files/uuid-abc123-reviewed_manuscript.pdf"
  }
}
```

3. **Frontend updates comment draft** with file link using `download_html`:

```html
<p>I've completed my review. Please see my detailed comments in the attached PDF:</p>
<p><a href="/requests/abc-123/files/f1e2d3c4-...-reviewed_manuscript.pdf">
  reviewed_manuscript.pdf
</a></p>
```

When user clicks the link, the file downloads (due to `Content-Disposition: attachment` header) rather than navigating away from the page.

4. **Reviewer also pastes a screenshot** showing an issue:

```http
PUT /api/requests/abc-123/files/upload/figure-issue.png HTTP/1.1
Content-Type: application/octet-stream
Content-Length: 234567

<figure-issue.png binary data>
```

Response:
```json
{
  "id": "a2b3c4d5-e6f7-8901-bcde-f234567890ab",
  "key": "uuid-xyz789-figure-issue.png",
  "links": {
    "download_html": "/requests/abc-123/files/uuid-xyz789-figure-issue.png"
  }
}
```

Frontend loads image, detects dimensions, and embeds with size attributes:
```html
<p>Additionally, Figure 3 has a rendering issue:</p>
<img src="/requests/abc-123/files/uuid-xyz789-figure-issue.png"
     alt="figure-issue.png"
     width="1920"
     height="1080" />
```

Note: TinyMCE detects actual image dimensions (1920x1080) and includes them in the HTML for layout stability.

5. **Reviewer submits comment with file references:**

```http
POST /api/requests/abc-123/comments HTTP/1.1
Content-Type: application/json

{
  "payload": {
    "content": "<p>I've completed my review...</p><p><a href=\"...\">reviewed_manuscript.pdf</a></p><p>Additionally, Figure 3 has a rendering issue:</p><img src=\"...\" />",
    "format": "html",
    "files": [
      {"file_id": "f1e2d3c4-e5f6-7890-abcd-ef1234567890"},
      {"file_id": "a2b3c4d5-e6f7-8901-bcde-f234567890ab"}
    ]
  }
}
```

Backend:
- Validates both file IDs exist in request bucket
- Stores comment with file associations
- Expands file metadata in response

6. **Later, if reviewer edits to remove PDF attachment:**

- Frontend detects PDF file (id: `f1e2d3c4-...`) removed from `files` array
- After comment update succeeds, frontend sends: `DELETE /api/requests/abc-123/files/uuid-abc123-reviewed_manuscript.pdf`
- Image file remains (still in files array with id: `a2b3c4d5-...`)

### Request JSON structure (files not included)

```http
GET /api/requests/abc-123 HTTP/1.1
```

```json
{
  "id": "abc-123",
  "title": "Manuscript Review Request",
  "status": "submitted",
  "created_by": {"user": "1"},
  "receiver": {"user": "2"},
  "links": {
    "self": "{scheme+hostname}/api/requests/abc-123",
    "timeline": "{scheme+hostname}/api/requests/abc-123/timeline"
  }
  // Note: NO "files" field - files are internal only
}
```

## How we teach this

### User documentation

- **User guide section**: "Attaching Files to Request Comments"
  - How to drag/drop or paste images
  - How to attach non-image files (PDFs, etc.)
  - Supported file types and size limits
  - What happens to files when comments are deleted

### API documentation

- **REST API reference**: Document file upload endpoints
  - Request/response formats
  - File URL structure for embedding
  - Permission requirements

### Developer documentation

- **Architecture guide**: File storage in requests
  - FilesField configuration and lazy initialization
  - File lifecycle and cleanup mechanisms
  - How to extend for other attachment types

This feature follows existing InvenioRDM patterns (FilesField systemfield, quota configuration via bucket_args, configuration-based service composition) making it familiar to developers already working with Records with files.

## Drawbacks

**Storage growth**: File attachments will increase storage requirements. Mitigation:
- Implement file size limits per upload and quotas per request
- Monitor storage usage and provide admin tools

**Orphaned files from abandoned drafts**: Files uploaded but never included in submitted comments waste storage. Mitigation:
- Implement periodic cleanup job for old unattached files
- Acceptable trade-off for simpler implementation

**Transaction coupling**: File deletion happens in same transaction as comment update, increasing transaction complexity. Benefits:
- **Atomic operation** ensures consistency
- **No orphaned files** from failed deletions
- **Simpler frontend** - no deletion logic needed
- Trade-off: Slightly longer transaction time acceptable

**No recovery from accidental file deletion**: Once file deleted from storage (via comment update), it's gone. Mitigation:
- Implement soft-delete with grace period (future enhancement)
- Admin tools to recover recently deleted files
- Transaction rollback prevents deletion if update fails

**File renaming breaks HTML links**: Renaming a file changes its key, breaking embedded image URLs. Mitigation:
- Accept limitation initially (renaming is rare)
- Can add HTML update logic later if needed
- File metadata expansion uses file_id, so attachments list remains correct

## Alternatives

### Automatic file cleanup on comment deletion

**Description:** Automatically delete files when comments referencing them are deleted. This requires:

- **HTML parsing component** to extract file references from comment content
- **Comment-file association tracking** in file metadata (`comment_id` field)
- **Validation component** to verify file references exist when comment is submitted
- **Cleanup component** triggered on comment deletion to remove associated files
- **Celery tasks** for async file deletion

**Implementation approach that was rejected:**

```python
class RequestFileCleanupComponent(ServiceComponent):
    """Component for automatic file cleanup on comment deletion."""

    def delete_comment(self, identity, request, comment):
        # Parse comment HTML to extract file URLs
        # Schedule Celery task to delete referenced files

class RequestCommentValidationComponent(ServiceComponent):
    """Validate comment file references on submission."""

    def create_comment(self, identity, request, data):
        # Extract file keys from HTML
        # Verify all referenced files exist
        # Update file metadata with comment_id
```

**Why rejected:**
- **Complexity**: HTML parsing is fragile and error-prone
- **Shared files**: Multiple comments can reference the same file
- **Comment edits**: Tracking file references across edits is complex
- **Edge cases**: Many scenarios where cleanup could fail or behave unexpectedly
- **Implementation burden**: ~600 lines of code for cleanup/validation logic

**Chosen approach:** Files tracked per comment via explicit `files` array. Simpler than HTML parsing, supports non-embedded attachments, enables validation.
**Trade-off:** Orphaned files from abandoned drafts may accumulate. Can be addressed later with periodic cleanup job if needed.

### Alternative: File references by key instead of ID

Store file keys directly in comment payload instead of file IDs:

```json
{
  "payload": {
    "content": "<p>See attached PDF...</p>",
    "format": "html",
    "files": [{"key": "uuid-filename.pdf"}]
  }
}
```

**Pros**: Simpler lookup (key is storage identifier), no need to query file metadata table
**Cons**: File renaming not possible (key is part of reference), no stable identifier if file moved

**Why we chose file_id references:**
- Enables future file renaming without breaking comment references
- Stable identifier decoupled from storage key
- Allows metadata evolution (can add fields without changing references)
- Trade-off: Renaming breaks HTML-embedded links (acceptable, can add HTML update logic later)

## Unresolved questions

**Orphaned file cleanup timing**: When should files uploaded but never included in submitted comments be cleaned up?
- Suggested: Periodic cleanup job for files older than X days with no comment association
- Implementation: Future enhancement, not in initial release

**Bulk operations**: Support for downloading all request attachments as ZIP?
- Useful for archival purposes (and there's a use-case already)
- Future enhancement?

**Image dimension handling**:
- Decision: TinyMCE determines dimensions client-side and includes in HTML
- Images loaded in browser to detect actual width/height
- Dimensions saved in comment HTML (`<img width="..." height="...">`)
- Provides layout stability without backend image processing

## Resources/Timeline

**Phase 1: Backend file storage**
- Data model extension (Request, RequestFile models)
- FilesField integration with lazy initialization
- File upload/retrieve service methods
- REST API endpoints for file operations

**Phase 2: Comment-file association**
- Add `files` field to comment payload schema
- File validation component (check file_id references exist)
- File expansion for API responses and OpenSearch
- Allow files array in comment updates
- Service integration tests

**Phase 3: Frontend integration**
- File upload on paste/drag-drop
- Add uploaded files to comment payload
- Display attachments list in timeline
- Auto-delete removed files (compare old/new files arrays)
