- Start Date: 2021-05-07
- RFC PR: [#49](https://github.com/inveniosoftware/rfcs/pull/49)
- Authors: Nicola Tarocco
- State: DRAFT

# Authentication

## Summary

This document describes the different authentication options in Invenio, focusing on local and external authentication.
The authentication mechanism of an external app via REST APIs (e.g. Single Page Application) is described in the dedicated RFC "Enable Single Page app authentication in Invenio".

**Definitions**

- Local authentication: user registers a new account in Invenio by providing email and password and (s)he will use such credentials to login.
- External authentication: user has already an existing account via an external provider (social accounts or organization-wide Single Sign On accounts). The Invenio installation is configured to use the external account authentication. On user login, it will create a local unmanaged account, linked to the remote account, without any password.

## Motivation

Below the main use cases that are handled:

- As a user, I want to create an account in the system and login with my email/password.
- As a user, I want to use a "social" account (e.g. GitHub and OAuth in general) or my organization authentication (for example SSO) to login, without registering a new account.
- As a user, when I login with my "social" account, I want to be able to disconnect it.
- As an admin, I want to configure the system to allow only local logins.
- As an admin, I want to configure the system to allow only external authentication.
- As an admin, I want to configure the system to allow both local logins and external authentication.
- As an admin, I want to easily define some policies to allow/deny access to users after the external authentication succeeded.
- As an admin, I want to customize the login page template.
- As an admin, I want to disable the possibility of editing the user profile when only external authentication enabled, given that the profile is fetched from the remote provider.
- As an admin, I want to enforce users, at first login, to accept specific terms of use.
- As an admin, I should be able to deactivate a specific user to deny access.
- As an admin, I want to define different user roles in my system based on organization group membership.

## Detailed design

In Invenio, local accounts and session management are provided by the `invenio-accounts` module, which relies on the popular `Flask-Security`, `Flask-Login` and `Flask-Principal` packages. A new user can create a new account in the system via a registration form: by default, the form contains email and password fields. The form can be configured to have different fields. Users are stored in the `accounts_user` table.
After submission a confirmation email is sent to the user with a secure link (containing a token) to confirm the email address. The account is then active and the user can login. The forgotten password feature allows to reset the password.

Along with local user authentication, administrators can also configure external authentication integration.

In such case, administrators might want to disable local logins and also automatically redirect users to the external authentication provider without showing the login page.
We propose to introduce a new config variable `ACCOUNTS_LOCAL_LOGIN_ENABLED = False` which will disable local logins, view template and endpoints. The login page will not display the username and password input fields and the various related endpoints (registration, account confirmation, password reset, etc.) will return an error if called.
We also propose to configure the automatic redirection to the external authentication provider with the config variable `OAUTHCLIENT_AUTO_REDIRECT_TO_EXTERNAL_LOGIN = True`. With this configuration enabled, when local login is disabled and only **1** external authentication provider is configured, the login page will be skipped and the user will be automatically redirected.

### External authentication

One of the main use cases of external authentication is certainly OAuth. The `invenio-oauthclient` module provides a set of OAuth plugins ready to use and also APIs to easily create new integrations.
It also provides integration with a Keycloak server for example when authentication if OpenID Connect.

For the existing integrations, the administrator will only have to enable the plugin and define the needed secrets.

OAuth ORCID example:

```python
from invenio_oauthclient.contrib import orcid

OAUTHCLIENT_REMOTE_APPS = dict(
   orcid=orcid.REMOTE_APP,
)

ORCID_APP_CREDENTIALS = dict(
   consumer_key="<CLIENT ID>",
   consumer_secret="<CLIENT SECRET>",
)
```

OpenID Connect:

```python
from invenio_oauthclient.contrib import keycloak as k

helper = k.KeycloakSettingsHelper(
    title="My organization",
    description="My organization authentication provider",
    base_url="http://yourkeycloakserver.com:8080",
    realm="invenio"
)

OAUTHCLIENT_REMOTE_APPS = dict(
    keycloak=helper.remote_app,
)

KEYCLOAK_APP_CREDENTIALS = dict(
    consumer_key='<CLIENT ID>',
    consumer_secret='<CLIENT SECRET>',
)
```

#### Multiple keycloak providers

The `KeycloakSettingsHelper` allows to configure multiple providers at the same time.
The `app_key` param allows to define a specific `APP_CREDENTIALS` config variable and the name of the app is used, as convention, to namespace config variables.

```python
foo = KeycloakSettingsHelper(
    title="Foo provider",
    description="Keycloak Authentication foo provider",
    base_url="https://foo.com:4430",
    realm="foo-realm"
    app_key="FOO_KEYCLOAK_APP_CREDENTIALS"
)

bar = KeycloakSettingsHelper(
    title="Bar provider",
    description="Keycloak Authentication bar provider",
    base_url="https://bar.com:4430",
    realm="bar-realm"
    app_key="BAR_KEYCLOAK_APP_CREDENTIALS"
)

OAUTHCLIENT_REMOTE_APPS = {
    "foo_keycloak": foo.remote_app,
    "bar_keycloak": bar.remote_app,
}

OAUTHCLIENT_FOO_KEYCLOAK_REALM_URL = foo.realm_url
OAUTHCLIENT_FOO_KEYCLOAK_USER_INFO_URL = foo.user_info_url
OAUTHCLIENT_FOO_KEYCLOAK_VERIFY_EXP = True
OAUTHCLIENT_FOO_KEYCLOAK_VERIFY_AUD = True
OAUTHCLIENT_FOO_KEYCLOAK_AUD = "<FOO.AUDIENCE>"

OAUTHCLIENT_BAR_KEYCLOAK_REALM_URL = bar.realm_url
OAUTHCLIENT_BAR_KEYCLOAK_USER_INFO_URL = bar.user_info_url
OAUTHCLIENT_BAR_KEYCLOAK_VERIFY_EXP = False
OAUTHCLIENT_BAR_KEYCLOAK_VERIFY_AUD = True
OAUTHCLIENT_BAR_KEYCLOAK_AUD = "<BAR.AUDIENCE>"

FOO_KEYCLOAK_APP_CREDENTIALS = {
    "consumer_key": "<FOO.CLIENT.ID>",
    "consumer_secret": "<FOO.CLIENT.CREDENTIALS.SECRET>",
}

BAR_KEYCLOAK_APP_CREDENTIALS = {
    "consumer_key": "<BAR.CLIENT.ID>",
    "consumer_secret": "<BAR.CLIENT.CREDENTIALS.SECRET>",
}
```

#### Other external authentications

SAML authentication is out-of-scope of this RFC. For more information, see the `invenio-saml` module.

#### New OAuth plugins

The creation of a new OAuth plugin can be done via the `OAuthSettingsHelper` class. The new plugin implementation will probably be very similar to any of the existing one, which can be used as example. The developer will have to:

1. Define a new class that inherits from `OAuthSettingsHelper` and eventually define some defaults for naming and URLs in the constructor. Then call the `super` constructor:

```python
class FooOAuthSettingsHelper(OAuthSettingsHelper):
    def __init__(...):
        base_url = "https://foo.com/oauth/"
        ...
        super().__init__(...)
```

2. Define any specific handler to correctly setup the account in Invenio at the first login and to fetch account info:

```python
class FooOAuthSettingsHelper(OAuthSettingsHelper):
    ...
    def get_handlers(self):
        return dict(
            authorized_handler='invenio_oauthclient.handlers'
                               ':authorized_signup_handler',
            disconnect_handler='invenio_oauthclient.contrib.orcid'
                               ':disconnect_handler',
            signup_handler=dict(
                info='mysite.foo_oauth:account_info',
                setup='mysite.foo_oauth:account_setup',
                view='invenio_oauthclient.handlers:signup_handler',
            )
        )
```

3. Implement the custom handlers, similarly to any of the existing ones.

#### Post login actions

For existing plugins or custom plugins, handlers functions can be re-implemented to fulfill different requirements.
For example, the handler could read extra OAuth attributes and create the account in Invenio in a different way or it could allow/deny access based on the email.

```python
from invenio_oauthclient.errors import OAuthClientUnAuthorized

def foo_get_account_info_handler(remote, resp):
    if not resp["email"].endswith("@foo.com"):
        raise OAuthClientUnAuthorized(f"Email {resp["email"]} is not allowed.")
    else:
        return {
            ...
        }

remote_app = foo.REMOTE_APP
remote_app["signup_handler"]['info'] = foo_get_account_info_handler

OAUTHCLIENT_REMOTE_APPS = dict(
   foo=remote_app,
)

FOO_APP_CREDENTIALS = dict(
   consumer_key="<CLIENT ID>",
   consumer_secret="<CLIENT SECRET>",
)
```

#### Account setup after first login

With external authentications, on first login, a local account will be created, linked to the remote account.
If the external authentication plugin provides enough information to Invenio (at least the email, which is required), the account is automatically created without user interactions.
If instead extra information is required, the user might be prompted with a registration form after the first successful login. For example, ORCID does not provide the email of the user and it has to be input by the user after login.

Additionally, some organizations or institutions might need the user to fill in extra fields: an example is to require the user to agree on certain terms of use before accessing the site. The user registration form should be customizable per external authentication.
We propose to introduce a new configurable function `OAUTHCLIENT_SIGNUP_FORM` that allows to define a custom registration form:

```python
from invenio_oauthclient.utils import create_registrationform
from wtforms import HiddenField, validators
from werkzeug.local import LocalProxy
from flask import current_app

_security = LocalProxy(lambda: current_app.extensions['security'])

def my_registrationform(*args, **kwargs):
    """My registration form."""
    oauth_remote_app = kwargs["oauth_remote_app"]
    if oauth_remote_app.name == "mykeycloak":
        class RegistrationForm(_security.confirm_register_form):
            email = HiddenField()
            terms_of_use = BooleanField("Accept the <a href=''>terms of use</a>", [validators.required()])
            password = None
            recaptcha = None
            submit = None  # defined in the template
        return RegistrationForm(**kwargs)
    else:
        return create_registrationform(*args, **kwargs)

OAUTHCLIENT_SIGNUP_FORM = my_registrationform
```

The function will have a `kwarg` argument that contains the remote app object which can be optionally used to define different forms per external authentication app. The Jinja template that renders the form can be already overridden for even more advanced look and feel customizations.

Extra fields, as in the example above `terms_of_use`, are at the moment not stored in the database. This is because it is not possible to store extra fields per user profile: the user profile database table does not allow extra data (see the [Drawbacks](#drawbacks) section).

#### Disable profile edition

When only external authentication is configured, system administrators might want to disable the possibility to edit the user profile and allow changing, for example, the name. This is because such information if most of the time fetched from the external authentication provider.
Since this is a system administration choice, depending on the external authentication provider, we propose to introduce a new config variable `USERPROFILES_READ_ONLY = True` to the `invenio-userprofiles` module: this setting with make the form to edit the user profile read only and will also disable the view function that handles the form submission.

### Groups, membership, roles

In very simple words, user authorization in Invenio works by comparing what a protected resource needs and what the current user provides. For example, a protected endpoint might need a certain user role to allow access, e.g. `role:administrator`, but the current logged in user does not have this role. The access is denied. On the contrary, if the logged in user is an admin, the role `role:administrator` is added to his session on login: the user can provide the required role by the endpoint and the access is granted (please refer to the related Invenio documentation for more details on the subject).

In an organization or institute context where authentication is delegated to the organization authentication provider and not locally to Invenio, authorization might also be managed externally to Invenio.
A basic scenario could be that, after successful authentication, the authentication response payload contains a set of user roles. Or such user roles might be fetched later on, after authentication, when fetching the user profile.
Another more complex scenario could be that each user in the organization belongs to a certain groups, and such groups are used to authorize access to resources (e.g. LDAP groups/attributes).

It is quite hard to design a solution that can fit all the possible requirements, scenario and organizations set ups. Because of this, in this RFC we propose interfaces and data structures that will be required by Invenio to enable user authorization in such contexts. Fetching roles/membership after user login is responsibility of the instance's administrators, who will have to consult their own organization documentation to implement it in the best possible way.

Below some possible scenario. The fetching of user groups will happen after having fetched user information, which happens in the `signup_handler.info` handler of each OAuth/Keycloak plugins in `invenio-oauthclient`.

```python
def info_handler(remote, resp):
    ...
    # existing code
    user_info = get_user_info(remote, resp)
    ...
    # new implementation: fetch groups synchronously
    roles_or_groups_names = fetch_roles_or_groups_names(remote, user_info)
    provides = set(UserNeed(current_user.email))
    # add groups as Invenio roles to user session
    for name in roles_or_groups_names:
        provides.add(RoleNeed(name))
    identity.provides |= provides
    session["<my_external_app_name>_roles"] = provides
    # end new implementation
    ...
```

The `fetch_roles_or_groups_names` might retrieve the user groups from the `user_info` attributes previously fetched
or perform a new network request to fetch them from other REST APIs.

When groups cannot be retrieved synchronously in the same HTTP request (slow or heavy task), a possible solution could
be:
1. Fetch user groups async in a celery task and store it in the database. The `RemoteAccount` database table
   contains a `extra_data` column which could be used to "cache" user groups.
2. On login, enrich the user session identity `provides` by reading the list of groups from the database.
3. Refresh the list of user groups of each user when it makes sense in the organization context.

#### Make organization groups searchable

When restricting access to records, sharing records or defining authorization (e.g. community curators) is often required
to select not only single users, but also groups.
To make groups searchable from the UI, we propose to cache all organization's relevant groups in the Invenio database,
in the `roles` table. It could be achieved with a celery task that will, from time to time, sync all organization groups
with the database tables.

In this way, the Invenio UI can query the `roles` table to propose the user setting restrictions to choose one of the
existing group.

The solution of performing search queries to an external system on demand (from the UI) has been evaluated but
at the moment discarded due to the potential complexity of connecting to external services in terms of performance
and access authorization.

## How we teach this

Concepts to clarify when teaching this:

- The difference between local authentication and external authentication
- The need of Invenio to create a local account, connected to the remote account, even when using external authentication only
- OAuth plugins and how to create a new plugin
- Why groups/memberships support? Mention resources (records, views, files, etc...) restriction, sharing, user roles

## Unresolved questions

At the time of writing, the user profile in Invenio cannot be extended to store extra fields. Therefore,
it is not possible at the moment to store any custom registration form field to keep track of user selections.
A dedicated RFC will be created to cover the implementation of user profile extension.
