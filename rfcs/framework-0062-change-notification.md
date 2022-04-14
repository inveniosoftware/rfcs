- Start Date: 2022-03-08
- RFC PR: [#62](https://github.com/inveniosoftware/rfcs/pull/62)
- Authors: Pablo Panero, Alex Ioannidis

# Record change notifications

## Summary

Record might have relations between one another. For example a _community record_ is related to _user records_. In addition, services might want to execute some functionality when the related records change, for example reindexing the record to keep the denormalized data consistent in Elasticsearch.

## Motivation

```
Q: the below use case sounds very weird (and obvious)
- As a developer I want to allow performat propagations of related records' updates (Without having to much config..)
- As a user, I want to be able to search on records with the latest information. For example, seeing the updated list of users who are memeber of my community, or affiliations changes on record's creator.
```

## Detailed design

### Backrelations definitions (TODO)

The backwards or inverse relations are defined in the record API class, along with the relations themselves.

```python
class Community:

    relations = RelationsField(
        assigee=PIDRelation(
            'metadata.assignee',
            backrel=['user']
            ...
        ),
        ...
    )
```

The `RelationsField` would store an inverted index in the `RelationsMapping`, to be able to query for the fields to updated.

### Change notifications registry

The change notification registry would hold the information about which functions to run when a record is updated.

```python
class ChangeNotificationRegistry:

    def __init__():
        self._registry = {}

    def register(record_type, handler, **kwargs):
        if self._registry.get(record_type, None):
            self._registry[record_type].append(handler)
        else:
            self._registry[record_type] = [handler]

    def get(record_type):
        return self._registry.get(record_type, None)
```

Notification handlers would be registered on extension loading following the same mechanism than for services and indexers.

```python
notification_registry = ...

notification_registry.register(
    "user",
    "communitymember.on_relation_update",
)
```

### Service level: UoW operation

The notification will be send after the record has been commited so the `version_id` has been increased.

```python
 class ChangeNotificationOp(Operation):

    def __init__(self, record_type, records):
        self._record_type = record_type
        self._records = records

    def on_post_commit(self, uow):
        send_change_notifications.delay(
            self._record_type,
            [
                (r.pid.pid_value, str(r.id), r.revision_id)
                for r in self._records
            ]
        )

```

### Service level: component

Records change when the `service.update` (or `update_many`) is called. After they have been updated, service components will run. One of those components would be `ChangeNotificationsComponent`, which would register the change notification task.

While it is true that registering a task per update could potentially lunch several tasks, optimistically speaking they would not lunch any re-index since they would be taken care of by previous ones and filtered out by the `indexed_at` part of the query.

```python
class ChangeNotificationsComponent(ServiceComponent):

    def update(self, identity, **kwargs):
        self.uow.register(ChangeNotificationOp(
            record_type=self.service.id,
            records=[record],
        ))
```

### Async task

The celery task would execute the notification handlers registered for the record type.

```python
@shared_task(ignore_result=True)
def send_change_notifications(record_type, records_info):
    task_start = arrow.utcnow().strftime("%Y-%m-%dT%H:%M:%S.%f")

    handlers = current_notifications_registry.get(record_type)
    for notif_handler in handlers:
        notif_handler(
            system_identity, record_type, records_info, task_start
        )
```

### Service level: notification handler

```python

def on_relation_update(
    self, identity, record_type, records_info, notif_time
):
    relations = self.record_cls.relations
    fieldpaths = relations.inverse_get(record_type)
    clauses = []
    for field in fieldpaths:
        for record in records_info:
            recid, uuid, revision_id = record
            clauses.append(Q("bool", must=[
                Q("term", **{f"{field}.id": recid}),
                Q("term", **{f"{field}.@v": f"!='{uuid}:{revision_id}'"})
            ]))

    filter = [Q("range", indexed_at={"lte": notif_time})]
    es_query = Q(
        "bool", minimum_should_match=1, should=clauses, filter=filter
    )

    self.reindex(identity, es_query=es_query)
    return True
```

## How we teach this

The implementation of the change notifications will be part of the core codebase. How to configure a record to have the relations updated would be documented.

## Drawbacks

...

## Alternatives

The usage of the registry for handler functions and the creation of a new UoW operation type run on `post_commit` shows that this use case could be suitable for an event based architecture. It boils down to an event bus (registry) for only one type of even (implicit). When the `event bus` is implemented this functionality could be refactored to be event based.

## Unresolved questions

**Async task**

What is a sensitive slice number for records loading. This could be an issue if the task receives a very big list of updated records. This is not a problem right now since there is no bulk update nor _update many_ implementations.

**Service.on_update_relations**

It could be smart to query also for those fields that have changed in the field and thefore not reindex if the dereferenced fields are not updated.

**Deletion**

When a record with back relations is deleted, how are the related ones updated?

a. This actions is not allowed. It is only possible via support, when all the relations had been properly handled.
b. If the back relation record is mandatory, the reference is removed.
c. If the back relation record accepts custom values (e.g. subjects) we add it as custom.

## Resources/Timeline

- ...
