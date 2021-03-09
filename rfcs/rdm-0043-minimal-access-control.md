- Start Date: 2021-02-03
- RFC PR: [#43>](https://github.com/inveniosoftware/rfcs/pull/43)
- Authors: Lars Holm Nielsen
- State: IMPLEMENTED

# Minimal access control for InvenioRDM

## Summary

This RFCs provide an overview of how minimal access control is going to be implemented in InvenioRDM. It will allow access control similar to Zenodo with the addition of fully restricted records.

## Motivation

Following are some user user stories:

- As a researcher, I want to upload my (open access or otherwise) publication, so that I can comply with funder/institution mandates.
- As a researcher, I want to upload my embargoed publication, so that I can comply with funder/institution mandates.
- As an uploader, I want to be able to set an embargo date, so that my upload can be made automatically available.
- As an uploader, I want my record metadata to be viewable by institutional staff but files only viewable by department X, so that a record is finddable by staff but files are restricted to a specific department.
- As an uploader, I want to share access to my upload, so that my colleageus can help me fill in the metadata or validate the upload.

For an initial minimal access control in InvenioRDM we want to be able to support the following use cases:

- Upload a record with public metadata and public files
- Upload a record with public metadata and private files
- Upload a record with private metadata and private files (i.e. record only visible by uploader and system administrator).
- Set an embargo date on which the metadata and files become public.

We do not support guests (unauthenticated users) submitting records.

## Background

### References

- [Invenio-Access documentation](https://invenio-access.readthedocs.io/en/latest/overview.html)
- [Flask-Principal](https://pythonhosted.org/Flask-Principal/)

### Definitions

**Resources**

A *resource* is something you want to protect. A record, a page, etc.

**Actions**

Actions are operations that you can do with a resource. There's the usual Create, Read, Update, Delete (CRUD) actions, but it can also be more high-level things like *can publish*.

**Subjects**

Subjects are the entities in the system that you can grant access to perfrom certain actions on a resources. Invenio has the following subjects:

- *Users* - represents an authenitcated account in the system.
- *Roles* - represents a job function (e.g. admins or curator) and are explicitly assigned to one or more users.
- *System roles* - roles automatically assigned by the system to users (e.g. any user, authenticated user, campus user).

**Identity**

An identity is an abstraction layer over users and roles to represent everything a user is allowed to do. For instance, a user may be assigned a role, and the role is granted access to deposit records. The identity combines all the actions that a user is granted via their user, roles and system roles.

Thus, from the access control system point of view, we always ask if an given *identity* is allowed to do something, and we never ask if a given *user* is allowed to do something.

**Permissions**

A permission is an statement on which actions are **required** to access a given resource.

For instance, a permission may express that it requires both the *create action* and *publish action*.

**Needs**

At the core, the access control system operates with a basic unit called **needs**. You can think of them as badges or smallest level of access control.

A need express statements such as "action admin-access". Needs are used both to express what is **required** but also what an identity **provides**.

Example:

- The landing page resource is protected with a permission that **requires** the **need**: read record 1 action.
- The identity of the current user then **provides** the **need**: read record 1 action

Checking the permission amounts to a **set intersection** between 1) required needs and 2) provided needs.

**Permission policies**

We group actions on a specific resource into what we call a **permission policy**. For instance a record permisison policy groups the following actions:

- can search records
- can create records
- can read a specific record
- can update a specific record
- can delete a specific record

## Design

### Resources

From a high-level overview InvenioRDM *currently* provides the following resources that needs to be protected:

- Records, drafts and files
- Vocabularies

In addition further resources will be added in the future such as e.g. communities. This RFC focuses on records, drafts and files. For vocabularies, we will implement a read-only mode on the REST API which will allow searching records but we will not allow creation or restriction of records in the vocabularies yet.

### Protection

A record can define if restrictions apply to a) the metadata, b) the files or c) the metadata and files:

- Metadata protection: Public or Restricted.
- Files protection: Public or Restricted.

If restrictions applies, the record is subject to access control. The purpose of defining the restrictions is two-fold: 1) performance and 2) easier to understand UX.

It is not possible to have restricted metadata and public files.

### Embargo

If either the metadata and/or files are protected, an embargo can be applied to the record. When an embargo is lifted the metadata **and** files proteciton is both set to public, and the embargo is deactivated. During the embargo, the record behaves as a normal restricted record.

The embargo is set using:

1) A timestamp on which the embargo should be lifted.
2) An optional textual reason for the embargo.

### Access status

The access status is a computed property that is used to inform users about the level of access control applied to a record. We rely on the [COAR access status vocabulary](http://vocabularies.coar-repositories.org/documentation/access_rights/). with the folllowing mapping:

1. Open:
  a. Public records **with public files**
2. Embargoed
  a. Public records **with restricted files under embargo**
  b. Restricted records **under embargo**
3. Metadata-only
  a. Public records **without files**
  b. Public records **with restricted files** (not under embargo)
4. Restricted
  a. Restricted records (not under embargo)

### Permission levels

A record permission level defines an aggregated set of low-level permissions, that grants increasing level of permissions to a record. We define the following four record permission levels that will be selected by users in the interface:

- View metadata: Allows viewing the metadata of a restrictred record.
- View metadata and files: Allows viewing the metadata and files of a restrictred record.
- Edit: Allows editing the metadata and files of a record
- Manage: Allows managing permissions of a record

In addition two hidden permission levels exists:

- Owners: Allows adding new owners and transfering ownership of a record.
- Administrators: Allows special actions like deletion of published records.

It's important to note, that a permission level includes the permissions on that level AND all permissions from lower levels. For instance, the edit permission level includes the permission from the edit, the view metadata and files and view metadata permission levels.

### Record ownership

All records in the system can be owned by both people or groups. Ownership of a record gives "owners" permission-level. By default, ownership will be assigned to the user submitting the record, however, InvenioRDM should not require that the submitter also becomes the owner.

### Grants

When a record's metadata and/or files are restricted, the access to a record is defined via grants. A grant consists of:

- Subject (user, role or system role)
- Permission level

Example: User 1 is given "Edit" permission level.

The grant gives access to the current per


### Schema

We model above concepts inside the record using the following structure:

```json
{
    "access": {
        "owned_by": [{"user": "1"}],
        "record": "public|restricted",
        "files": "public|restricted",
        "embargo": {
            "active": true,
            "until": "2021-02-09T12:00:00",
            "reason": "Some sort of reason.",
        },
        "grants": [
            {"subject": "user", "id": "2", "level": "manage"},
            {"subject": "role", "id": "curator", "level": "edit"},
            {"subject": "sysrole", "id": "authenticated_user", "level": "view"}
        ]
    }
}
```

### Indexing and grant tokens

Above structure is not well-suited for queryin. We therefore during indexing translate the grants in above structure into a list of grant tokens define as:

```
<permission level>-<subject>-<id>
```

For query performance we expand a grant into all the permission-levels. For instance:

```json
{"subject": "role", "id": "curator", "level": "edit"}
```

is translated into:

```
viewmeta-role-curator
viewfull-role-curator
edit-role-curator
```

Above expansion requires extra index storage, but ensures that we can perform exact looks up.

### Checking permissions

**Service layer**

It is primarily the service layer which is responsible for performing permission checks. Reading, creating, updating, publish records should always go through the service layer to ensure coherent application of permissions.

For instance the service layer is responsible for filtering out any data from a record that a given identity is not allowed to access.

Example: a given identity is not granted access to read the files of a record. The files part of the record is filtered out by the service layer. This way, the list of files will never reach the presentation layer (HTML landing pages or REST API serializers).

**Presentation layer exceptions**

In some cases, the presentation layer will have to do conditional rendering - e.g. to show an edit button only if an identity is allowed to update the record.

In these cases, the presentation layer should delegate the permission check to the service layer.

**Data access layer**

The data access layer should under no circumstances perform any permissions check.


### REST API

#### Serialization

- Basic information
    - Permissions should require manage permission to update and view.

```json
{
    "access": {
        "record": "public|protected",
        "files": "public|protected",
        "embargo": {
            "active": true,
            "until": "",
            "reason": "",
        }
    },
    "permissions": {
        "can_view": true,
        "can_edit": false,
        "can_manage": false,
    }
}
```

## UI/UX Design

### Landing page

A landing page display a single record. This includes for instance:

- Landing page: https://inveniordm.web.cern.ch/records/rny5x-zt515
- Export format pages: https://inveniordm.web.cern.ch/records/rny5x-zt515/export/dc

**Authorization**

Any landing page should require ``read`` permission to the entire record.

The record MUST be fetched through the service layer to ensure that field-level permissions have been applied (i.e. metadata that is not allowed to be access by a given identity is filtered out, e.g. specifically that files have been filtered out or not).

**Field-level permissions**

- Files: The ``files`` field should require ``read-files`` permission.

Later, further fields may need to be restricted, specifically access related information.

**Conditional rendering**

- Manage panel: Each landing page should display a special "manage" section which contains e.g. an Edit button. This section MUST require ``update`` permission to be rendered. Subparts MAY require other permissions (e.g. delete button MAY require ``admin`` permission).
- Files box:
    - Record without files:
        - Files box show that it is metadata only record.
    - Record with files
        - Files included in metadata (i.e. user has access):
            - If public: normal rendering.
            - If protected and files are embargoed: render with yellowish file box.
            - If protected and files are not embargoed: render with redish file box.
        - Files not included in metadata (i.e. user has no access):
            - If files are embargoed: render embargo information
            - If files are protected: render information that files are protected
- Preview:
    - Same rules as for the files box.

**Special considerations**

Invenio-Records-UI is not aware of the new Invenio-Records-Resources and the service layer. We will thus have to reimplement Invenio-Records-UI using the service layer to ensure the service layer is always used.

### Search results

**Authorization**

The list upload page should require ``search`` permission.

**Conditional rendering**

- List of records is determined by the REST API, but should only show records that a user has ``read`` permission for.
- Files are only included in the REST API result if a user has access to them.

### List uploads

**Authorization**

The list upload page should require ``create`` permission.

**Conditional rendering**

- List of records is determined by the REST API, but should only show records that a user has ``update`` permission for.

The REST API should define the following facets:

- Status: Draft or Published
- Protection: Public, Public with protected files, Embargoed (file-only), Embargoed (full record), Protected

### Deposit form

**Authorization**

The list upload page should require ``create`` permission. In addition it should require ``update draft`` for the given record.

**Conditional rendering**

The "protection" seciton should require ``manage`` permissions to be rendered. If a user does not have permissions, a box should be displayed that tells the user they don't have access to manage permissions.

### Permission denied error

In the UI, if an anonymous user does not have permission to access a given record, they should be redirected to a login page. If a user is logged in, and does not have permissions, they should be presented with an error page displaying permission denied error.

In the REST API, if a user is not logged in an HTTP 401 error is returned, otherwise a HTTP 403 error. This mans that existstence of a given record can be learned by an attacker. As identifiers are randomly generated, no information is leaked from the identifier itself.

### Conditional rendering in templates

Templates may need to display specific elements depending on the users permissions. For consistency, we imagine this being done by a serializer producing a JSON that includes information needed for rendering. E.g.

```
{% if record.permissions.can_view %}
{% endif %}
```

### Embargo updater

We need a Celery task running on a regular basis, which X times a day scan for records where the embargo expired. The actual logic should be implemented in the RDMRecord service. The query is:

```
access.embargo.active:true AND access.embargo.until:<={now}
```

The embargo updater just have to change the protection mode accordingly:

- Record protection: Public
- Files protection: Public
- Embargo: set as inactive

It does not matter if the full record or only the files were embargoed, since the end state in both cases is the same.

### Vocabularies REST API.

As an initial simplification we'll make the vocabularies REST API read-only.

## Unresolved questions

- How we lift an embargo on when both a record and draft exists?

## Resources/Timeline

> Which resources do you have available to implement this RFC and what is the overall timeline?
