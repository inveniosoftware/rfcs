- Start Date: 2020-03-12
- RFC PR: [16](https://github.com/inveniosoftware/rfcs/pull/16)
- Authors: Lars Holm Nielsen

# End of Python 2 support

## Summary

The end of Python 2 support will be handled by gradually removing Python 2 from
the test matrix.

## Motivation

Python 2 reach end of life on January 1st, 2020. Invenio consequently no longer
supports Python 2. This can mean many things to developers so the purpose of
this RFC is to define what "no longer supports Python 2" means in practical
terms.

## Detailed design

TLDR: We stop testing Python 2, but are not proactively going to remove Python
2 constructs from the code base.

### Current situation

Currently, Invenio's code base is Python 2 and 3 compatible, by using e.g.
the six library as well as ``__future__`` imports.

Cleaning the entire source code for Python 2 related code would be a very big
task with very little impact at this point. We are therefore taking a more
light and practical approach.

### No Python 2 tests

Python 2.7 is currently tested in the Travis CI test matrixes. You can either
simply remove Python 2 from the test matrix when a Python 2 test fails, or you
can actively remove Python 2 tests when you make a new PR to a repository.

This will not be done proactively, but reactively when needed.

### Python 2 constructs

Python 2 constructs may be removed from the source code.

Examples:

```
from __future import ...

if PY2:
    # ...
```

### New releases

New package releases should update the ``setup.py`` classifiers as well as
ensure that ``.travis.yml`` does not use the Python 2.7 build for deploying
a release to PyPI.

This will not be done proactively, but reactively when needed.

### Code coverage

In order to increase code coverage, you may remove code snippets only running
under Python 2. If there exists a ``_compat.py`` module for Python 2 support
it can be removed.

### New Python 3 syntax

We will stay with the current syntax code style which means that there's newer
Python 3 syntax that we will not start using (e.g. type hints).

## Example

### ``.travis.yml``

From:

```yaml
python:
   - "2.7"
   - "3.5"

deploy:
  provider: pypi
  # ...
  python: "2.7"
  condition: $DEPLOY = true
```

To:

```yaml
python:
   - "3.6"
   - "3.7"
   - "3.8"

deploy:
  provider: pypi
  # ...
  on:
    tags: true
  skip_existing: true
```

### ``.setup.py``

From:

```python
setup(
    # ...
    classifiers=[
        # ...
        'Programming Language :: Python :: 2',
        'Programming Language :: Python :: 2.7',
        'Programming Language :: Python :: 3',
        'Programming Language :: Python :: 3.5',
    ],
)
```

To:

```python
setup(
    # ...
    classifiers=[
        # ...
        'Programming Language :: Python :: 3',
        'Programming Language :: Python :: 3.6',
        'Programming Language :: Python :: 3.7',
        'Programming Language :: Python :: 3.8',
    ],
)
```

## How we teach this

We will reference this RFC in discussion for developers. Also, the cookiecutter
templates should all be updated to be Python 3 only.

## Drawbacks

This method will keep Python 2 code and libraries like six around for a
forseeable future. Six is however likely to say around from other third-party
libraries as well.
