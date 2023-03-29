- Start Date: 2022-06-13
- RFC PR: [#63](https://github.com/inveniosoftware/rfcs/pull/63)
- Authors: David Eckhard, Zacharias Zacharodimos, Alex Ioannidis

# Notification System

## Summary

Implementation details for a notification system to be used in InvenioRDM.

## Motivation

As the product is getting larger and new features are added, it is hard to keep an overview of everything going on as a user. With the communities feature, certain tasks require an action to be executed by the user (f.e. accepting an inclusion request for a community). These actions can easily be overlooked, if a user does not explicitly check the community dashboard. Therefore, a notification system will not only solve this problem (and similar ones) but also increase user engagement by advertising the repository some more.

We need a system that will take care of sending the notifications based on instance configuration and user preferences (i.e. if the repository disables certain notifications or a user does not want to receive certain notifications, they should not be sent).

### Use cases

- As a developer, I want to send notifications after various operations in the system, so that a set of users can be informed about them.
- As a user, I want to receive notifications for events in the system that require my attention, so that I can take action (or just be informed).
  - Note: this is a very generic statement, supplemented by the ["Notification examples" table](#notification-examples) below.
- As a user, I want to be able to completely disable notifications, so that I don't get spammed by notifications from the instance.
- As a user, I want to select different levels of notificiations, so that I only receive notifications for what I consider important.
- As a user, I want to receive notifications in my preferred locale, so that I can read them with ease.
- As an instance administrator, I want to override the notification templates that are sent, so that I can personalize and control the exact wording communicated to users.

#### Notification examples

| Action             | Who triggers   | Who receives                                                                            |
| ------------------ | -------------- | --------------------------------------------------------------------------------------- |
| Comment create     | Creating user  | All involved in conversation                                                            |
|                    |                |                                                                                         |
| Submission create  | Creating user  | Community owners/managers                                                               |
| Submission accept  | Accepting user | User who created submission (if not same as the user triggering)                        |
| Submission decline | Declining user | User who created submission (if not same as the user triggering)                        |
|                    |                |                                                                                         |
| Invitation accept  | Accepting user | User who created invitation                                                             |
| Invitation decline | Declining user | User who created invitation                                                             |
| Invitation expire  | System         | User who created invitation                                                             |

## Detailed design

In this design, we describe a new `invenio-notifications` module based on the following:

- Simple "client" for triggering notifications from the service layer with low
- Notification configuration helpers for:
    - Resolving notification entities (e.g. requests, communities, users), via calls to the **service layer**
    - Generating the list of recipients for a notification, via calls to the **service layer**
    - Filtering recipients by taking into account user preferences (individual and as community members)
    - Binding recipients to notification delivery backends (email, chat, etc.) based on their preferences
- Common interface for implementing different notification delivery backends (email, chat, etc.)
- Templating interface for:
    - Rendering a message's subject and HTML/plaintext/Markdown body
    - Provide default templates for each notification type
    - Allow extending/customizing templates based on notification type, backend, and locale

### Notification manager

A Notification manager/client will be used for sending notifications as a Celery task.

```python
class NotificationManager:

    def __init__(self, backends: Dict[str, Backend], builders: Dict[str, Builder]):
        self.backends = backends  # via config "NOTIFICATIONS_BACKENDS"
        self.builders = builders  # via config "NOTIFICATIONS_BUILDERS"

    def validate(self, notification: Notification):
        """Validate notification object."""
        # Validate the type
        ...
        # Validate context (if possible)
        ...

    # Client
    def broadcast(self, notification: Notification, eager: bool=False):
        """Broadcast a notification via a Celery task."""
        self.validate(notification)
        task = broadcast_notification.si(notification)
        return task.apply() if eager else task.delay()

    # Consumer
    def handle_broadcast(self, notification: Notification):
        """Handle a notification broadcast."""
        builder = self.builders[notification.type]
        # Resolve and expand entities
        builder.resolve_context(notification)
        # Generate recipients
        recipients: list[Recipient] = builder.build_recipients(notification)
        builder.filter_recipients(notification, recipients)
        for recipient in recipients:
            recipient_backends = builder.build_recipient_backends(notification, recipient)
            for backend in recipient_backends:
                dispatch_notification.delay(backend, recipient.data, notification.data)

    # Dispatch delivery/sending via a backend
    def handle_dispatch(self, backend: Backend, recipient: Recipient, notification: Notification):
        """Handle a backend dispatch."""
        self.backends[backend.id].send(backend, notification, recipient)
```

The `broadcast_notification` and `dispatch_notification` tasks will be handling retries:

```python
# tasks.py
@shared_task
def broadcast_notification(notification: Notification):
    """Handles a notification broadcast."""
    current_notifications_manager.handle_broadcast(notification)

@shared_task(max_retries=5, default_retry_delay=5 * 60)
def dispatch_notification(backend, recipient, notification):
    """Dispatches a notification to a recipient for a specific backend."""
    current_notifications_manager.handle_dispatch(backend, recipient, notification)
```

### Building notifications

#### Datamodel

- Keeps all the relevant data necessary for a notification to be dispatched via a backend
- It must be JSON-serializeable so that it can be passed to Celery tasks.

```python
@dataclass
class Notification:

    # Unique type identifier for the notification
    type: str  # request_comment, review_submit, review_accept, invitation_create, etc.

    # Contextual information for the notification
    context: dict

@dataclass
class Recipient:

    #
    data: dict
```

The APIs described above, accept a `Notification` object as their main payload. Generating this requires two fields:

- `type`: The unique type of the notification. This key is used for configuration lookup and templating. Examples: `submission_create`, `request_comment`, `invitation_accepted`, etc.
- `context`: Contextual data relevant to the notification.

is something that is left to the client, which in our case is usually a service method. We want to provide a set of utilities that will streamline this process and make it configurable.

Another consideration is that different parts of the platform contribute different business rules for:

- types of notifications and their templates
- the set of recipients for a notification
- individual notification preferences for each recipient
- what notification backends (delivery methods) should be used for a recipient

To encapsulate all this functionality we're introducing a set of classes that can be composed to build individual notification builders for each type of notification.

```python
class NotificationBuilder:

    type: str = None  # notification type ID

    context: list[ContextGenerator]
    recipients: list[RecipientGenerator]
    recipient_filters: list[RecipientFilter]
    recipient_backends: list[BackendGenerator]

    @classmethod
    @abstractmethod
    def build(cls, **kwargs) -> Notification:
        ...

    @classmethod
    def resolve_context(cls, notification: Notification) -> Notification:
        """Resolve all references in the notification context."""
        for ctx_func in cls.context:
            # NOTE: We assume that the notification is mutable and modified in-place
            ctx_func(notification)
        return notification

    @classmethod
    def build_recipients(cls, notification: Notification) -> Dict[str, Recipient]:
        """Return a dictionary of unique recipients for the notification."""
        recipients = {}
        for recipient_func in cls.recipients:
            recipient_func(notification, recipients)
        return recipients

    @classmethod
    def filter_recipients(cls, notification: Notification, recipients: Dict[str, Recipient]) -> Dict[str, Recipient]:
        """Apply filters to the recipients."""
        for recipient_filter_func in cls.recipient_filters:
            recipient_filter_func(notification, recipients)
        return recipients

    @classmethod
    def build_recipient_backends(cls, notification: Notification, recipient: Recipient) -> list[str]
        """Return the backends for recipient."""
        backends = []
        for recipient_backend_func in cls.recipient_backends:
            recipient_backend_func(notification, recipients)
        return backends
```

Notification builder implementation for community record inclusion request submit:

```python
# invenio_rdm_records/notifications.py
class CommunityRecordInclusionSubmitNotificationBuilder(NotificationBuilder):

    type = "submission_created"

    def build(cls, request) -> Notification:
        return Notification(
            type=cls.type,
            context=EntityResolverRegistry.reference_entity(request),
        )

    context = [
        EntityResolve(key="request"),
        EntityResolve(key="request.created_by"),
        EntityResolve(key="request.topic"),
        EntityResolve(key="request.receiver"),
    ]

    recipients = [
        CommunityMembersRecipient(key="request.receiver", roles=["curator", "owner"]),
        UserRecipient(key="request.created_by"),
    ]

    recipient_filters = [

    ]

    recipient_backends = [
        UserEmailBackend(),
    ]


# invenio_requests/notifications.py
class RequestCommentNotificationBuilder(NotificationBuilder):

    context = [
        EntityResolve(key="request_event", resolver_id="request_event"),  # comment
        EntityResolve(key="request_event.created_by"),  # comment creator
        EntityResolve(key="request_event.request"),  # request where the comment was added
        EntityResolve(key="request_event.request.created_by"),  # creator of the request
        EntityResolve(key="request_event.request.topic"),  # topic of the request
        EntityResolve(key="request_event.request.receiver"),
    ]

    recipients = [
        CommunityMembersRecipient(key="request.receiver", roles=["curator", "owner"]),
        # UserRecipient(key="request.created_by"),
    ]

    recipient_backends = [
        UserEmailBackend(),
    ]

#
# Context builder
#
class EntityResolve:

    def __init__(self, key):
        self.key = key

    def __call__(self, notification: Notification):
        entity_ref = dict_lookup(notification, self.key)
        entity = EntityResolverRegistry.resolve_entity(entity_ref)
        dict_set(notification, self.key, entity)
        return notification

#
# Recipients
#
# invenio_communities/notifications.py
class CommunityMembersRecipient:
    def __init__(self, key, roles=None):
        self.key = key
        self.roles = roles

    def __call___(self, notification, recipients: list):
        community = dict_lookup(notification, key)
        if isinstance(community, Community):
            members = CommunityMember.search(
                system_identity,
                community["id"],
                roles=self.roles,
            )
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
                recipients.append(user)
        return recipients

# invenio_users_resources/notifications.py
class UserRecipient:

    def __init__(self, key):
        self.key = key

    def __call__(self, notification, recipients: list[Recipient])
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
    def __call__(self, notification, recipients: list[Recipient])
        for rec in recipients:
            user = rec.data
            rec.backends.append({
                "backend": "email",
                "to": f"{user.profile.full_name} <{user.email}>",
            })

#
# Usage in a service method
#
uow.register(NotificationOp(CommunityRecordInclusionSubmitNotificationBuilder.build(request)))
```

### Backends

A notification backend will receive all the information from the notification system through `Notification` primitives (i.e. notification context/data, recipient, backend/delivery parameters) to perform the actual sending action. The backend itself should _not_ have to perform any queries (ElasticSearch or database) to resolve information. Its only purpose should be to take the received information and deliver the actual message.

```python
class Backend(ABC):

    id = None
    """Unique ID of the backend."""

    @abstractmethod
    def send(self, notification: Notification, recipient: Recipient, backend: ):
        """Send the notification message."""
        raise NotImplementedError()
```

For example, if a user wants to consume the message via an email notification they will have to implement something similar to:

```python
from invenio_mail.tasks import send_email

class EmailBackend(Backend):

    id = "email"

    def send(self, notification: Notification, recipient: Recipient):
        """Mail sending implementation."""
        content = self.render_template(notification, recipient)  # see "Templating" below
        resp = send_email({
            "subject": content["subject"],
            "html_body": content["html_body"],
            "plain_body": content["plain_body"],
            "recipients": [f"{recipient.name} <{recipient.email}>"],
            "sender": current_app.config["MAIL_DEFAULT_SENDER"],
        })
        return resp  # TODO: what would a "delivery" result be
```

#### Templating

A generic Jinja template loader should make life easier for backends to integrate templates and take into account locale:

```python
class JinjaTemplateLoaderMixin:
    """Used only in NotificationBackend classes."""

    template_folder = 'notifications'  # or from config?

    def render_template(self, notification: Notification, recipient: Recipient):
        # Take locale into account
        locale = recipient.data.get("locale")
        template = current_app.jinja_env.get_template([
            # Backend-specific templates first, e.g notifications/email/comment_edit.jinja
            f"{template_folder}/{self.id}/{notification.type}.{locale}.jinja"
            f"{template_folder}/{self.id}/{notification.type}.jinja"
            # Default templates, e.g notifications/comment_edit.jinja
            f"{template_folder}/{notification.type}.{locale}.jinja",
            f"{template_folder}/{notification.type}.jinja",
        ])
        ctx = tpl.new_context({
            "notification": notification,
            "recipient": recipient,
        })
        return {
            block: block_func(ctx)
            for block, block_func in template.blocks.items()
        }
```

The Jinja templates will include all parts of the notification that are subject to special formatting by a backend (e.g. subject, HTML/plaintext/markdown body, etc.) in separate Jinja blocks. We can establish  This will allow customizing and extending templates much easier, while at the

Here's an example Jinja template for a new record submission request to a community:

```jinja
{# notifications/submission_create.jinja #}

{%- block subject -%}
New record submission for your community {{ notification.request.community.name }} submitted by {{ notification.request.created_by.name }}
{%- endblock subject -%}

{%- block html_body -%}
<p>The record "{{ notification.request.record.title }}" was submitted to your community {{ notification.request.community.name }}, by {{ notification.created_by.name }}.</p>

<a href="{{ notification.request.links.self_html }}" class="button">Review the request</a>
{%- endblock html_body -%}

{%- block plain_body -%}
The record "{{ notification.request.record.title }}" was submitted to your community {{ notification.request.community.name }}, by {{ notification.created_by.name }}.

Review the request: {{ notification.request.links.self_html }}
{%- endblock plain_body -%}

{# Markdown for Slack/Mattermost/chat #}
{%- block md_body -%}
The record "{{ notification.request.record.title }}" was submitted to your community {{ notification.request.community.name }}, by {{ notification.created_by.name }}.

[Review the request]({{ notification.request.links.self_html }})
{%- endblock md_body -%}
```

### Dependencies tree

- Records-Resources (Entity resolvers base classes)
    - Requests
    - Users-Resources
    - Draft-Resources
        - RDM-Records
    - Communities
    - **Notifications**
        - Requests
        - Communities
        - RDM-Records
        - Users-Resources

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
