- Start Date: 2019-10-24
- RFC PR: [#12](https://github.com/inveniosoftware/rfcs/pull/12)
- Extends RFC: [#7](https://github.com/inveniosoftware/rfcs/pull/7)
- Authors: Guillaume Viger

# Access Control Record Metadata

## Summary

RFC [#7](https://github.com/inveniosoftware/rfcs/pull/7) covers the overall
access control strategy and the policy implementation details. This RFC (#12)
complements #7 by outlining the permission metadata fields on a record and how
those should be interpreted.

## Motivation

The user stories it supports are the same as #7. A representative one being:

As a user, I want to grant access to my restricted records by listing specific
users and/or groups, so that I can easily revoke the access if needed.

Presently, encoding in a record metadata that a specific user can perform an
action is not possible. This RFC proposes a way to achieve this.

## Detailed design

At the record schema level, access control can be encoded in various means as
long as the appropriate Generators exist. This RFC proposes a metadata schema
for additional explicit targeted permissions and how they should be interpreted.

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
        "type": {
          "type": "string",
          "description": "Kind of permission_identity. What the id represents.",
          "enum": [
            "person",
            "role",
            "org"
          ]
        }
      },
      "required": ["id", "type"]
    }
  },
  "permissions": {
    "type": "object",
    "description": "Additional explicit access control for specific user/groups and specific actions",
    "properties": {
      "can_read": {
        "type": "array",
        "items": { "$ref": "#/definitions/permission_identity" }
      },
      "can_update": {
        "type": "array",
        "items": { "$ref": "#/definitions/permission_identity" }
      },
      "can_delete": {
        "type": "array",
        "items": { "$ref": "#/definitions/permission_identity" }
      }
    },
    "additionalProperties": true,
  }
}

```

The `permissions` field should be part of the internal (`sys`) metadata schema
section, while the `access_right` field should be part of the public metadata
schema.

**Interpretation**

Policies and their Generators will be provided out-of-the-box to enforce the
permissions listed in the record.

The fields `permissions.can_read, .can_update, .can_delete, ...` list the
`permission_identities` that have been given **additional** permission to
read, update, delete... It is used to give permissions to entities that would
not have been otherwise covered by the permissions given to `owners` or
`access_right` or other reasons.


## How we teach this

This new `permissions` field should be described as a means to **add additional
explicit permissions**. "Add" because this approach doesn't subtract otherwise
permitted identities. It whitelists and de-whitelists, but doesn't blacklist.
"Additional" because there are already other ways of granting permissions
depending on the enabled Policies. "Explicit" because the content of this field
is clear and self-explanatory. The term also draws a contrast with how permissions
can be granted through Policies that look at a combination of other fields and
are therefore "implicit".

Documentation about this schema and its interpretation should be part of the
general RDM access control documentation.


## Drawbacks

There is a subtlety in the fact that the `permissions` field does not
explicitly list *all* permissions. It only seeks to list *additional* permissions.
In fact, most permissions will be implicit through Policies. This will have to
be addressed via documentation.


## Alternatives

An even more generic `permissions` like so:

```json
"permissions": {
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "permission_identity": { "$ref": "#/definitions/permission_identity" },
      "action": {
        "type": "string",
        "description": "Typically 'read', 'update', 'delete', but you can add your own."
      }
    }
  }
}
```

was considered, but would not be as practical for Elasticsearch queries.


## Unresolved questions

Should blacklisting be allowed? I don't think so anymore, but the practical experience
of service operators should prevail here; it could be the case that  an "open"
record has files and a bad actor is relentlessly downloading them and they need
to be stopped. Even that scenario could be dealt with differently I believe...
