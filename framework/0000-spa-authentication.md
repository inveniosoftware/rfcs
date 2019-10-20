- Start Date: 2019-10-20
- RFC PR: [#10](https://github.com/inveniosoftware/rfcs/pull/10)
- Authors: Nicola Tarocco
- State: IMPLEMENTED

# Enable Single Page app authentication in Invenio

## Summary

Provide a secure authentication flow for Single Page Applications with CSRF support and ensure protection to
Cross-Site Scripting (XSS) and Cross-Site Request Forgery (CSRF) attacks.

[Here](https://jcbaey.com/authentication-in-spa-reactjs-and-vuejs-the-right-way) a reference article.

## Motivation

Note: with the term `SPA` we mean Single Page Application.

- As a SPA developer, I want to authenticate on Invenio backends from a client-side app, so that I can perform
  authenticated requests.
- As a SPA developer, I want to authenticate on Invenio backend using Invenio local accounts or remote OAuth
  providers.
- As a SPA developer, I want to fetch user information for the current authenticated user, so that I can use
  them to provide a better user experience.
- As a developer, I want to be sure that requests to backend are secure, so that the JavaScript client side application
  is not exploitable by malicious attackers.

## Detailed design

### Current state

The current authentication workflow in Invenio is implemented in `invenio-accounts` module.
The module relies on `Flask-Security` for:
- login/logout/register/change psw/psw reset/email confirmation view
- session based authentication
- login tracking
- basic user roles

`Flask-Security` relies on `Flask-Login` for:
- login and logout workflows
- methods decorators to protect views (e.g. `login_required`)
- user mixins
- storing user session id

Currently, an user login flow with username and password (not token based) uses `Flask-Security` views to
create/validate HTML forms and user credentials. Then, it relies on `Flask-Login` to create the user session and the
cookie.

Note: even if `Flask-Security` supports login via JSON request, it cannot be easily used because on login success, it
returns as response the generated token which unfortunately cannot be stored in a secure way in a JavaScript application.

### New authentication flow for SPA

#### Simple login with local accounts

We propose to create a new endpoint `POST /api/login` to allow client-side applications to submit
an authentication request to the backend with user credentials.
On new request, the backend should validate the credentials, as done with the classic form submission and validation in
[Flask-Security login()](https://github.com/mattupstate/Flask-Security/blob/develop/flask_security/views.py#L67).

After validation, the backend should login the user as done in `Flask-Login` but returning a different response. It should:
- Return a JSON payload with the result of the login (success or fail) and user id, to allow the client-side app to fetch the
  user from an user endpoint.
- On success, create a pair of cookies.
    - The first should be readable by JavaScript (`secure`, but not `http_only`) and should contain at least a
      CSRF token to be added to any future POST/PUT/DELETE requests. It can optionally contain
      more data useful to the client-side app, e.g. username, email and roles. This cookie could contain, for example,
      a JWT payload with structured info.
    - The second cookie should be the one already created by Invenio with the user session. Such cookie must not be readable
      by the JavaScript application (`secure` and `http_only`).

Client-side application will be able to access the cookie with the user information and CSRF token, but will not be
able to access the session cookie.

Any other view provided by `Flask-Security` that authenticates user and performs a `render_template` or `redirection`
should be implemented in the same way (`logout`, `register`, `psw reset`, `change psw`, etc...).

The two cookie authentication approach is well described [here](https://medium.com/lightrail/getting-token-authentication-right-in-a-stateless-single-page-application-57d0c6474e3).

We propose to also create a new logout endpoint `/api/logout` which will logout the user and change the cookie payload
to return an anonymous user and allow the client app to change state.

#### Login via OAuth

Invenio implements OAuth login with the module `invenio-oauthclient`. We propose to create an optional endpoint,
for example `GET /api/oauth-providers/`, to expose all configured OAuth providers and allow client-side apps
to render a login view and display all available authentication methods.

To perform external authentication, we propose to create a new endpoint `POST /api/login/<remote_app>` to handle
the OAuth flow. On a new request, the backend should return a redirection to the external OAuth provider and perform
the first part of the OAuth flow. The callback parameter should have as value the backend endpoint to handle
the first step of the OAuth login `/api/login/<remote_app>/authorized/`.
On callback called, `invenio-oauthclient` should complete the OAuth flow as already implemented. A successful login
flow should behave as described above with the creation of the double cookie. It should then redirect back to the
client-side app with a success/failure status.

### CSRF

Any HTTP request that will perform a state change on the backend (normally `POST`, `PUT`, `DELETE`) and relies on
session cookies for authentication should be protected from CSRF attacks.

The backend should require and validate the CSRF token for such requests. The CSRF might be submitted as part of the
request payload or as HTTP header.

We propose to implement a `before_request` hook that will require and validate the CSRF token for the requests above.
The CSRF token sent by the client will be validated by comparing it with the version issued by the backend. The token
can be stored it backend or in trusted cookie. In any case, the CSRF cookie should enforce the `SameSite` cookie policy
(see below) so that the CSRF token cookie will be only sent for same-site requests.

#### CSRF expiration

For normal server side apps, the expiration of the CSRF token is normally quite short because it is injected with the
form at rendering time, with the assumption that the form will be submitted shortly.
For client-side apps, the expiration should be set longer, e.g. 30 minutes, because the backend cannot guess when a
request will be submitted.

To achieve so, the CSRF validity should be set with longer expiration and a new token should be generated on each
POST/PUT request to ensure that a fresh one is always available to the client-side app.
In case the CSRF token validation fails, the backend should provide a new token in the response.

### SameSite cookie for extra protection

[SameSite cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) is a feature that allow servers to
instrument browser to send cookies only to requests to the same site, and not to send them cross-site.
This will enable protection against CSRF.
The `SameSite` feature is relatively new and it is supported by recent browsers.
See [here](https://caniuse.com/same-site-cookie-attribute) the browser support for `SameSite` cookie attribute.

In preparation for the future, this new feature could be already integrated in Invenio.
In `invenio-accounts`, cookies and sessions are created by [flask-kvsession](https://github.com/mbr/flask-kvsession/)
to allow storing sessions on different types of store (Redis, DB, memory, etc...).
To enable `SameSite` cookie, an extra option has to be passed when creating the cookie,
[here](https://github.com/mbr/flask-kvsession/blob/master/flask_kvsession/__init__.py#L197). Unfortunately, this library
seems not maintained anymore. We propose to fork it and becomes maintainer for such fork.
See [here](https://github.com/inveniosoftware/flask-kvsession) the Invenio fork.

## How we teach this

This new flow should be extensively described in the `invenio-accounts`, `invenio-oauthclient` and Invenio
documentation. This is an advanced topic and it should be probably not part of the Invenio Bootcamp/Tutorial for
beginners.

The documentation should add a new section and highlight the following:
- description of the flow, mentioning the double cookie technique
- the need of the CSRF token and security implications
- the SameSite support

The referenced blog posts in this RFC are great resources to deeply understand the complexity, the challenges
and the potential security issues involved in client-side app authentication flows.

## Drawbacks

The authentication flow, implementation wise, becomes more complicated.

## Alternatives

No alternatives at this moment.

## Unresolved questions

- Should `Flask-Security` be replaced? Unfortunately, it seems that it is not actively maintained.
