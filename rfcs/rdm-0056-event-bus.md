- Start Date: 2022-02-01
- RFC PR: [#56](https://github.com/inveniosoftware/rfcs/pull/56)
- Authors: Pablo Panero, Alex Ioannidis  


# Event Bus

## Summary

This RFC defines the implementation design of the _event bus_ pattern in order to provide support for event-driven design in InvenioRDM.

## Motivation

The _Unit of Work_ pattern has been implemented in Invenio in order to group several actions into one atomic operation, meaning one database commit.

Moreover, there are actions that can trigger other potential operations. For example, _publishing a record_ can trigger an OCFL object creation, a webhook delivery and/or the indexing of said record in OpenAIRE. All these triggered actions can vary (i.e. are configurable). Therefore, they are isolated from the main system flow (_record publish_), who has no knowledge of what operations will be triggered and who will perform them.

In terms of those triggered operations, the user stories are:

- As a developer I want to implement operations that should happen after the _unit of work_ has been commited.
- As a developer I want to implement new ways of queueing events (e.g. adapt to a new technology).
- As a system administrator I want to be able to configure the handlers for specific events.
- As a system administrator I want to be able to configure which type of queue is used for the bus (e.g. enable persistance).

## Detailed design

Across this RFC the term _persistence_ will be mentioned when refering to saving events into a persistent storage. On the other hand, the term _logging_ will refer to the auditing of the events (i.e. which events have been handled, and which have errored, etc.). You can read more about this topic in the _unresolved questions_ section.

### Events

Events are _dataclasses_, which contain **facts** about **what happened**, and have no capabilities to perform actions. The event class will represent the action itself (e.g. _record created_). In addition, events will have two attributes that will help identification if needed:

- `type`, which represents the entity to which it refers to (e.g. a record, file, community).
- `action`, which represents the action that took place over the entety (e.g. create, update, publish).

```python

@dataclass
class RecordEvent(Event):
    recid: dict
    type: ClassVar[str] = "RECORD"
    handling_key: ClassVar[str] = "RECORD"


@dataclass
class RecordCreatedEvent(RecordEvent):
    action: ClassVar[str] = "CREATE"
    handling_key: ClassVar[str] = f"{RecordEvent.type}.{action}"

event = RecordCreatedEvent(
    created = datetime(2022, 2, 8)
    recid = "12345-ABCDE"
)
```

The `handling_key` is used to assign event handlers, you can read more about it in the next section, in _[how to add handlers](#How-to-add-handlers)_.

### Event handling

Event handlers receive an event and perform the desired action with them. Note that event handling is implemented in an optimistic fashion, therefore events handlers might be called more than once and/or out of order. As a consequence, every implementation should be **idempotent** and take into consideration optimistic concurrency.

In addition, all event handlers must be celery tasks. Then, they can be configured to be run eagerly instead of delayed.

#### How to add handlers

Handlers can be configured as a mapping of event classes _handling keys_ to a _list of handlers_, using the `EVENT_HANDLERS` variable.

Take into account that handling keys are treated similar to rounting keys in queuing systems. The string value will be splited by `.` (dot) and chain the handlers. For example the handling key `parent.child` will be assing all hadnlers for the key `parent` and `parent.child`. For example:

```python
EVENT_HANDLERS = {
    RecordEvent.handling_key: [],
    RecordCreatedEvent.handling_key: [
        (sync_handler_task, True),
        (explicit_asyn_handler_task, False),
        implicit_asyn_handler_task,
    ],
}
```

Finally, note that some of the previous handlers are tuples and some string. This is used to specify if they should be handled synchrouns or asynchronously. By default, tasks are delayed (i.e. run asynchronously), this can be explicitly stated by setting the second part of the tuple to `False`.

### Plumbing, the event bus

Once we have the events, and we know how they are handled we need to trigger them somehow.

The events are stored in a configurable queue. It uses `invenio-queues` for queue handling, which exposes a simple API `publish` and `consume`.

Note that dataclasses are not JSON serializable. Since they do not contain anything else than plain data, using pickle does not pose a security concern.

```python
import pickle

class EventBus:

    def __init__(self, queue_name=None):
        self._queue = ... # obtained from invenio-queues

    def publish(self, event):
        return self._queue.publish(pickle.dumps(event))

    def consume(self, event):
        for event in self._queue.consume():
            yield pickle.loads(event)
```

Events are registered to the _unit of work_ as any other operation, with the constraint that the logic (publishing) of the event **must** happen in the `on_post_commit` hook.

```python
class UnitOfWork:

    def __init__(self, session=None):
        self._event_bus = EventBus()


class SendEventOp(Operation):

    def __init__(self, event, *args, **kwargs):
        self._event = event

    def on_post_commit(self, uow):
        uow._event_bus.publish(event)
```

The following is an example of how an event would be added from a _service_ call. However, note that _service components_ have access to the service itself so there would be no difference on how to do it.

```python
@unit_of_work()
def create(self, ...):
    ...
    # persist record to DB and Index it
    uow.register(RecordCommitOp(record, self.indexer))
    # add events
    event = RecordCreatedEvent(recid=..., created=...)
    uow.register(SendEventOp(event))

    return self.result_item(...)
```

Once the service action has finished, and the `uow` has commited itself, it will execute the `on_post_commit` hooks, which will publish the events to the queue.

### Receiving and handling events

Ultimately, the responsability to handle the events is falls on the _event worker task_. This task will be periodically spawned by Celery beat, similarly to how bulk indexing works in Invenio Framework. Therefore, the processing of events will happen in pseudo real-time, and will have a pseudo autoscalability built in. This comes from the fact that workers will be spawned even if the previous ones did not finish, increasing that way the amount of workers. This can be configured using the `CELERYBEAT_SCHEDUL` variable:

```python
CELERYBEAT_SCHEDULE = {
    'event_handling': {
        'task': 'invenio_records_resources.services.events.handle_events',
        'schedule': timedelta(minutes=5),
    },
}
```

Moreover, it is important to configure the celery beat tasks correctly to avoid an excesive resource consumption (e.g. long lived open DB connections). For example, by setting a maximum amount of events handled and a TTL. See implementation for more details [invenio-records-resources#300](https://github.com/inveniosoftware/invenio-records-resources/pull/300).

> FIXME (Pablo): Did not find a configuration e.g. on consumers to limit the amount of events/life of a consumer

```python
@shared_task(ignore_result=True)
def handle_events(queue_name=None, max_events=1000, ttl=300):
    bus = EventBus(queue_name)
    start = datetime.timestamp(datetime.now())
    end = start
    spawn_new = False
    with bus.active_consumer() as consumer:
        while max_events > 0 and (start + ttl) > end:
            spawn_new = False
            event = consumer.consume()  # blocking
            _handle_event(event)  # execute all handlers
            end = datetime.timestamp(datetime.now())
            spawn_new = True

    if spawn_new:
        handle_events.delay(
            queue_name=queue_name, max_events=max_events, ttl=ttl
        )

```

## Example

**A lightweight case: metrics and monitoring**

Considering metrics and monitoring a lightweight case, the event handler will be implemented as a synchronous task, i.e. the `handle` function is blocking. For this example we will be icrementing the "views" metric of a specific record. Assuming some sort of API (e.g. Prometheus) is available, the handler would look like:

```python
@shared_task
def views_handler(event):
    views.increment(event.recid)
```

Then, we need to configure the map so our handler gets executed when a record is "viewed" (i.e. `read`). Note that we are aware that a `read` operation happens also on REST API calls, which should not be counted as views but for the sake of simplicity we ignore it.

```
EVENT_HANDLERS = {
    RecordReadEvent.handling_key: [
        (views_handler, False),  # sync
        ...
    ]
}
```

**A heavy task case: Sending notifications**

Sending notifications, e.g. emails, is a heavy task and therefore the `handle` function is non-blocking. This means that it will delegate the core of the implementation to an asynchronous task.

```python
@shared_task
def notification_handler(event)
    user = User.get_user(event.user_id)
    send_mail(
        user.email,
        f"Thanks for publishing your record {event.recid}!"
    )
```

Then, like in the previous example the last part is to configure the handler to be called for the _record published_ event.

```
EVENT_HANDLERS = {
    RecordPublishedEvent.handling_key: [
        notification_handler,  # async by default
        ...
    ]
}
```

## How we teach this

This pattern will be implemented in core `invenio-records-resources` and as a consequence it will be propagated to modules that use it as base (e.g. `invenio-drafts-resources`, `invenio-rdm-records`).

## Drawbacks

Enabling _pickle_ on all queues might become a security concern.

## Alternatives

**To events**

Python `signals`. However, they are unpredictable since there is no easy way of knowing _who_ is subscribed to a singal nor in which _order_ will subscriptors be executed.

**To events definition**

In this RFC we opted by defining the _action_ e.g. `RecordCreatedEvent` in the event class name, instead of having a generic `RecordEvent` with an `action` property. This was to ease the configuration and processing of the handlers.

In addition, if events should be heavy or light was discussed. Opting by light events, since it would be easier to add more information than to remove it.

_Heavy events_
A heavy event would contain enough information for a handler to replicate the fact, avoiding that way extra expensive operations (e.g. db query, permission checks, etc.).

```python
@dataclass
class RecordCreatedEvent(RecordEvent):
    created: datetime
    updated: datetime
    recid: str
    revision_id: int
    metadata: dict
    access: dict
    ...
```

On the other hand, heavy events might suppose a challenge in memory terms to the queues. In addition, if they are sent to another system (e.g. Elasticsearch) for example for audit logging, they would require the implementation of post processing logic to anonymize.

_Light events_
A light event would contain only the minimum information needed for a handler to achieve it's goal, but it will require service calls (i.e. DB queries, permission checks, etc.).

```python
@dataclass
class RecordCreatedEvent(RecordEvent):
    recid: str
```

Light events might allow for easier composition, e.g. add request context information to a `RecordReadEvent` if it happend on the Web UI.

**To configuration loading**

Several options were taken into account to allow the customization of handlers. Both entrypoints and class configurations (e.g. like `ServiceConfig`) were discarded because of the complexity they add, while all that is looked for here is a simple way to establish a mapping between events and handlers (a _dict_).

**To event worker task**

Two alternatives were considered to the event worker task are:

- Delegating the handling to the event bus itself. However, this would put even more responsability on `invenio-rdm-records`, which is not desired at the moment.
- Having a new app run with for example a `supervisord`. This would increase the maintainance cost significantly to provide scalability, resilience, etc.

## Unresolved questions

**How to add events that require request context**

Is this suitable for cases where request context is required? For example usage statistics would require IP addr, referer, User Agent among others. These are available at _resource_ level, how can an event be added/queued from there?

**How to deal with cases that do not require a UoW**

For example, record `read` operations do not require a _unit of work_ and therefore it is not possible add events such as `RecordReadEvent`.

**Error handling**

Error handling plays a bit role in this event bus implementation. While there are mechanisms to potentially allow persistance of events there are no mechanisms to:

- Retry errored handlers.
- Resume event handling in case of an application crash (e.g. imagine the application is killed when only half of the events were handled).

**Refactoring of existing code**

There are actions such as ES indexing that currently happen within the _unit of work_ commit operation. However, if the indexing fails there is are no means to retry it making this operation somehow _not atomic_. Making ES indexing an event handler would solve this problems (assuming error handling is taken care of, see previous point).

## Resources/Timeline

TBD @Alex @Lars
