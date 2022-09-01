- Start Date: 2022-06-13
- RFC PR: [#63](https://github.com/inveniosoftware/rfcs/pull/63)
- Authors: David Eckhard, Zacharias Zacharodimos, Alex Ioannidis

# Notification System

## Summary

Implementation details for a notification system to be used in InvenioRDM.

## Motivation

As the product is getting larger and new features are added, it is hard to keep an overview of everything going on as a user. With the communities feature, certain tasks require an action to be executed by the user (f.e. accepting an incoming request for a community). This incoming request can easily get "lost", if a user does not explicitly check the community dashboard. Therefore, a notification system will not only solve this problem (and similar ones) but also increase user engagement by advertising the repository some more.

It will take care of sending the notifications based on the configuration and user preferences (i.e. if the repository disables certain notifications or a user does not want to receive certain notifications, they should not be sent).
In a first iteration, the user preferences may be neglected. They have to be specified and implemented first or the recipients function will have to take care of it explicitly.

### Use Cases
- Send notifications (see [table](#notifications))
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
    
- notification events
- data payloads involved around a notification
- delivery methods/backends for sending notifications to end users
- configuration for connecting notifications to backends
- notification management

### Event
Events are purely information based classes. These shall uniform the naming and ease the configuration process and readability.
    
```python
@dataclass
class Event:
    """Base event."""
    action: str
    type: str
    handling_key: str = f"{type}.{action}"

@dataclass
class CommunitySubmissionEvent(Event):
    """Community related events."""

    type: ClassVar[str] = "community-submission"
    handling_key: ClassVar[str] = "community-submission"

@dataclass
class CommunitySubmissionSubmittedEvent(CommunitySubmissionEvent):
    """Record related events."""

    action: ClassVar[str] = "submitted"
    handling_key: ClassVar[str] = f"{CommunitySubmissionEvent.type}.{action}"
```
    
### Notification
A notification shall capture all the necessary information for the notification system. It shall have a `dumps` method, which will make it possible to be passed around in celery tasks.


```python
@dataclass
class Notification:
    """Notification class."""

    type: str  # one of the registered notification types e.g comment_edit, new_invitation etc
    data: dict  # depending on the type. dump of a record, community, etc.
    recipients: List[dict]  # list of user dumps
    trigger: dict  # info about who triggered (and dump thereof), if it was manual or automatic 
    timestamp: datetime  # when the action happened
    
    def dumps(self):
        """Make it serializable for passing around via celery."""
        d = {k:v for k,v in self.items()}
        d.update(self.__dict__)
        return d
```

### Notification Backend

A notification backend will receive all the information from the notification system through a notification (i.e. type of action/event, user who triggered,  recipients, community/record (based on action/event)) to construct a suitable notification. The backend itself should *not* have to perform any queries (ElasticSearch or database). Its only purpose should be to take the received information, create the payload for the notification and send it to the recipients.
    
    
    
```python
```

```python
from abc import ABC, abstractmethod
 
class NotificationBackend(ABC):
    id = None
    """Unique id of the backend."""

    def __init__(self, notification_manager: NotificationManager):
        self.notification_manager = notification_manager
        self.notification_manager.register(self)
    
    @abstractmethod
    def send_notification(self):
        """Here each concrete implementation will be able to consume the notification message."""
        raise NotImplementedError()
```

For example, if a user wants to consume the message via an email notificaiton they will have to implement something similar to below:

A general jinja template loader should make the life easier in the long run.
```python
class JinjaTemplateLoaderMixin():
    """Used only in NotificationBackend classes."""
    
    template_folder = 'notifications'
    
    def _load_template(path):
        try:
            template = current_app.jinja_env.get_template(path)
        except TemplateNotFoundError:
            template = None

        return template


    def get_template(self, name):
        # e.g /notifications/email/comment_edit.html
        base_template = load_template(f"{template_folder}/{name}")
        specific_template = load_template(f"{template_folder}/{self.id}/{name}")

        if not specific_template and not base_template:
            raise TemplateNotFoundError()

        return specific_template or base_template

```

Now the email backend can get the templates in html and plaintext and render it accordingly.

```python
from invenio_mail.tasks import send_email

class EmailNotificationBackend(NotificationBackend, JinjaTempateLoaderMixin):
    
    id = 'email'
    
    def _send_email(notification, template_html, template_txt):
        mail_data = {}
        mail_data["recipients"] = [r["email"] for r in notification.recipients]
        mail_data["html_body"] = template_html.render(notification)
        mail_data["html_plain"] = template_txt.render(notification)
        ...
        send_email(mail_data)

    @shared_task
    def send_notification(notification):
        """Here each concrete implementation will be able to consume the notification message."""
        notification_tpl_html = self.get_template(notification.type + ".html")
        notification_tpl_txt = self.get_template(notification.type + ".txt")
        self._send_email.apply_async(args=[notification, template_html, template_txt])
```

A basic text notification backend might have their own templates or simply construct a message on the fly. This is very specific to the backend.

```python
class TextNotificationBackend(NotificationBackend):
    """Send notification as a mobile text."""
    
    id = 'text'

    def _send_text(notification_str):
        send_sms(notification_str)
        pass
    
    @shared_task
    def send_notification(self, notification):
        """Here each concrete implementation will be able to consume the notification message."""
        notification_str = self.serialize(notification)
        self._send_text(notification_str)
```


### Notification Manager
Notification manager shall take care of (de)registering backends and start tasks of sending notifications. It will also set the recipients of the notification by calling the recipients function.

```python
class NotificationManager:

    def __init__(self, config):
        self._config = config
        self._backends = dict<str, NotificationBackend>
        self._notification_policy = config.notification_policy

    def _backend_exists(self, backend_id):
        """Check if backend is registered."""
        return backend_id in self._backends


    def validate_policy(self):
        """Validate policy to make sure no backend is used without being specified."""
        for event_type, event_policy in self._notification_policy.items():
            if not backend_config.get("default_recipients"):
                raise NotificationError("Default recipients function must be defined")

            backend_configs = event_policy.get("backends", [])
            for backend_config in backend_configs:
                backend_cls = backend_config.get("cls")
                if not backend_cls:
                    raise NotificationError("Backend_cls must be defined")
                backend_id = backend_config.get("cls").id
                if not self._backend_exists(backend_id):
                    raise NotificationBackendNotFoundError(backend_id, type_=event_type)

                
    def _dispatch_notification(self, notification, backend, **kwargs):
        """Start task for sending."""
        try:
            backend.send_notification.apply_async(
                args=[notification.dumps()],
            )

        except Exception as e:
            current_app.logger.warning(e)
            raise e

    @shared_task
    def broadcast(self, notification):
        """Broadcast notification to backends according to event policy."""
        event_policy = self._notification_policy.get(notification.type)
        if event_policy is None:
            current_app.logger.warning(NotificationTypeNotFoundError(notification.type))
            return

        for backend_config in event_policy.get("backends", []):
            backend_id = backend_config["cls"]
            backend = self._backends.get(backend_id)
            if backend is None:
                current_app.logger.warning(NotificationBackendNotFoundError(backend_id, type_=notification.type))
                continue

            _recipients_func = backend_config.get("recipients") if backend_config.get("recipients") else event_policy["default_recipients"](notification) 
            notification.recipients = _recipients_func(notification)
      
            self._dispatch_notification(notification, backend)

    @shared_task
    def notify(self, notification, backend_id):
        """Set notification and notify specific backend.

        Will pass the key for the specific backend to notify.
        """
        backend = self._backends.get(backend_id)
        if backend is None:
            current_app.logger.warning(NotificationBackendNotFoundError(backend_id))
            return

        # TODO: Fetch recipients based on notification.type or let caller set them?
        self._dispatch_notification(notification, backend)

    def register(self, backend):
        """Register backend in manager."""
        if self._backend_exists(backend.id):
            raise NotificationBackendAlreadyRegisteredError(backend.id)
        self._backends[backend.id] = backend

    def deregister(self, backend):
        """Deregister backend in manager."""
        del self._backends[backend.id]
```
    
The broadcast and notify functions are then available as tasks

```python
# tasks.py
@shared_task
def broadcast(notification):
    """Broadcast notification to backends according to event policy."""
    current_notifications_manager.broadcast(notification)

@shared_task
def notify(notification, backend_id):
    """Set notification and notify specific backend.

    Will pass the key for the specific backend to notify.
    """
    current_notifications_manager.notify(notification, backend_id)

```


### Configuration
    
Making the provided functionality configurable is a key part. The available notification backends as well as the notification policy can be defined. Every entry in the policy contains the event as key and an event policy as value.
Each event policy must define a function for the default recipients. Additionally, backends for an event may be defined, which can also override the recipients function.
    
These config variables will then be loaded by the manager and validated.


```python
    
# config.py
    
class NotificationConfig(ConfiguratorMixin):
    backends = FromConfig("NOTIFICATIONS_BACKENDS", [])
    notification_policy = FromConfig("NOTIFICATIONS_POLICY", {})


# NOTIFICATIONS_CONFIG = NotificationConfig

NOTIFICATIONS_BACKENDS = [
    EmailNotificationBackend,
]

NOTIFICATIONS_DEFAULT_SUBJECT = _("New notification from repository")


NOTIFICATIONS_POLICY = {
    CommunitySubmissionSubmittedEvent.handling_key: {
        'default_recipients': lambda notification: [],
        'backends': [
            {
                'cls': EmailNotificationBackend,
                'recipients': lambda notification: [],
            },
        ],
    },
    CommunitySubmissionCreatedEvent.handling_key: {
        'default_recipients': lambda notification: some_func(notification),
        'backends': [
            {
                'cls': TextNotificationBackend,
            },
            {
                'cls': TextNotificationBackend,
            },
        ],
    },
    ...
}  
```

Each module that triggers events (via service calls), should also expose a default notifications policy config for these events.
    
All these configurations come together in invenio_app_rdm.config or invenio.cfg.
```python
# invenio_communities config.py
COMMUNITY_NOTIFICATIONS_POLICY = {
    CommunitySubmissionSubmittedEvent.handling_key: {
        'default_recipients': lambda notification: [],
        'backends': [
            {
                'cls': EmailNotificationBackend,
                'recipients': lambda notification: [],
            },
        ],
    },
}
    

# invenio.cfg
from invenio_communities.config import COMMUNITY_NOTIFICATIONS_BACKENDS, COMMUNITY_NOTIFICATIONS_POLICY
from other_module.config import OTHER_MODULE_NOTIFICATIONS_POLICY
    
NOTIFICATIONS_BACKENDS = {
    **COMMUNITY_NOTIFICATIONS_BACKENDS,
}
NOTIFICATIONS_POLICY = {
    **COMMUNITY_NOTIFICATIONS_POLICY,
    **OTHER_MODULE_NOTIFICATIONS_POLICY,
}
```

### Registration/Initialization


```python

# ext.py

cfg = NotificationConfig.build(app)
        manager = NotificationManager(
            config=cfg,
        )
        for id, backend_cls in cfg.backends.items():
            manager.register(backend_cls())
        
        manager.validate_policy()
        self.manager = manager


# actions.py

my_notification = Notification(
    data={},
    notification_type='comment_create',
)

# if user wants to notify all registered backends
current_notification.broadcast(my_notification)

# if user wants to use specific backend
current_notification.notify(
    notification=my_notification, 'email')

```


#### Notifications
| Action                       | Who triggers                 | Who receives           |
|------------------------------|------------------------------|------------------------|
| Comment create      | Creating user    | All involved in conversation                  |
| Comment edit        | Editing user     | All involved in conversation                  |
| Comment delete      | Deleting user    | All involved in conversation                  |
|       |     |                  |
| Submission create      | Creating user     | Community owners/managers                 |
| Submission accept      | Accepting user    | User who created submission               |
| Submission decline     | Declining user    | User who created submission               |
| Submission edit        | Editing user      | Either user who created submission or Community owners/managers (depends on who edited)                 |
|       |     |                  |
| Invitation accept      | Accepting user    | User who created invitation               |
| Invitation decline     | Declining user    | User who created invitation               |
| Invitation expire      | System            | User who created invitation               |


## Example

> Show a concrete example of what the RFC implies. This will make the consequences of adopting the RFC much clearer.

> As with other sections, use it if it makes sense for your RFC.

## How we teach this

> What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing Invenio patterns, or as a wholly new one?

> Would the acceptance of this proposal mean the Invenio documentation must be re-organized or altered? Does it change how Invenio is taught to new users at any level?

> How should this feature be introduced and taught to existing Invenio users?

This should probably become a module of its own, as it is kept very general without any specific requirements for other invenio modules (except celery for starting tasks).

Other modules with the need of sending notifications can then import it.
The config should happen in `invenio-app-rdm`, as it knows all the underlying modules and events going on.
    
## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Invenio, on the integration of this feature with other existing and planned features, on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

With regards to the probable implementation of the event bus, it is not clear how event types and actions will be implemented/named and it may cause a refactoring.

Current service methods will have to be adapted to broadcast notifications.

    
## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

An alternative is to rely on the implementation of the event bus and register methods. Though it is not known which data will be passed along for the events and there may be some missing data for the notification system.
    
If we do not implement this, each service method with the need of notifiying certain users, will have to take care of putting together a proper notification content, render it, and figure out on which channel to reach the users.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still TBD? Use it as a todo list for the RFC.
    
Tracking success/failure of delivery is not taken care of. If a task gets stuck or dies, there will not be any mention of this.
## Resources/Timeline

> Which resources do you have available to implement this RFC and what is the overall timeline?

Discussion and implemenation during sprint for v9.1
