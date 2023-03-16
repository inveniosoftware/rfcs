- Start Date: 2022-06-13
- RFC PR: [#63](https://github.com/inveniosoftware/rfcs/pull/63)
- Authors: David Eckhard, Zacharias Zacharodimos, Alex Ioannidis

# Notification System

## Summary

Implementation details for a notification system to be used in InvenioRDM.

## Motivation

As the product is getting larger and new features are added, it is hard to keep an overview of everything going on as a user. With the communities feature, certain tasks require an action to be executed by the user (f.e. accepting an incoming request for a community). This incoming request can easily get "lost", if a user does not explicitly check the community dashboard. Therefore, a notification system will not only solve this problem (and similar ones) but also increase user engagement by advertising the repository some more.

It will take care of sending the notifications based on the configuration and user preferences (i.e. if the repository disables certain notifications or a user does not want to receive certain notifications, they should not be sent).

### Use Case

- Send notifications for operations involving multiple users (see [table](#notification-examples))
- Overridable notification templates
  - One template for each notification
  - Can be overridden via theme (like other templates)
  - Add new templates for custom notifications
  - Should be translatable
    - Store preferred language in user profile
- Modify notifications
  - Set which notifications should be send
    - As a repository admin activate/deactive certain notifications
    - As a user (user profile)
  - Define notifications for events per module
  - Define notification backends per module

## Detailed design

In this design, we describe:

- New `invenio-notifications` module
    - Simple self-contained API that is business-logic agnostic
    - Minimal dependencies, only for supporting different notification transfer backends (email, chat, etc.)
    - Defines a common notification payload/datamodel
- Utilities for configuration on services for building notifications payloads (receivers, data payload)
    - Generates the list of recipients
    - Takes into account user preferences, e.g.
        - if they want to receive notifications at all
        - what kind of notifications the want to receive

### Datamodel/payload

- Acts as a "client payload"
- Keeps all the relevant data necessary for a notification to be dispatched via a backend
- It must be JSON-serializeable so that it can be passed to Celery tasks.

```python
@dataclass
class Notification:
    """Notification datamodel."""

    # Unique type identifier for the notification
    type: str

    # Actual subject/payload of the notification's objects, e.g. record,
    # community, request, etc.
    context: dict

    # Recipients list
    recipients: list[Recipient]

    # Creation time of the notification
    timestamp: datetime


@dataclass
class Recipient:
    """Recipient datamodel."""

    # Recipient information (e.g. expanded user/group)
    data: dict

    # The "rendering" template to be used for this recipient
    template: str

    # Backend delivery options/config
    backends: list[Backend]


@dataclass
class Backend:
    """Backend delivery datamodel."""

    # Unique backend identifier ("email", "slack", "cern-notification")
    id: str

    # Options/parameters for the backend to deliver the notification
    params: dict
```

### API

A Notification manager/client will be used for sending notifications as a Celery task.

```python
class NotificationManager:

    def __init__(self, backends):
        self.backends = backends

    def validate(self, notification):
        ...

    # Client
    def broadcast(self, notification: Notification, eager=False):
        """Broadcast a notification via a Celery task."""
        self.validate(notification)
        task = broadcast_notification.si(notification)
        return task.apply() if eager else task.delay()

    # Consumer
    def handle(self, notification: Notification):
        """Handle a notification."""
        for recipient in notification.recipients:
            for backend in recipient.backends:
                dispatch_notification(backend, recipient.data, notification.data)
```

The `broadcast_notification` and `dispatch_notification` tasks will be handling retries:

```python
# tasks.py
@shared_task
def broadcast(notification: Notification):
    """Broadcast notification to backends according to event policy."""
    current_notifications_manager.broadcast(notification)

@shared_task
def notify(notification, backend_id):
    """Set notification and notify specific backend.

    Will pass the key for the specific backend to notify.
    """
    current_notifications_manager.notify(notification, backend_id)
```

The final usage looks like the following:

```python
from invenio_notifications import current_notifications

notification = ...  # see "Building notifications"
current_notifications.broadcast(notification)
```

### Backends

A notification backend will receive all the information from the notification system through a notification (i.e. type of action/event, user who triggered,  recipients, community/record (based on action/event)) to construct a suitable notification. The backend itself should *not* have to perform any queries (ElasticSearch or database). Its only purpose should be to take the received information, create the payload for the notification and send it to the recipients.

```python
from abc import ABC, abstractmethod

class Backend(ABC):

    id = None
    """Unique ID of the backend."""

    @abstractmethod
    def send(self, notification, recipient, params):
        """Here each concrete implementation will be able to consume the notification message."""
        raise NotImplementedError()

    @abstractmethod
    def validate(self, notification, recipient, params):
        """Validate notification message."""
        raise NotImplementedError()
```

For example, if a user wants to consume the message via an email notification they will have to implement something similar to below:


```python
from invenio_mail.tasks import send_email

class EmailBackend(Backend):

    id = "email"

    def send(self, notification: Notification, recipient: Recipient, params: Backend):
        """Mail sending implementation."""
        mail_data = {}
        ctx = {
            "notification": notification,
            "recipient": recipient,
        }

        mail_data["recipients"] = [recipient["email"]]
        template = get_template(recipient.template)  # see "Templating" below
        mail_data["html_body"] = template_html.render(**ctx)
        mail_data["plain_body"] = template_txt.render(**ctx)
        resp = send_email(mail_data)
        return resp  # TODO: what would a "delivery" result be
```

#### Templating

A generic Jinja template loader should make life easier for backends to integrate templates and take into account locale:

```python
class JinjaTemplateLoaderMixin:
    """Used only in NotificationBackend classes."""

    template_folder = 'notifications'

    def _load_template(path):
        try:
            template = current_app.jinja_env.get_template(path)
        except TemplateNotFoundError:
            template = None
        return template

    def get_template(self, recipient: Recipient):
        # e.g /notifications/email/comment_edit.html
        base_template = self._load_template(f"{template_folder}/{recipient.template}")
        specific_template = self._load_template(f"{template_folder}/{self.id}/{recipient.template}")

        template = specific_template or base_template
        if not template:
            raise TemplateNotFoundError()

        # Take locale into account
        locale = recipient.data.get("locale")
        if locale != "en":
            template += f".{locale}"

        return template
```

### Building notifications

The API described above, accepts a `Notification` object as its main payload. Generating this object is something that is left to the client, which in our case is usually a service method. We want to provide a set of utilities that will streamline this process and make it configurable.

Another consideration is that different parts of the platform contribute different business rules for:

- types of notifications and their templates
- the set of recipients for a notification
- individual notification preferences for each recipient
- what notification backends (delivery methods) should be used for a recipient

To encapsulate all this functionality we're introducing a set of classes that can be composed to build individual notification builders for each type of notification.

```python
class NotificationBuilder:

    payload: list[PayloadGenerator]
    recipients: list[RecipientGenerator]
    backends: list[BackendGenerator]

    def build(context: dict) -> Notification:
        ...
```

Notification builder implementation for community record inclusion request:

```python
# invenio_requests/notifications.py
class CommunityRecordInclusionNotificationBuilder(NotificationBuilder):

    payload = [
        RequestPayload,
        CommunityPayload,
        RecordPayload,
    ]

    recipients = [
        RequestRecipients,
        CommunityRecipients,
        UserRecipient,
    ]

    backends = [
        UserEmailBackend,
    ]

#
# Payload
#
# invenio_requests/notifications.py
class RequestPayload:
    def __call__(self, request=None, **kwargs):
        return {"request": request.to_dict()}

# invenio_rdm_records/notifications.py
class RecordPayload:
    def __call__(self, record=None, **kwargs):
        return {"record": record.to_dict()},

# invenio_communities/notifications.py
class CommunityPayload:
    def __call__(self, community=None, **kwargs):
        return {"community": community.to_dict()}

#
# Recipients
#
# invenio_requests/notifications.py
class RequestRecipients:
    def __init__(self, include_creator=True, include_receiver=True):
        self.include_creator = include_creator
        self.include_receiver = include_receiver

    def __call__(self, notification, recipients: list):
        ret = []
        if self.include_creator:
            ret.append(request.created_by.resolve())
        if self.include_receiver:
            ret.append(request.receiver.resolve())
        return ret

# invenio_communities/notifications.py
class CommunityRecipients:
    def __init__(self, roles=None):
        self.roles = roles

    def __call___(self, notification, recipients: list):
        ret = []
        for rec in recipients:
            if isinstance(rec, Community):
                comm = rec
                members = Member.get_members(comm.id)
                for m in members:
                    if not m.user_id:
                        continue
                    if self.roles and m.role not in self.roles:
                        continue
                    user = m.relations.user.dereference()
                    notif_pref = user.preferences["notifications"]
                    com_pref = notif_pref.get(comm.id)
                    if com_pref:
                        # check if notification is enabled/disabled for the member
                        # if ...:
                        #     continue
                    ret.append(user)
            else:
                ret.append(rec)
        return ret

# invenio_users_resources/notifications.py
class UserRecipient:
    def __call__(self, notification, recipients: list)
        ret = []
        for rec in recipients:
            if isinstance(rec, User):
                if rec.preferences["notifications"]["enabled"]
                    ret.append(rec.dump())
            else:
                ret.append(rec)
        return ret

#
# Backends
#
# invenio_users_resources/notifications.py
class UserEmailBackend:
    def __call__(self, notification, recipients: list)
        for rec in recipients:
            user = rec.data
            rec.backends.append({
                "backend": "email",
                "to": f"{user.profile.full_name} <{user.email}>",
            })

#
# Usage in a service method
#
uow.register(NotificationOp(CommunityRecordInclusionNotificationBuilder.build(
    {
        "request": request,
        "community": community,
        "record": record,
    }
)))
```

#### Notification examples

| Action             | Who triggers   | Who receives                                                                            |
| ------------------ | -------------- | --------------------------------------------------------------------------------------- |
| Comment create     | Creating user  | All involved in conversation                                                            |
| Comment edit       | Editing user   | All involved in conversation                                                            |
| Comment delete     | Deleting user  | All involved in conversation                                                            |
|                    |                |                                                                                         |
| Submission create  | Creating user  | Community owners/managers                                                               |
| Submission accept  | Accepting user | User who created submission                                                             |
| Submission decline | Declining user | User who created submission                                                             |
| Submission edit    | Editing user   | Either user who created submission or Community owners/managers (depends on who edited) |
|                    |                |                                                                                         |
| Invitation accept  | Accepting user | User who created invitation                                                             |
| Invitation decline | Declining user | User who created invitation                                                             |
| Invitation expire  | System         | User who created invitation                                                             |

## Example

> Show a concrete example of what the RFC implies. This will make the consequences of adopting the RFC much clearer.

> As with other sections, use it if it makes sense for your RFC.

## How we teach this

> What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing Invenio patterns, or as a wholly new one?

> Would the acceptance of this proposal mean the Invenio documentation must be re-organized or altered? Does it change how Invenio is taught to new users at any level?

> How should this feature be introduced and taught to existing Invenio users?

## Alternatives

An alternative is to rely on the implementation of the event bus and register methods. Though it is not known which data will be passed along for the events and there may be some missing data for the notification system.

## Unresolved questions

Tracking success/failure of delivery is not taken care of. If a task gets stuck or dies, there will not be any mention of this.

## Resources/Timeline

Discussion and implemenation during sprint for MS5.
