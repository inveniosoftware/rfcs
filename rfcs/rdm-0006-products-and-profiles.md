- Start Date: 2019-10-02
- RFC PR: [#6](https://github.com/inveniosoftware/rfcs/pull/6)
- Authors: Guillaume Viger

# Customize created Invenio instance by choosing profile and optionally product

## Summary

When running the instance bootstrapping CLI command (`invenio-cli init`), the
hoster should be given a choice to create one of Invenio's **product** e.g., an
Integrated Library System (ILS) or a Resource Data Management (RDM) system. If RDM
is chosen, there are further **repository profiles** to be chosen from, for
instance "default" and "biomed". These choices pre-configure the application to varying levels.


## Motivation

As a (prospecting-)hoster, I want to have a quick working application for my
main use case: library system or data management, so I can evaluate it
right away (prospecting) and/or start using it (configuration/development). By
choosing the RDM choice I want to further leverage pre-configuration /
pre-customizations for my select domain in order to get started faster and adopt
same settings as other installations in my domain (e.g. choose the "biomed"
repository profile to take advantage of commonality in biomed repositories).

As an InvenioRDM (core-, extension-, plugin-)developer for domain stakeholder,
I want to satisfy the exclusive needs/wants of my stakeholders
while still being able to develop and use domain authorities **within**
the InvenioRDM context, so that I can customize my repository for my situation.


## Terminology

**Invenio product**: A *product* represents the main use case of the application
(e.g., ILS or RDM for now) and it translates to a different packaged application.
Products have completely different application structures. This calls
for different project repositories to define them.

**repository profile**: A *repository profile* represents a different domain of focus/interest
(e.g., generic or biomedical) for the InvenioRDM *product*. It translates to
different configurations, data models, and customizations. Profiles share the
same application structure but differ in the content.

**generic** or **default** profile: A profile without specific domain of
interest customizations. This profile defines generic resource types and
doesn't load domain subjects. It's a good choice for libraries and general data
index/repositories. The resulting installation is flexible at the cost of no
built-in customizations for a specific context. Despite making no assumption
about its context, it can be further customized.

**biomed** profile: A profile with configurations and functionalities relevant
to bio-medical institutions (MeSH subject terms, medical resource types...).
Despite making more assumptions about its context, it can be further customized.


## Detailed design

```console
$ invenio-cli init --product=<RDM|ILS|framework>
  [... other questions ...]
  Select repository profile:
  1 - generic
  2 - biomed
  Choose from 1, 2 (1, 2) [1]:
```

(or something close to this syntax) generates the pre-configured RDM application
with the specific resource types, data model, UI and other configuration relevant
to Bio-Medical institutions. Passing `--profile generic` generates one with
default configurations.

The [invenio-app-rdm](https://github.com/inveniosoftware/invenio-app-rdm)
repository contains the RDM product implementation.
The [invenio-cli](https://github.com/inveniosoftware/invenio-cli)
repository contains the CLI tool to generate an instance with chosen profile/product.

The [invenio-app-ils](https://github.com/inveniosoftware/invenio-app-ils) contains
the ILS application product.


## How we teach this

This might be challenging as the terms `profile` and `product` have close meaning
in every day language. Consistently adhering to the correct word for the correct
technical meaning will be important to anchor that relationship, especially in
person-to-person, chat and documentation settings.

The terminology should be brought up in the [invenio-cli](https://github.com/inveniosoftware/invenio-scripts) app docs.
The usage should be demonstrated in the general RDM documentation. `product`
should also be described in [invenio-app-rdm](https://github.com/inveniosoftware/invenio-app-rdm).


## Drawbacks

As this is the start of the project, what qualifies as profile-worthy
and what qualifies as product-worthy differences will need to be felt out from
experience. Luckily, at first, only the RDM profile will be supported.

The implied differences between products may be overstated and provide decision
uncertainty as more products are added. Lindsay works in a biology research
institution. She may be unsure about using a "BioMed" profile versus a generic
one. Should she create her own profile? It should be communicated that
further customization is possible to match her setting (has been addressed above);
the decision is not irrevocable. Manual reconfiguration can be done. It should
also be made clear, she doesn't *have* to create a new profile. This will be
communicated in the documentation.


## Alternatives

No Invenio application hierarchy: all configuration is at the same level.
This was not chosen because of the extent of differences between ILS and RDM.
As for products, to be frank, stakeholders involved wanted products to
address their needs and give themselves leeway in implementation details.

## Unresolved questions

To discuss further / clarify upon implementation: where are products defined?
