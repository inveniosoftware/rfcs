# Invenio RFCs

The purpose of an Invenio RFC (Request For Comments) is to:

- coordinate the design process
- produce consensus among Invenio stakeholders
- document design decisions

Many changes can be implemented via the normal GitHub pull request workflow
(bug fixes and improvements), however some changes are "substantial" and we ask
that these are written up as an RFC.

RFCs are per product:

- [Invenio Framework](framework/)
- [Invenio RDM](rdm/)
- [Invenio ILS](ils/)

The RFCs are not meant to be a heavy long process, but rather as an aid to
communicate between geographically dispersed teams as well as to document
Invenio development so that we can avoid knowledge loss when people leaves and
ease knowledge transfer when people joins.

### When to write a RFC?

You need to write a RFC to make substantial changes to Invenio. A substantial
change could be:

- Adding/removing larger features and/or modules.
- Changing existing features/APIs.
- Changes of design patterns, idiomatic usage or conventions.

You do not need an RFC for:

- Modules/features you develop privately (you only need it, if you want it as
  an official Invenio module).

If in doubt, just ask on [Gitter](http://gitter.im/inveniosoftware/invenio).

### How to submit a RFC?

- Fork this repository.
- Copy ``0000-template.md`` to ``<product>/0000-<my-title>.md``.
- Fill in the RFC.
- Submit a pull request
- Update your pull request:
    - Rename your ``<product>/0000-<my-title>.md`` to ``<product>/<pull request id with leading zeros>-<my-title>.md`` (e.g. ``rdm/0012-persistent-identifiers.md``).
    - Add a link to the pull request inside the pull request.

### Process (WIP)

The decision process on RFCs have not yet been decided. Ideas is very welcome.

---

The Invenio RFC process owes it's inspiration to the
[Ember RFC process](https://github.com/emberjs/rfcs).
