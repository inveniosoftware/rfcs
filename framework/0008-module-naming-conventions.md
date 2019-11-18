- Start Date: 2019-10-03
- RFC PR: [8](https://github.com/inveniosoftware/rfcs/pull/8)
- Authors: Guillaume Viger

# Module naming conventions

> There are only two hard things in Computer Science: cache invalidation and naming things.
> - Phil Karlton

## Summary

Here are *recommendations* for naming modules in Invenio.

## Motivation

As a developer, I want names to be consistent and have help in naming
things, so that I can name, parse and use modules and configuration variables
without namespace clashes (especially on public package indices), by relying on
these conventions to make automation tools and to not have to go through
bike-shedding whenever I name things.

We think [convention over configuration](https://rubyonrails.org/doctrine/#convention-over-configuration)
in naming is a good approach to cut down on mundane recurring decisions. This
RFC will suffer so you don't have to.

## Detailed design

Any installable Invenio module, should be named `invenio-<module>`. This naming
pattern can be snake_cased directly when used in Python code: `invenio_<module>`.

Module names should be plural when about countable things e.g.,
`invenio-records`, `invenio-stats`, `invenio-communities`.

The `<module>` part of the module name should be used to prefix the configuration
variables of the modules. Those are always in CAPITAL_SNAKE_CASE. For example,
for the `invenio-bar-foos` module: `BAR_FOOS_<CONFIGURATION_VARIABLE>` is used
in `config.py`

Increasingly specific sub-projects should prefix the `<module>` part:
`invenio-rdm-<module>`, `invenio-records-<module>` (e.g., `invenio-records-rest`,
`invenio-records-permissions`)

Non-Invenio projects part of another framework should follow that framework's
conventions. (e.g., `flask-menu` for a flask module).

Other than that (general libraries beyond Invenio, e.g. `datacite`), follow the
conventions of the language you are using.

## How we teach this

New modules will always have to go through an RFC, and thus naming conventions
will be part of the RFC process.

## Drawbacks

A couple of existing Invenio modules are not currently conforming with these practices, e.g.
`invenio-deposit -> invenio-deposits`, `invenio-previewer -> invenio-previewers`, etc.
Since the work that has to be done to rename these modules though outweighs the
naming convention benefits, it's better to leave them as is.

Over-specifying naming conventions is restrictive, under-specifying leads
to ambiguities and muddled names where it is hard to understand what the project
is about.

## Unresolved questions

Documentation module: should they be `docs-<module>` or `<module>-docs`? I
would lean for the former.
