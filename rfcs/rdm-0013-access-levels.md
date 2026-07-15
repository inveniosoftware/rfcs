- Start Date: 2019-11-27
- RFC PR: [#13](https://github.com/inveniosoftware/rfcs/pull/13)
- Extends RFC: [#7](https://github.com/inveniosoftware/rfcs/pull/7)
- Authors: Guillaume Viger

# Access Levels

## Summary

RFC [#7](https://github.com/inveniosoftware/rfcs/pull/7) covers the overall
access control strategy and the policy implementation details. In particular,
it covers low-level permissions such as `can_create`, `can_read`, `can_update`
and so on. This RFC (#13) complements #7 by outlining access levels (typically)
on a record. Access levels are shorthand labels that tie a set of low-level
permissions to an identity on a record.

## Terminology

**Access level**: A human-friendly label that ties a set of low-level
permissions to an identity on a record. Example: Taylor has *Metadata Curator*
access level on record A which means they can read and edit metadata of A.
**Identity**: This is the person, role or organization that holds an access
level. Example: Taylor in the above case.

## Motivation

The user stories it supports are a subset of those in #7. Namely:

- As an administrator, I want to restrict access to certain records based on
  user features (e.g. registered user, member of a specific community, member
  of academia), so that I can implement my institution's policies.
- As a user, I want to be able to obtain a shareable link, so that my
  colleagues can access the files of my **restricted record**.
- As a user, I want to share a preview of a **deposit** record prior to
  publishing it, so that my colleagues can validate it.
- As a user, I want to grant access to my restricted records by listing
  specific users and/or groups, so that I can easily revoke the access if needed.

Convenience is also an important factor:

- As an administrator, I want to keep track of the reasons behind assigned
  permissions, so that I can change them confidently.
- As a user, I want to allow another user without setting each low-level
  permissions individually but instead use a predefined group, so that I can
  do so quickly and without contradictions.

Presently, encoding in a record that a specific user is allowed a
range of actions on that record is not possible. This RFC proposes a way to
achieve this.

## Detailed design

This RFC proposes (1) various access levels and the low level permissions they
map to, (2) the record schema encoding and (3) the PermissionPolicy
configuration and the Generator used.

**Proposed access levels and corresponding permissions**

| Access Level / Low-level Permissions   | read_metadata      | read_files         | update_metadata    | update_files       | delete             |
|----------------------------------------|--------------------|--------------------|------------------- |--------------------|--------------------|
| metadata_reader                        | :heavy_check_mark: |                    |                    |                    |                    |
| metadata_curator                       | :heavy_check_mark: |                    | :heavy_check_mark: |                    |                    |
| files_reader                           | :heavy_check_mark: | :heavy_check_mark: |                    |                    |                    |
| files_curator                          | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: |                    |
| admin                                  | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: |


**Proposed schema fields**

```json
{
  "definitions": {
    "permission_identity": {
      "type": "object",
      "description": "Identity that can be targeted by a permission.",
      "additionalProperties": false,
      "type": "object",
      "properties": {
        "id": {
          "type": "string",
          "description": "Per type unique id.",
        },
        "scheme": {
          "type": "string",
          "description": "Kind of permission_identity. What the id represents.",
          "enum": [
            "person",
            "role",
            "org"
          ]
        }
      },
      "required": ["id", "scheme"]
    }
  },
  "access_levels": {
    "type": "object",
    "description": "Assigned access levels to permission identities",
    "properties": {
      "metadata_reader": {
        "type": "array",
        "items": { "$ref": "#/definitions/permission_identity" }
      },
      "metadata_curator": {
        "type": "array",
        "items": { "$ref": "#/definitions/permission_identity" }
      },
      "files_reader": {
        "type": "array",
        "items": { "$ref": "#/definitions/permission_identity" }
      },
      "files_curator": {
        "type": "array",
        "items": { "$ref": "#/definitions/permission_identity" }
      },
      "admin": {
        "type": "array",
        "items": { "$ref": "#/definitions/permission_identity" }
      }
    },
    "additionalProperties": true
  }
}

```

The `access_levels` field should be part of the system (`internal`) metadata
schema section.

**PermissionPolicy Configuration + Generator**

InvenioRDM's default `PolicyPermission` will have the `AllowedByAccessLevel`
generators set by default:

```python
class RecordPermissionPolicy(BasePermissionPolicy):
    """Access control configuration for records."""
    # Other permissions omitted for clarity but same pattern applies
    can_read = [..., AllowedByAccessLevel('read')]
```

This Generator encapsulates the logic to select the action, verify the access
level and allow the permitted identities.

Access levels are a means to add permissions, not subtract them.


## How we teach this

Access levels map well to every-day permission granting. Bob wants Alice to
check his record's metadata before making it fully public: he gives her the
Metadata Curator access level on his record. What needs to be clear is what
each access level means, so that the right one can be picked. The UI should
provide that information when it comes time to choose.


## Drawbacks

Because InvenioRDM decides the mapping of access level to permissions, there
will always be administrators who have different needs. They will have to
code their own, but the structure allows it.

Granular control on permissions might be needed in the future, because the
access levels are too broad. Supporting higher and lower-level permissions
might prove more complex than desired. Access levels *do* provide a more
convenient solution and could be extended to cover **new** access levels
however.


## Alternatives

Using the explicit permission approach of shelved RFC #12 with added
`"reason"` field was discarded. Although, it would bring everything down to
lower-level permissions and keep an explanation for attributed permissions
(through the `"reason"` field), it is clunky and contradiction-prone
implementation wise.
