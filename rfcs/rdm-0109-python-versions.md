- Start Date: 2025-10-24
- RFC PR: [#109](https://github.com/inveniosoftware/rfcs/pull/109)
- Authors: Guillaume Viger

# Python support in Invenio/RDM ecosystem

## Summary

This RFC outlines the Python support policy for InvenioRDM and the related Invenio ecosystem. In particular, it outlines how the policy relates to concrete implementations/use-cases/actions. Some aspects are left flexible for agile decision making in future contexts.

The general idea is:

|                                 | Minimum Python version required               | Maximum Python version supported    |
| ------------------------------- | --------------------------------------------- | ----------------------------------- |
| Invenio framework level modules | Keep current minimum                          | Latest version officially supported |
| InvenioRDM specific modules     | Decided miminum for that InvenioRDM's version | Latest version officially supported |


## Motivation

As an Invenio(RDM) developer (contributor or maintainer), I want to know:

- what Python features I can use or not, so that I am effective at my work.
- what Python deprecations must be supported via a compatibility layer, so that Invenio(RDM) can be stable across Python versions.
- which deprecations in these compatibility layers can be dropped completely and when, in order to keep the maintenance burden low.
- what `requires-python =` (`pyproject.toml`) or `[requires] python_version =` (`Pipfile`) or `python_requires =` (`setup.cfg`) constraints to apply to which module/application, in order to enforce the policy.
- what versions of Python should be used for the continuous integration tests, in order to test well but be efficient with test runners.
- how to deal with yearly release cadence of the Python language, in order to stay relevant (secure/performant).

More than just describing what the policy is, this RFC will try to outline how it is manifested.


## Detailed design

After maintainer meetings and online discussions, the following approach was outlined. The problem space was divided at a high-level between:
InvenioRDM v14 onwards, InvenioRDM periods of overlap with deprecated Python versions, and the wider Invenio framework.

### InvenioRDM — v14 and onwards

Starting with InvenioRDM v14, the idea is to set the minimum required Python version to a specific version and keep it so until the InvenioRDM release of the year where that Python version is outdated OR until maintainers decide to change that minimum version. This version will be tested to ensure backwards compatibility over time. That version of Python will also be the *maximum officially supported Python version* at time of release of InvenioRDM. This applies to RDM specific modules and RDM instances.

Maintainers should tend towards not changing the minimum Python version frequently as upgrades like these can be costly in time and resource to implement. Every year is too frequent. Five years is typically how long a Python version is officially supported by the Python community. Somewhere in the middle or past it is reasonable. There must be good reasons (significant performance gains, significant API benefits, significant backward-compatible efforts that can be dropped...) to change the minimum Python version.

**Note** InvenioRDM and Python are on different release cadences with new Python versions typically releasing annually early October while InvenioRDM releases annually late July.

#### Applicable package distributions

This policy applies to the InvenioRDM specific modules:

- invenio-app-rdm
- invenio-audit-logs
- invenio-checks
- invenio-collections
- invenio-communities
- invenio-drafts-resources
- invenio-github/vcs
- invenio-jobs
- invenio-notifications
- invenio-rdm-records
- invenio-requests
- invenio-sitemap
- invenio-users-resources
- invenio-vocabularies

As InvenioRDM drives the creation of new modules, those should be considered to fall under this category by default. Exceptions will be indicated.

#### Example for InvenioRDM v14

For InvenioRDM v14, the target minimum version is Python 3.14 (it turns out nice that both have 14 in the version number). Here is how Python support would look like pre and post release of that version:

| Timeline        | Context                                                           | Officially supported Python version | Max. Compatible Python version |
| --------------- | ----------------------------------------------------------------- | ----------------------------------- | ----------------------------- |
| 2025-10         | post Python 3.14 release — pre RDMv14 release — development       | 3.14                                | 3.14                          |
| 2026-08         | RDMv14 release — users + development                              | 3.14                                | 3.14                          |
| 2026-10         | post Python 3.15 release — pre RDMv15 release — development       | 3.14                                | 3.15                          |
| 2026-10         | post Python 3.15 release — pre RDMv15 release — users             | 3.14                                | 3.14                          |
| 2027-08         | RDMv15 release — users + development                              | 3.14                                | 3.15                          |
| ...             | ...                                                               | 3.14                                | ...                           |
| 2028-10 (e.g.,) | post Python 3.17 release, decided to be new minimum — development | 3.17                                | 3.17                          |
| ...             | ...                                                               | 3.17                                | ...                           |
| 2030-10         | post Python 3.19 release — pre RDMv19 release — development       | 3.19                                | 3.19                          |

Although a version of InvenioRDM could un-officially support a newly released version of Python, official support is only guaranteed
when a dedicated release is made that officially supports it.

#### Continuous integration (CI) tests
The CI should test the minimum version required and the maximum supported. Skip in-between versions as they would have been tested in the past and this keeps the CI efficient in aggregate *and* per individual PR.

#### Reasoning
By adopting this policy, we ...

- ... benefit from the longest support window by jumping to 3.14 as minimum version for RDMv14
- ... take advantage of Python performance gains as a baseline
- ... don't have to change minimum version every year and constantly churn
- ... are alerted and account for upcoming deprecations in development as soon as official next latest Python version is out
- ... align Python and InvenioRDM version numbers which is nice

### InvenioRDM — periods of overlap with deprecated Python versions

The Python support policy outlined above leads to a scenario where a newly released InvenioRDM and the previous, but still supported for 6 months per [RDM support policy](https://inveniordm.docs.cern.ch/releases/maintenance-policy/), do not share the same mimimum required Python version for that overlap period. This could be a rare occurrence (e.g., every 5 years), but it has to be accounted for.

As is, it would mean that during that period of time, any fix would have to be compatible with the previous minimum Python version — a different and unsupported (by Python community) one.

The current (at time of writing) scenario of RDMv13 is illustrative of this "periods of overlap with deprecated Python versions" section. Note however that Python 3.10 would have been the next smallest
version, but it was never officially supported by InvenioRDM, so 3.11 would probably be the next minimum supported version (per the second option).

### Invenio framework modules

Invenio-wide modules like invenio-app,-search... (as opposed to InvenioRDM-specific ones like invenio-records-resources,-rdm-records ...) are not driven by the InvenioRDM cadence and should be even more stable. They should keep supporting the minimum Python version they did and be independent of RDM.

For those, the lowest but still supported Python version needs to be supported. They must not be incompatible with the most recent Python version.

#### Applicable package distributions

Any Invenio module not covered by the InvenioRDM section above falls under this policy.

#### Example

For example, for `invenio-base`

| Timeline | Context                  | Min. Python version required | Max. Python version supported |
| -------- | ------------------------ | ---------------------------- | ----------------------------- |
| 2025-10  | post Python 3.14 release | 3.10                         | 3.14                          |
| 2026-10  | post Python 3.15 release | 3.11                         | 3.15                          |

For Invenio module, the delineation between development and user readiness is not as clear as for InvenioRDM. Any change in supported range should be accounted for soon after.

#### Continuous integration tests

As laid out, it becomes apparent that a single shared reusable GitHub Action test workflow will not be able to cover all modules. A framework module with a minimum Python version of 3.8 would either not be tested by such a workflow or the addition of 3.8 to the workflow's tested versions would be incompatible to an InvenioRDM module that requires at least Python 3.14. Add to the mix the above note with respect to CI bounds in the context of periods of overlap with deprecated Python versions and the need for different test ranges (potentially workflows/branches) becomes apparent.

To be efficient with CI resources, only the minimum and maximum versions should be explicitly tested with in-between versions skipped.
To achieve this either:

**Same workflow, per repo ranges**. The same test workflow is adopted by all repositories, but each repository defines their own Python version range. This has a lot of range repetition, but otherwise keeps things simple.

**One test workflow for Invenio modules and one for InvenioRDM modules (both versioned)**. Each of these test workflows defines its default range and therefore addresses repetition and enforces uniformity. To account for periods of overlap with deprecated Python versions, new versions of these workflows would exist with the appropriate range. Specific version of workflows would be used. This adds some complexity to the approach.

## How we teach this

We can document the gist of it in the docs in the [Maintenance Policy](https://inveniordm.docs.cern.ch/releases/maintenance-policy/) section.

By having the appropriate workflows, this will also be conveyed to developers.

## Drawbacks

This lays out potentially long support times for some Python versions (up to 5 year) which means accounting for deprecations for a long time and sometimes not using helpful newer features. But we get stability and set ourselves up to work within our available development resources in exchange.
The minimum Python version required by InvenioRDM can also be changed by maintainers to alleviate some of this.

Other drawbacks were inlined.

## Alternatives

Alternatives were inlined.

## Unresolved questions

Unresolved questions were inlined.

## Resources/Timeline

This should be adopted as soon as possible since minimum Python version of InvenioRDM v13 is Python 3.9 which went out of support at end of 2025-10.
