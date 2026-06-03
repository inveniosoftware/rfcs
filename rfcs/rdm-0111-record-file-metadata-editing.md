- Start Date: 2026-04-22
- RFC PR: [#](https://github.com/inveniosoftware/rfcs/pull/)
- Authors: Martin Čorovčák, Miroslav Bauer (CESNET)

# Record File Metadata Editing

## Summary

Enable users to edit metadata associated with draft files **after** upload via the deposit UI. InvenioRDM already stores per-file metadata in `FileRecord.metadata` (a JSON `DictField`), and the REST API already supports reading (`GET`) and updating (`PUT`) file metadata. What is missing is:

1. **A user-facing UI** for editing file metadata in the deposit form.
2. **Optimistic concurrency control** (`ETag`/`If-Match`) to prevent silent overwrites when multiple users edit the same file concurrently.

This RFC proposes a modal-based "Edit metadata" flow in the deposit form and adds `ETag`/`If-Match` support to the existing file metadata `PUT` endpoint.

The scope of metadata fields — whether fixed, community-driven, or fully extensible — is intentionally left as an open question for community discussion. If demand for diverse, instance-specific metadata fields is high, the implementation could leverage InvenioRDM's existing [Custom Fields](#custom-fields) framework rather than hardcoding a fixed schema.

## Motivation

Research datasets often contain many files, and per-file metadata is essential for discoverability, citation, and reuse. Users need to describe individual files (e.g., "raw survey data", "processed analysis script", "supplementary figure") separately from the record-level metadata.

- As a **depositor**, I want to add a human-readable title and description to each uploaded file, so that consumers understand what each file contains without opening it.
- As a **curator**, I want to edit file metadata after the initial upload, so that I can correct or enrich descriptions during the review workflow.
- As a **deposit UI user**, I want the metadata editing to happen **after** the file is already in the file list (not during upload), so that I can see what I'm annotating.

Note: File type and other system-generated metadata (e.g., image `width`/`height` computed by a background task after upload) are **not** user-editable. File type is derived from MIME type. System annotations are computed post-upload. Only explicitly user-provided fields (`title`, `description`, plus any custom fields) are editable.

## Detailed design

### Core interaction flow

Full sequence diagram: [Excalidraw](https://excalidraw.com/#room=84d7492dda5c27ce95ac,XO3BMGpLJJOkz-QMvm-Vkw)

The proposed UX follows a two-phase pattern: first upload, then annotate.

1. **Upload phase**: User drags/drops or selects files via Uppy. Files upload as today. No metadata is collected during upload.
2. **Review phase**: The file list table in the deposit form shows each uploaded file with an action column.
3. **Edit trigger**: Each row includes a pencil icon button ("Edit metadata").
4. **Modal form**: Clicking the button opens a modal dialog containing a form. The form is pre-filled with the file's current metadata fetched from the backend.
5. **Save**: User edits fields and clicks "Save". A `PUT` request is sent to the files API with an `If-Match` header.
6. **Conflict handling**: If another session changed the file metadata in the meantime (HTTP `412 Precondition Failed`), the frontend fetches fresh metadata, merges the user's input on top of it, and retries the save automatically. The user is informed that a concurrent edit occurred.

### Concurrency control protocol

File metadata updates use [optimistic concurrency control](https://en.wikipedia.org/wiki/Optimistic_concurrency_control) via HTTP `ETag`/`If-Match`.

**Current state**

`GET /api/records/{id}/draft/files/{key}` already returns file metadata JSON:

```http
GET /api/records/{id}/draft/files/dataset.csv
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "key": "dataset.csv",
  "updated": "2025-04-22T10:00:00Z",
  "metadata": {
    "title": "Raw survey data",
    "description": "Participant responses..."
  },
  ...
}
```

However, the current response does **not** include an `ETag` header, and `PUT` on the same endpoint does **not** accept `If-Match`. This RFC proposes adding both.

**Proposed: `ETag` on GET**

Add `ETag` header to the response. The value is a metadata revision identifier (e.g., hash of `updated` timestamp or a `revision_id` counter).

```http
HTTP/1.1 200 OK
ETag: "123"
Content-Type: application/json
```

**Proposed: `If-Match` on PUT**

The existing `PUT` endpoint (`FileResource.update` → `FileService.update_file_metadata`) is extended to accept `If-Match`:

```http
PUT /api/records/{id}/draft/files/dataset.csv
Content-Type: application/json
If-Match: "123"

{
  "metadata": {
    "title": "Updated title",
    "description": "Updated description"
  }
}
```

**Success (no conflict)**

```http
HTTP/1.1 200 OK
ETag: "124"
```

**Conflict detected**

```http
HTTP/1.1 412 Precondition Failed
```

**Retry with merge**

On `412`, the frontend:

1. Fetches current metadata (`GET /api/records/{id}/draft/files/{key}`).
2. Obtains new `ETag`.
3. Applies user's form input as patch on fresh metadata:
   - Scalar fields (title, description): user input overwrites.
   - Structured fields (lists, objects): shallow merge — user-provided keys overwrite; absent keys are preserved from server.
4. Re-submits `PUT` with new `If-Match`.
5. Repeat up to 3 retries, then show error.

Retry loop is transparent unless max retries exceed.

### Metadata schema — open question

The exact set of metadata fields is **not yet decided** and requires community input. Two approaches are proposed:

#### Approach A: Fixed fields (recommended if communities converge on common needs)

A small, well-defined set of universally useful fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | no | Human-readable file title (defaults to filename) |
| `description` | string | no | Free-text description of file contents |

Note: `type` is intentionally excluded. File type is system-derived from MIME type. System-generated metadata (e.g., image `width`/`height` computed by a background task after upload) is also excluded. Both are stored in file metadata but are not user-editable.

This approach is simpler to implement, validate, index, and translate. It aligns with DataCite and Zenodo patterns.

#### Approach B: Custom fields (recommended if diverse, instance-specific needs dominate)

File metadata fields are driven by InvenioRDM's [Custom Fields](../rdm-0064-custom-fields.md) framework. Instance administrators configure `RDM_FILE_METADATA_CUSTOM_FIELDS` and `RDM_FILE_METADATA_CUSTOM_FIELDS_UI`, declaring domain-specific fields (e.g., CERN-specific parameters, discipline-specific provenance).

**Pros**:
- Instance administrators control the schema without code changes.
- Supports complex field types (controlled vocabularies, dates, booleans).

**Cons**:
- Higher complexity in UI rendering (dynamic form generation).
- Indexing and faceting require additional mapping management.
- Translation and validation surface area grows.

### Decision trigger for community discussion

The community should evaluate whether the majority of use cases can be satisfied by the small fixed set in **Approach A**. If significant use cases require instance-specific fields (e.g., "experiment run number", "detector channel", "software version"), then **Approach B** should be adopted from the start — or as a secondary phase.

### Existing file metadata schema

FileRecord already has a `metadata` DictField (`invenio_records_resources.records.api`). Current `MetadataSchema` (`invenio_rdm_records.services.schemas.files`) defines:

| Field | Type | Editable | Source |
|-------|------|----------|--------|
| `page` | integer | no | System/computed |
| `type` | string | no | System-derived from MIME type |
| `language` | string | no | System/computed |
| `encoding` | string | no | System/computed |
| `charset` | string | no | System/computed |
| `previewer` | string | no | System/computed |
| `width` | integer | no | System (post-upload background task for images) |
| `height` | integer | no | System (post-upload background task for images) |

All current fields are system-generated. This RFC proposes adding user-editable and potentially configurable/custom (Approach B) fields to the schema.

### Backend changes

#### API endpoints (already exist)

The endpoints for reading and updating file metadata already exist:

```
GET  /api/records/{id}/draft/files/{key}   <- returns file metadata JSON
PUT  /api/records/{id}/draft/files/{key}   <- updates file metadata via update_file_metadata()
```

What is **missing**: `ETag`/`If-Match` on file responses.

#### Adding ETag to GET responses

```http
GET /api/records/{id}/draft/files/dataset.csv
```

```http
HTTP/1.1 200 OK
ETag: "123"
Content-Type: application/json
```

The `ETag` value is the file record's `revision_id` (record version counter), same pattern as record-level endpoints. This requires wiring `etag_headers` into `FileResourceConfig.response_handlers`.

#### Adding If-Match to PUT

The existing `FileService.update_file_metadata` signature:

```python
def update_file_metadata(self, identity, id_, file_key, data, uow=None)
```

Needs `revision_id` parameter:

```python
def update_file_metadata(self, identity, id_, file_key, data, revision_id=None, uow=None)
```

And `check_revision_id` call before update, same pattern as `RecordService.update()`.

#### ETag semantics

`ETag` reflects file metadata changes (not binary content). The `revision_id` from `RecordMetadataBase` serves this purpose — it increments on every DB update to the file record JSON.

#### Validation

- If **Approach A** (fixed fields): validate using a Marshmallow schema (or Pydantic, per project conventions). Unknown keys are rejected (`400 Bad Request`).
- If **Approach B** (custom fields): validate using the existing custom fields validation pipeline.

#### Serialization in responses

File metadata is included in the existing `GET /api/records/{id}/draft/files/{filename}` and `GET /api/records/{id}/draft/files` responses under the `metadata` key.

```json
{
  "entries": [
    {
      "key": "dataset.csv",
      "size": 12345,
      "mimetype": "text/csv",
      "metadata": {
        "title": "Raw survey data",
        "description": "Participant responses..."
      }
    }
  ]
}
```

### Frontend changes

#### File list table — Edit metadata button

In the deposit form's file list component (`FileUploader`/`FileUploaderArea`), add a new action button to each file row:

```jsx
<Button
  icon="pencil"
  size="mini"
  title={i18next.t("Edit file metadata")}
  onClick={() => openMetadataEditor(file)}
/>
```

#### Metadata edit modal

A reusable modal component (`FileMetadataEditorModal`) that renders a form based on the chosen schema:

- For **Approach A**: hardcoded form with hardcoded metadata input fields.
- For **Approach B**: dynamic form rendered using the custom fields widget pipeline.

```jsx
<FileMetadataEditorModal
  file={selectedFile}
  onSave={handleSaveMetadata}
  onClose={closeModal}
/>
```

**State machine**:

| State | UI |
|-------|-----|
| `idle` | Form pre-filled with fetched metadata |
| `saving` | Save button shows spinner; form read-only |
| `conflict` | Banner: "Metadata was updated by another session. Your changes were merged." (transparent retry) |
| `error` | Inline field errors or global error message |

#### PUT with `If-Match`

The save handler wraps the HTTP client to include `If-Match`:

```js
async function saveFileMetadata(recordId, filename, metadata, etag) {
  try {
    return await http.put(
      `/api/records/${recordId}/draft/files/${filename}`,
      { metadata },
      { headers: { 'If-Match': etag } }
    );
  } catch (error) {
    if (error.response.status === 412) {
      // Conflict: fetch fresh, merge, retry
      const fresh = await http.get(`/api/records/${recordId}/draft/files/${filename}`);
      const mergedMetadata = mergeMetadata(fresh.data.metadata, metadata);
      return saveFileMetadata(recordId, filename, mergedMetadata, fresh.headers.etag);
    }
    throw error;
  }
}
```

**Merge strategy** (`mergeMetadata`):

```js
function mergeMetadata(serverMetadata, userMetadata) {
  return {
    ...serverMetadata,        // preserve server fields user didn't touch
    ...userMetadata,          // overwrite with user's explicit changes
  };
}
```

For list-type custom fields, a deeper merge strategy may be needed; this is an implementation detail.

### Deprecating the PR #2288 approach

This RFC supersedes the metadata-during-upload approach proposed in [invenio-rdm-records#2288](https://github.com/inveniosoftware/invenio-rdm-records/pull/2288). That PR added metadata fields to the Uppy Dashboard UI, collecting metadata **before** upload. The community feedback was that:

1. Users want to see the file in the list before deciding what metadata it needs.
2. Uppy's metadata editor is ephemeral (lost on page refresh) and does not integrate well with the record's persistent file metadata.
3. The edit-after-upload pattern mirrors GitHub's file browser, Dataverse, and other repository platforms, making it more intuitive.

Therefore, PR #2288's Uppy metadata integration is **not recommended** for the InvenioRDM core. Integrators who need Uppy-side metadata collection can still use Uppy's `meta` feature independently; this RFC focuses on the persistent, backend-backed metadata editing flow.

## Example

### Scenario: Depositor edits a file description after upload

1. Alice uploads `wave1_responses.csv` via the deposit form. The file appears in the file list table with only its filename.
2. Alice clicks the pencil icon next to the file. A modal opens showing:
   - Title: `wave1_responses.csv` (auto-populated from filename)
   - Description: *(empty)*
3. Alice fills:
   - Title: `"Wave 1 Survey Responses"`
   - Description: `"CSV export of Qualtrics survey responses from October 2024 (n=1,234)."`
4. Alice clicks "Save". The frontend sends:
   ```http
   PUT /api/records/{id}/draft/files/wave1_responses.csv
   If-Match: "1"
   { "metadata": { "title": "...", "description": "..." } }
   ```
5. Backend returns `200 OK` with new `ETag: "2"`. The modal closes and the file list updates to show the title.

### Scenario: Concurrent edit

1. Alice opens the metadata editor for `wave1_responses.csv`.
2. Bob (a co-author) edits the same file and changes the title while Alice is still typing her description.
3. Alice clicks "Save". The frontend sends with Alice's stale `ETag`.
4. Backend returns `412 Precondition Failed`.
5. Frontend fetches fresh metadata (now showing Bob's title change), merges Alice's input on top:
   - Bob changed `title` to `"Final Survey Data"`; Alice left `title` unchanged in her form → keep `"Final Survey Data"`.
   - Alice edited `description`; Bob left it unchanged → use Alice's value.
6. Frontend retries the `PUT` with the new `ETag`. Save succeeds.
7. Alice sees a brief non-blocking banner: "Your changes were saved. Another user also edited this file."

## How we teach this

### End users

The edit-metadata flow is self-discoverable via the pencil icon in the file list. No documentation change is required for basic usage, but the "Depositing records" guide should mention:
- How to edit file metadata after upload.
- What file metadata fields are available (title, description, any custom fields configured by the instance administrator).

### Instance administrators

If Approach B (custom fields) is chosen, extend the Custom Fields documentation with a `RDM_FILE_METADATA_CUSTOM_FIELDS` section, mirroring the existing `RDM_CUSTOM_FIELDS` configuration pattern.

### Developers

- Document the `If-Match` requirement for file metadata updates in the REST API reference.
- Provide a frontend component reference for `FileMetadataEditorModal`.

## Drawbacks

- **UI complexity**: Adding a modal and form per file increases the deposit form's complexity.
- **Concurrency corner cases**: The merge-and-retry logic handles most real-world cases, but rapid-fire edits (e.g., automated scripts or two users editing simultaneously) could still produce confusing results. Mitigation: retry limit with explicit user notification.
- **Schema migration (if custom fields)**: Adding or removing custom file metadata fields after data exists would require re-indexing, similar to record custom fields. If we need to index file metadata somehow. This is a known limitation of the Custom Fields framework.
- **Translation burden**: File metadata field labels and vocabulary values must be translatable.

## Alternatives

### Alternative 1: Edit metadata before upload (PR #2288 approach)

Collect metadata via Uppy's built-in metadata editor during the upload phase.

**Rejected because**: Complicates file upload flow (especially for batch uploads). Metadata update **after** a file is uploaded is not easily possible, would require a very hacky approach to implement with Uppy.

### Alternative 2: Edit metadata in a collapsible panel instead of a modal

Display the metadata form in a collapsible side panel next to the file list, similar to some DAM (Digital Asset Management) interfaces.

**Evaluation**: A collapsible panel (accordion) could be a viable option, but a modal avoids layout reflow issues in the deposit form. A collapsible panel can could be still considered if UX research favors it, might
have better accessibility.

### Alternative 3: No ETag/If-Match — last-write-wins

Omit optimistic concurrency control and simply overwrite whatever is on the server.

**Rejected because**: In multi-user or multi-tab workflows, silent data loss is unacceptable for a repository. The ETag pattern is already used elsewhere in Invenio (e.g., records API) and should be reused for consistency.

## Unresolved questions

### Metadata fields: fixed vs. custom?

This is the most important open question. The community should discuss:

- Is the proposed fixed set (`title`, `description`) sufficient for the majority of repositories?
- Are there compelling use cases that require instance-specific or discipline-specific file metadata fields?
- Should the first phase implement fixed fields, with custom fields as a second phase?

**Suggested Decision path**:
- If **≥3 independent repositories** express a need for instance-specific file metadata fields → implement **Approach B (custom fields)** from the start.
- If **most communities** agree some kind of fixed set is sufficient → implement **Approach A**.

### Should metadata be editable on published records?

The initial implementation targets **draft files only**. Editable file metadata on published records is a future enhancement that may require versioning semantics similar to record metadata updates.

### Maximum retry count for conflicts

Default is proposed as 3 retries. Should this be configurable per instance?

### Indexing and search

Should file metadata be indexed in OpenSearch/Elasticsearch for searching? If so:
- Which fields should be indexed (title only, or all metadata)?
- Should file metadata be searchable independently of record metadata?

This is deferred to a separate RFC or future enhancement.

## Resources/Timeline

**Phase 1: Community discussion & field scoping**
- Deliverable: Decision on fixed vs. custom fields

**Phase 2: Backend — fixed fields (if chosen)**
- Extend files API `PUT` endpoint with metadata support
- Implement ETag/`If-Match` logic on file metadata
- Marshmallow validation and serialization
- Estimated effort: 1–2 developer-weeks

**Phase 3: Backend — custom fields (if chosen, or as follow-up)**
- Extend custom fields framework to file metadata
- Migration CLI for mapping updates
- Estimated effort: 2–3 developer-weeks

**Phase 4: Frontend**
- Add "Edit metadata" action to file list table
- Implement metadata editor modal with form
- Implement fetch/merge/retry logic with `If-Match`
- Estimated effort: 1–2 developer-weeks

**Phase 5: Testing & documentation**
- End-to-end tests for concurrent edit scenarios
- API documentation updates
- User guide updates
- Estimated effort: 1 developer-week
