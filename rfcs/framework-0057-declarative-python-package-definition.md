- Start Date: 2022-02-16
- RFC PR: [#57](https://github.com/inveniosoftware/rfcs/pull/57)
- Authors: Lars Holm Nielsen
- State: DRAFT

# Declarative Python package definition

## Summary

We're proposing to replace ``setup.py`` with a ``pyproject.toml`` and ``setup.cfg`` for Python packages to have purely declarative definitions of Python packages.

## Motivation

The overall motivation is to move to a purely declarative definition of Python packages instead of using the current programmatic method.

The pure declarative method allow us to easily parse a package's dependencies in a reliable fashion. This opens possibilities for better managing our dependencies in a controlled manner, as well as e.g. automated upgrade of specific package over many repositories.

In addition, this aligns our Python packages with the future of Python packaging and opens the possibility to use other build systems in the future if so desired.

## Detailed design

We propose to replace the current ``setup.py`` with a ``pyproject.toml`` and a ``setup.cfg``.

Python PEP 517 and PEP 518 (adopted 2015/2016) defines a new build-system independent format for source trees and how to specify minimum build system requirements in Python projects. Full support for PEP 517/518 is implemented in Setuptools 40.9+ (released April 2019) and PIP 19+ (released January 2019).

### Specifying build system

First of all our Python packages should add a ``pyproject.toml`` file that defines the build system:

```toml
# pyproject.toml
[build-system]
requires = ["setuptools", "wheel"]
build-backend = "setuptools.build_meta"
```

### Setuptools as build sytem

We could use another build system than Setuptools, however staying with Setuptools allows us to focus on migrating to a pure declarative format for our packages and maintain backward compatibility to a very large degree.

Having a minimal ``setup.py`` as below allow us have almost the entire package work as previously:

```python
# setup.py
from setuptools import setup
setup()
```

This way tools like Requirements Builder work without modifications and you can still install a package via ``python setup.py install`` and build docs and compile message catalogs via ``python setup.py build_sphinx/compile_catalog``.

### Declarative package definition

Below is a commented example how purely declarative ``setup.cfg`` would look like:

```
# setup.cfg
[metadata]
name = invenio-base

# The version number is read by parsing the AST.
version = attr: invenio_base.__version__
description = "Base package for building Invenio application factories."

# Files are concatenated as we do today.
long_description = file: README.rst, CHANGES.rst
keywords = invenio
license = MIT
author = CERN
author_email = info@inveniosoftware.org
platforms = any
url = https://github.com/inveniosoftware/invenio-base
classifiers =
    Development Status :: 5 - Production/Stable

[options]
include_package_data = True
packages = find:

# Replaces the PyPI classifiers.
python_requires = >=3.7
zip_safe = False
install_requires =
    blinker>=1.4
    Flask>=1.0.4,<3.0
    # importlib needed until Python 3.9 end of life.
    importlib-metadata>=4.4
    importlib-resources>=5.0
    Werkzeug>=1.0.1,<3.0

[options.extras_require]
tests =
    pytest-invenio>=1.4.2
    sphinx>=4.2.0

[options.entry_points]
console_scripts =
    inveniomanage = invenio_base.__main__:cli
flask.commands =
    instance = invenio_base.cli:instance

[build_sphinx]
# This option is being deprecated as of Sphinx 5.0 so we
# might consider removing it already.
source-dir = docs/
build-dir = docs/_build
all_files = 1

[bdist_wheel]
universal = 1

[pydocstyle]
add_ignore = D401
```

#### PyPI classifiers

We suggest removing all classifiers except the classifier related to development status.

- License: The license classifier is already specified in the ``license`` keyword.
- Python versions: The version classifiers were not kept updated, and where purely metadata, whereas the ``python_requires`` keyword is actionable and easier to keep updated.

#### Version number

The version number has to be moved from ``<package>/version.py`` to ``<package>/__init__.py``. This is because setuptools is only able to extract the version number using the Abstract Syntax Tree (i.e. not executing the code) when located in the ``__init__.py``. If the version number is placed in ``version.py`` setuptools will import the main package which may lead to imports failing.

#### Extras require

We can no longer programmatically define an ``all`` extra from other extras (like ``tests`` and ``docs``. There are two options:

- Duplicate all extra dependencies in an ``all`` extras (risk of getting outdated).
- Gather as much as possible into a ``tests`` extras (extra documentation needed).

We suggest as far as possible keeping a single ``all`` extra requires that includes tests and docs requirements. Extra requirements needs to be duplicated into this ``all`` requirements, but this means that all packages can mostly be installed via ``pip install -e ``.[all]`` to start development on the package.

### Building the package

The Python package can be built using (without first installing the package):

```
pip install build
python -m build
```

If we use above method, we however need to commit the compiled message catalogs to the source code repository.

We suggest for now, until Babel is better integrated with ``pyproject.toml`` and ``build`` that we stay with using the ``setup.py`` to compile the message catalog and bulid the source and wheel distributions:

```
python setup.py compile_catalog sdist bdist_wheel
```


### GitHub Actions

#### CI tests

Only changes related to the extra requirements is required if an ``all`` extra requirement is no longer provided.

#### PyPI publishing

No changes neeed.

## How we teach this

Above should be implemented in the ``cookiecutter-invenio-module`` template repository so that it can always be used as a reference implementation of a Python package for Invenio.


## Drawbacks

The benefits of this change only becomes available on the long-term as there are no immediate benefits. However long-term it enables us to improve and automate our dependency management which continues to be a major source of disruptions to development.

## Resources/Timeline

We suggest first adopting this PR. Once adopted, we can migrate repositories that we work with in the background, similar to how we remove Python 2 support. This avoids having to spend a large effort upfront.
