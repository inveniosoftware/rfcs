- Start Date: 2020-08-28
- RFC PR: [#38](https://github.com/inveniosoftware/rfcs/pull/38)
- Authors: Diego Rodriguez Rodriguez, Pablo Panero

# PyPi releases using GitHub actions

## Summary

Python packages will be released to PyPi using GitHub actions, while the tests
will run on Travis.

## Motivation

In order to ease the load on travis and unblock the release process, packages
will be released using GitHub Actions.

- As a developer, I want to release packages and not be blocked waiting for
travis, so that I can continue developing/releasing with a fast pace.

In addition, this changes take advantage of the *organizational* secrets on
GitHub. That way, PyPi credentials are stored in a single point, and only one
token. This eases the load on token management (one token vs. one per
repository).

## Detailed design

In order to implement this change we need to:
- Disable the tag builds on travis.
- Add a github workflow (action) that is executed only for tags. This action will release to PyPi, authenticating with an organization level token.

How to implement this changes can be seen in the example below.

## Example (How to migrate)

### Set the organization token (one-off action)

1. In [PyPi](https://pypi.org/manage/account/) create an API Token. Set the
    scope to *Entire account*. **Important: do not close that window, you
    won't see the token againg**.

    When creating the token you can restrict any organization secret, to a
    subset of selected (PyPi) projects. Meaning, that not all will be able
    to access it.

3. Add the organizational secret in GitHub:
    <https://github.com/organizations/inveniosoftware/settings/secrets>.

### Disable travis for tags

Assuming that the versions (tags) follow the `vX.Y.Z` semantics, add the
following line at the beginning of the `travis.yml` file:

```yaml
# blocklist
branches:
  except:
  - /^v\d+\.\d+(\.\d+)?(\S*)?$/
```

It would match, for example:

- `v1.0`
- `v1.0.3`
- `v1.0.3a3`

In addition, remove the `deploy` section and any legacy remaining variables
(e.g. `DEPLOY=True` in the matrix).

```diff
- deploy:
-   provider: pypi
-   user: inveniosoftware
-   password:
-     secure: PYPI_SECRET
-   distributions: "sdist bdist_wheel"
-   on:
-     tags: true
-     repo: inveniosoftware/docker-services-cli
-   skip_existing: true
```

### Add the GH workflow (Action)

**Note**: If you want to test before production use
<https://test.pypi.org> instead of <https://pypi.org>.


Add a `.github/workflows/pypi-publish.yml` file with the following content:

**Important notes**:
- The code below code assumes that the secret name is `pypi_token`. You set
    the name given in step 2.
- If you have translations, you should add `compile_catalog` to the build
    package command.

```yaml
on:
    push:
    tags:
        - v*

jobs:
    build-n-publish:
    runs-on: ubuntu-latest
    steps:
        - uses: actions/checkout@v2
        - name: Set up Python 3.7
        uses: actions/setup-python@v2
        with:
            python-version: 3.7
        - name: Install dependencies
        run: |
            python -m pip install --upgrade pip
            pip install setuptools wheel
        - name: Build package
        run: |
            python setup.py sdist bdist_wheel
        - name: pypi-publish
        uses: pypa/gh-action-pypi-publish@v1.3.1
        with:
            user: __token__
            password: ${{ secrets.pypi_token }}
```

### Manifest

If the tests fail in `lowest` you might need to add the following line to the
`MANIFEST.in` file:

```
recursive-include .github/workflows *.yml
```

## How we teach this

We will reference this RFC in discussion for developers. Also, the module
cookiecutter template for modules will generate the corresponding files.

## Alternatives

There are other CD/CI alternatives like Circle CI and GitlabCI. It was decided
to use GitHub actions after the comparisson made by Diego
(<https://github.com/inveniosoftware/invenio/issues/4036#issuecomment-677447050>)

## Unresolved questions

- Will the migration be done in batch with the new
[automation tools](https://github.com/inveniosoftware/automation-tools) or
will it be done progresively on a per repository basis.

## Resources/Timeline

This RFC states the migration process. However, it's migration depends on the
resolved quesion above.
