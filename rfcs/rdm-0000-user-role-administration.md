- Start Date: 2026-02-13
- RFC PR: [#0113](https://github.com/inveniosoftware/rfcs/pull/0113)
- Authors: Osama Al-Arbid (@samk13)
- State: DRAFT

# User role administration

## Summary

This RFC proposes adding role/group management to the InvenioRDM administration panel and exposing the corresponding CRUD API in `invenio-users-resources`. The feature allows authorized user administrators to list, create, edit and delete roles, while protecting system roles and superuser-related assignments from accidental or unauthorized changes.

The RFC also proposes exposing a user's roles in administration responses, so administrators can understand role membership from the user list and detail views.

## Motivation

- As an administrator, I want to manage user roles from the administration panel, so that I do not need to rely on CLI or direct database access for common role maintenance.
- As an administrator, I want to see a user's roles in the user administration UI, so that I can understand why a user has certain permissions.
- As an instance manager, I want system roles and superuser roles to be protected, so that administration users cannot accidentally break access to the instance.
- As a security-conscious operator, I want user administrators to be unable to impersonate or manage superusers unless they are superusers themselves.

### Not in scope

- Managing role membership from the role detail page.
- Managing arbitrary `invenio-access` actions from the administration UI.
- Replacing existing CLI/system-process setup for protected built-in roles.

## Detailed design

We introduce role administration in two modules:

- `invenio-users-resources` provides the service, REST API, validation, search, facets and permission model for groups/roles.
- `invenio-app-rdm` registers administration views for listing, reading, creating and editing roles, and adds role information to user administration views.

### Groups service and REST API

The existing groups resource is extended from read/search-only to CRUD:

```text
GET    /groups
POST   /groups
GET    /groups/<id>
PUT    /groups/<id>
DELETE /groups/<id>
GET    /groups/<id>/avatar.svg
```

Groups are resolved by role `id`, not role `name`. This avoids ambiguity and aligns entity resolution, avatar lookup, links and administration routes around stable identifiers.

The `GroupsService` gains `create`, `update` and `delete` methods. Mutations are executed through the accounts datastore and registered in the unit of work for indexing:

- create/update commit and refresh the indexed group document;
- delete removes the indexed group document;
- validation errors are returned as clean JSON `400` responses;
- permission failures are returned as clean JSON `403` responses.

Group payload validation is intentionally small:

- `name` is required;
- `name` must start with a letter and contain only letters, numbers, hyphens or underscores;
- `name` is limited to 80 characters;
- `description` is sanitized and limited to 255 characters;
- duplicate role names are rejected.

### Search and facets

Group search supports administration-friendly sorting and filtering:

- best match;
- name ascending/descending;
- managed first;
- unmanaged first;
- facet by managed/unmanaged state.

The administration UI reads its role search configuration from:

```python
USERS_RESOURCES_GROUPS_ADMIN_SEARCH
USERS_RESOURCES_GROUPS_ADMIN_SORT_OPTIONS
USERS_RESOURCES_GROUPS_ADMIN_FACETS
```

### Permissions

Role administration is restricted to identities with user-management capabilities. The final UI access helper checks the `administration-moderation` action through `invenio-access` permissions.

The service-level permission model adds stronger separation between regular user administrators and superusers:

- regular authenticated users can only read unmanaged groups;
- user administrators can manage managed groups;
- user administrators cannot manage unmanaged externally-provided groups;
- user administrators cannot see or manage superuser groups;
- superusers can see and manage regular managed groups;
- protected built-in roles can only be mutated by the system process.

Protected group names are configured with:

```python
USERS_RESOURCES_PROTECTED_GROUP_NAMES = [
    "admin",
    "administration",
    "superuser-access",
    "administration-moderation",
]
```

The protection applies to create, update, rename and delete operations. This keeps CLI/bootstrap-managed roles under system-process control.

### Administration UI

`invenio-app-rdm` registers a new role administration section under user management:

- role list;
- role detail;
- role create;
- role edit.

The views use the generic administration resource views and the `/groups` API. They expose the fields needed for role maintenance: `id`, `name`, `description`, `is_managed`, `created` and `updated`.

The existing user administration list and detail views are also protected by the same administration access helper.

## Drawbacks

- Role membership is visible in user administration responses, which requires careful permission checks.
- Protecting built-in roles by configured names requires deployments to keep the protected-name list aligned with their bootstrap roles.

## Alternatives

- Keep role management CLI-only. This avoids UI/API complexity but does not satisfy common administration workflows.

## Unresolved questions

The main unresolved question is whether Invenio should continue using role `id` instead of role `name` for authorization checks.

There are two possible paths:

- **Continue with role `id`:** keep role `id` as the authorization key and introduce a centralized cached name-to-id resolution layer for configured/readable role names. This avoids repeated database lookups in permission checks, but it is a breaking change for existing custom code and configuration that checks against role `name`.
- **Revert to role `name`:** revert the role id change and keep role `name` as the authorization/configuration key. This is simpler for instance configuration and existing code, but keeps the risk around unmanaged/SSO roles where names can change or be reused.

## Resources/Timeline

The implementation is already implemented in the master branch of `invenio-app-rdm` v14.0.0b5.dev2
