- Start Date: 2020-03-23
- RFC PR: [#21](https://github.com/inveniosoftware/rfcs/pull/21)
- Authors: Alex Ioannidis
- State: Draft

# Example apps removal

## Summary

Example applications in Invenio modules will be gradually removed.

## Motivation

Example applications were meant to be a quick and intuitive way to showcase an
invenio module and its basic features. Over time several issues have emerged:

- New features are added to the module and the example app is not updated.
- The quality of example apps is varying from module to module. Sometimes they
  really serve as creative and useful examples on how the module's features can
  be used, and other times they were done as a "chore".
- The black-box-like testing approach for example apps, besides being a bit
  "unorthodox" and difficult for newcomers to understand, is also slow to run
  since it bootstraps an entire Invenio/Flask application with assets building,
  DB/ES initialization, etc.

## Detailed design

When we come across an example app that causes problems (e.g. tests or coverage
failure) we remove from the repository:

- the entire `examples` directory
- the `tests/test_example_app.py` file
- the `docs/exampleapp.rst`, and its `toctree` entry from `docs/index.rst`

We're not approaching this proactively, i.e. there's is no need to remove the
example apps from all the Invenio repositories all at once.

## Example

To remove any traces of an example app from a repository the following shell
commands can be run:

```bash
rm -rf examples
rm tests/test_example_app.py
rm docs/exampleapp.rst
sed -i "/exampleapp/d" docs/index.rst
```

## How we teach this

We will reference this RFC in discussion for developers. Also, the module
cookiecutter template shouldn't generate an example app anymore.

## Drawbacks

By removing example apps we might be also removing some actually useful
code examples that were written at some point. Ideally these should be either
moved of the the module's "Usage" documentation, or in a separate section.
