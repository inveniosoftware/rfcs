- Start Date: 2020-04-08
- RFC PR: [#33](https://github.com/inveniosoftware/rfcs/pull/33)
- Authors: Alex Ioannidis
- State: DRAFT

# Resource access tokens (RATs)

## Review/Discussion

The RFC was reviewed. An initial implementation lives in Zenodo. The RFC is
planned to be updated once work restarts in InvenioRDM/Zenodo to implement
"share by link" feature and migrate the Zenodo-AccessRequest module.

**Comments**

- Where would the code live? Maybe OAuth2Serer (possible candidate), maybe
  Accounts, maybe new module.
- Who should generate the token? Probably both a user and instance should be
  possible. Key issue is that a user can lie about the expiration date. We need
  documentation on how a user should generated the toke, and a REST API for
  an instance to allow users generating a token.  Also, secret keys more than
  the personal access tokens needs to be introduced to allow for controlling
  if a user can or cannot generate a token).
- What is in the payload, and how does it translate to permissions? Feeling is
  it needs to generate needs, but in a sane way that doesn't expose internals.
- How do we make it generic enough to work for Google Docs/Dropbox like share
  a link where the expiry is much longer. Which parts can be shared.

## Summary

Allow users to generate via their OAuth2 Personal Access Tokens short-lived
signed JWTs that enable granular access to a specific resource. This is useful for:

- Ad-hoc generation of "secret" links to private resources, like e.g. closed-access files
- Integration with 3rd-party systems that use your platform to store records, and don't
  want to proxy all the protected resources via their system

## Motivation

In order to currently share any protected resource (e.g. deposits, deposit
files, closed access files) there are two options:

- For deposits and their files, or published records that are
  `Closed`/`Embargoed` you can use personal access tokens.
  - Although this allows flexibility in terms of sharing all kinds of resources
    it still lacks in terms of granularity; even though our OAuth2Server
    personal access tokens contain the concept of `scopes`, these cannot be
    narrowed down to individual resources (and their sub-resources) like e.g.
    specific deposits, records, or files. Practically this means that
    sharing one's private access token allows for impersonating this user
    across all their resources (e.g. all closed access records).
- Custom workflows, like e.g. an [access request mechanism](https://github.com/zenodo/zenodo-accessrequests/)
  - This has its own limitations based on the design process (e.g. validating
    a user's email before they can request access), and is not a viable
    solution for sharing things "on-demand".

On top of that, some popular feature requests are:

- Generate "secret" links for closed access files and share them with others
- Give read-only access to a deposit's files before publising (e.g. for peer
  review)

## Detailed design

The basic concept is that the user generates a **Personal Access Token
(`PAT`)** with the `tokens:generate` scope. This generated PAT, also has a
**unique ID (`PAT-ID`)** which the user knows. The user can then use `PAT-ID`/`PAT`
pair to generate signed JWT tokens on their side (i.e. without using any of the
service's APIs), called **Resource Access Tokens (`RAT`)** that allow granular
access to specific resources they manage in the system.

The generated JWTs contain the following data:

```python
{
    "header": {
        # This is needed so that the server can know which PAT to use to verify
        # the JWT's signature.
        "kid": "<PAT-ID>"
        "alg": "HS256",
        "typ": "JWT",
    },
    "payload": {
        # The "Issued at" key is required for expiration checks
        "iat": 1516239022,  # (timestamp of issue time)
        # The data format of the "Subject" key is up to each implementation to
        # allow flexibly describing resources (see example below)
        "sub": ...,
    },
    "signature": HMACSHA256(header + payload, secret="<PAT>")
}
```

The system is responsible for implementing the authorization logic the specific
resources require and the format in which they are described inside the `sub`
key of the JWT. For example, if you want to allow RATs to be generated to
access specific files of a deposit for read/write access, you might use a `sub`
such as:

```python
{
  ...
  "sub": {
    "deposit_id": 5678,  # the deposition ID
    "file": "data.zip",  # the file
    "access": "read",    # level of access allowed
  }
}
```

...and then e.g. allow the generated RAT to be passed via a `token` querystring
parameter when requesting the resource:

```http
GET /api/deposit/depositions/5678/files/data.zip?token=<generated-RAT>
```

There are a couple of important details on the RAT's design:

- They are always connected to a "signer", i.e. there's always a user
  associated with a RAT. This is a required design feature in order to allow
  the backend to perform the necessary permission checks regarding the
  owner/manager of a resource. It is also useful though for auditing purposes.
- For security reasons they are meant to be short-lived. The maximum lifetime
  of the token is defined by the consuming system and not the issuer/signer of
  the RAT. This is importnat since the only way to "revoke" a RAT is by
  deleting the PAT that was used to sign it.

## Example

Here is a proof of concept implementation of RATs for generating them on the
client-side, and validating them on the server-side.

### Generating the RAT

```python
# User generates his personal access token (PAT) and its ID (PAT-ID) via the UI
PAT_TOKEN = 'abcdef...'
PAT_TOKEN_ID = 1234

# Example generation of a RAT in Python, using PyJWT
import jwt
from datetime import datetime

RAT_TOKEN = jwt.encode(
    payload={
        'sub': {
          'deposit_id': 5678,
          'file': 'data.zip',
          'access': 'read',
        },
        'iat': datetime.utcnow(),
    },
    key=PAT_TOKEN,
    algorithm='HS256',
    # We store the personal token ID in "kid" (see JWT spec:
    # https://tools.ietf.org/html/rfc7515#section-4.1.4)
    headers={'kid': PAT_TOKEN_ID},
)

# Make an HTTP to fetch a file from the specific deposit
import requests
res = requests.get(
    '/api/deposit/depositions/5678/files/data.zip
    params={'token': RAT_TOKEN},  # could also be in a special header...
    stream=True,
)

#
# Possible response scenarios
#
# Token expired (i.e. "iat" + 1day > utcnow())
assert res.status_code == 401
assert res.json() == {"message": "File access token is expired"}

# Token invalid (i.e. bad signature, missing "kid" or "iat" fields)
assert res.status_code == 400
assert res.json() == {"message": "Invalid token"}

# Payload conflict, e.g. deposit ID mismatch, token ID doesn't exist
# Note: maybe it's better for security purposes to mask these errors under
#       "Invalid token" to avoid exposing too many implementation details.
assert res.status_code == 401
assert res.json() == {"message": "You cannot access this deposit"}

# Token is valid - file is returned in payload
assert res.status_code == 200
open('data.zip', 'wb').write(res.raw)
```

### Validation on the server

```python
import jwt
from datetime import datetime, timedelta
from invenio_oauth2server.models import Token

TOKEN_LIFETIME = timedelta(minutes=30)

def decode_rat(token):
    # Retrieve token ID from "kid"
    access_token_id = jwt.get_unverified_header(token)['kid']
    access_token = Token.query.get(access_token_id)
    signer = access_token.user

    # Check scopes of the PAT used to generate the token
    assert 'tokens:generate' not in access_token.scopes

    try:
        payload = jwt.decode(
            token,
            key=access_token,
            algorithms=['HS256'],
            options={'require_iat': True},
        )
        # Verify that the token is not expired based on its issue time
        if payload['iat'] + TOKEN_LIFETIME < datetime.utcnow():
            raise jwt.ExpiredSignatureError('Signature has expired')
    except jwt.ExpiredSignatureError:
        # TODO: handle
    except jwt.InvalidTokenError:
        # TODO: handle

    return (
        signer,
        payload['sub'],
    )


@route('/api/deposit/<deposit_id>/files/<filename>')
def deposit_files_view(deposit_id, filename):
    deposit = resolve_deposit(deposit_id)

    # Check for RAT
    rat_token = request.args['token']
    signer, payload = decode_rat(rat_token)

    # Check that the underlying resource is actually managed by the RAT signer
    if not signer.id in deposit['owners']:
        raise PermissionError(...)

    # Business logic checks for the RAT
    valid_claim = (
        payload['deposit_id'] != deposit_id or
        payload['file'] != filename or
        (request.method == 'PUT' and payload['access'] in ('admin', 'write'))
    )
    if not valid_claim:
        raise PermissionError(...)

    # ...continue with processing the request
```

## Drawbacks

- Each RAT has a **signer**, and it is very important, but easy to forget, when
  doing the permission check business logic to also check that the underlying
  resource is actually managed by the signer. In simple terms, Bob can only
  generate RATs for resources (e.g. files) that he actually owns/manages, so
  when verifying Bob's RATs you should not forget to check that the resource
  that is being accessed actually belongs to Bob.

  Maybe this sounds obvious, but failing to do this simple check, is basically
  an easy way to "bazooka" yourself in the foot and allow everyone global
  access to all of your resources, since technically anyone can generate RATs
  with any payload they want.

  Although this "hidden user" aspect of the RAT is a bit dangerous when
  implementing business logic on top of it, there are ways to mitigate it, for
  example:
  - By providing high-level programmatic APIs that always require a user to
    validate the resource against the signer.
  - Via translating the RAT to a normalized set of `Need`s, and making sure to
    know how to configure the set of `Need`s a resource requires.
- This entire design is very much similar to how public-key cryptography works,
  although the values used for the JWT signing are much smaller (a PAT is
  usually ~60 bytes). This might be considered a security issue, since weak
  cryptography is no cryptography at all, but the fact that the generated JWTs
  are short-lived, which is also a parameter defined by the system (and not the
  signer), and rotating the token is easy.

## Alternatives

- Not any obvious ones that wouldn't require major rewrites and development of
  existing well-established modules like e.g. Invenio-OAuth2Server.

## Unresolved questions

- [ ] [Read more](https://connect2id.com/products/nimbus-jose-jwt/algorithm-selection-guide#signatures) on what kind of JWT algorithms are appropriate for our use case
- [ ] Display token ID in the personal token generation UI settings page
- [ ] Figure out what the expiration window should be (minutes vs. day)
  - [ ] Discuss with Dryad what's the use case in their UI workflow
- [ ] Figure out appropriate HTTP response codes (check maybe equivalent OAuth2 spec?)
- [ ] Allow passing the token via a header? We have to avoid clashing with the
  `Authorization` header of `OAuth2Server` though, so we need a better name...

## Resources/Timeline

- Proof of concept already implemented in the Zenodo code-base, see [PR
  #1971](https://github.com/zenodo/zenodo/pull/1971).

