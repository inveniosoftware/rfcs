- Start Date: 2023-04-12
- RFC PR: [#71](https://github.com/inveniosoftware/rfcs/pull/71)
- Authors: Zacharias Zacharodimos

# Deletion of communities and users as related resources

### Summary

This RFC describes the technical implementation of the mechanism that returns a "ghost" representation of any deleted community and/or user in every associated request.

### Motivation

In InvenioRDM, communities and users are linked with requests in a way that if a community or user is deleted, any requests linked to them will also be affected.

When a community is deleted, any requests associated with that community will no longer be associated with a valid community. Similarly, if a user is deleted, any requests associated with that user will no longer be associated with a valid user.

Therefore, it is important to carefully consider the impact of deleting communities and users within InvenioRDM, and to ensure that any associated requests are properly managed and re-assigned as needed.

### Detailed design

When a related resource is resolved using the `FieldResolver`, a search is executed in opensearch instead of accessing the db.The reason for that is to maximize performance when expanding related fields on demand.

Once a community/user is deleted, then it cannot be found in the search results anymore but the related ids are still present in the related resources. More specifically, the following cases are covered:

- `record.parent.review.receiver`
- `record.parent.communities.default`
- `request.created_by`

The proposed solution is to hook into the dereferencing mechanism of the field resolution and define a new abstract method that is required to be implemented by the concrete implementations of expandable fields.

More specifically, the abstract class of `ExpandableField` is changed as:

```python
class ExpandableField(ABC):
    """Field referencing to another record that can be expanded."""
    ...

    @abstractmethod
    def ghost_record(self, value):
        """Return the ghost representation of the unresolved value.
        This is used when a value cannot be resolved. The returned value
        will be available when the method `self.pick()` is called.
        """
        raise NotImplementedError()

    def add_dereferenced_record(self, service, value, resolved_rec):
        """Save the dereferenced record."""
        if resolved_rec is None:
            resolved_rec = self.ghost_record({"id": value})
            # mark the record as a "ghost" record i.e not resolvable
            resolved_rec["is_ghost"] = True
        self._service_values[service][value] = resolved_rec

```

The goal of this method is to return the "ghost" representation of a related entity once the latter is not resolvable. The derefenicing mechanism enriches that resolved data with a new key called `is_ghost` that indicates whether the expanded field is a valid resolved value or "ghost" one.

This key is used across the RDM ecosystem to identify if the information received belongs to an existing entity or is just the "ghost" representation.

#### Ghost community

The ghost community representation is defined as a marshmallow schema:

```python
class CommunityGhostSchema(Schema):
    """Community ghost schema."""

    id = SanitizedUnicode(dump_only=True)
    metadata = fields.Constant(
        {
            "title": _("Deleted community"),
            "description": _("The community was deleted."),
        },
        dump_only=True,
    )
    is_ghost = fields.Boolean(dump_only=True)
```

The schema above is used when resolving a deleted community as part of the field expansion on:

- `record.parent.review.receiver`
- `record.parent.communities.default`

#### Ghost user

The ghost user representation is defined as a marshmallow schema:

```python
class UserGhostSchema(Schema):
    """user ghost schema."""

    id = SanitizedUnicode(dump_only=True)
    profile = fields.Constant(
        {
            "full_name": _("Deleted user"),
        },
        dump_only=True,
    )
    is_ghost = fields.Boolean(dump_only=True)
```

The schema above is used when resolving a deleted user as part of the field expansion on:

- `request.created_by`

#### REST API response

##### Example

An example response, when expanding a record with an open request to a community and the latter is deleted is show below:

```json
{
...
"expanded": {
    "parent": {
      "review": {
        "receiver": {
          "id": "f7f24b1b-f563-4416-b845-9dff09f123af",
          "metadata": {
            "title": "Deleted Community",
            "description": "The community was deleted."
          },
          "is_ghost": true
        }
      }
    }
    }
}
```

### UI/UX

Below there is a list of images, presenting how the different views across the InvenioRDM UI are visually changing based once a community is deleted i.e marked as `is_ghost=True`.

#### User dashboard requests

##### Accepted/Declined/Cancelled requests

![](https://codimd.web.cern.ch/uploads/upload_9cd2a47de4732269ed4eb8a993009534.png)

![](https://codimd.web.cern.ch/uploads/upload_f26e92cf0a5489df3c58db2b7e927eed.png)

#### Record landing page

If a community is deleted, then no header will be shown in a published record.

![](https://codimd.web.cern.ch/uploads/upload_f744c5ba1a8bf19471bccbc8970e61b8.png)

#### Upload form

##### Published to a deleted community

When editing a published community, the community header is not shown in alignement with the landing page

![](https://codimd.web.cern.ch/uploads/upload_82fb08cba76c5f8c455f9134d486ac3b.png)

##### Declined upload of a deleted community

![](https://codimd.web.cern.ch/uploads/upload_fa256632c083fb2dba9509ffeeb0ac0e.png)

### Future work

- When a community/user is deleted, a notification should be triggered towards
  - the creator of the request, if open requests exist
  - the community members
- Log an action on the requests associated with the deleted community so users can understand what happened
