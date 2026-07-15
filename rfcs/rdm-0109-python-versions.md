- Start Date: 2025-10-24
- RFC PR: [#109](https://github.com/inveniosoftware/rfcs/pull/109)
- Authors: Guillaume Viger

# Python support in Invenio/RDM ecosystem

This RFC outlines the Python support policy for InvenioRDM and the related Invenio ecosystem. In particular, it outlines how the policy relates to concrete implementations/use-cases/actions. Some aspects are left flexible for agile decision making in future contexts.


## Motivation

As an Invenio(RDM) developer (operator, contributor or maintainer), I want to know:

- what Python features I can use or not, so that I am effective at my work.
- what Python deprecations must be supported via a compatibility layer, so that Invenio(RDM) can be stable across Python versions.
- which deprecations in these compatibility layers can be dropped completely and when, in order to keep the maintenance burden low.
- what `requires-python =` (`pyproject.toml`) or similar constraints to apply to which module/application.
- what versions of Python should be used for the continuous integration tests, in order to test well but be efficient with test runners.
- how to deal with yearly release cadence of the Python language, in order to stay relevant (secure/performant) but avoid churn.

As an InvenioRDM instance operator, I want to know which version of Python I can use, so that I can setup my environment and know what support I can expect from the community.


## Summary of approach

- Each InvenioRDM major release has one chosen Python version as the **anointed version** the InvenioRDM project supports.
    * There can be multiple anointed version supported at the same time in periods of transitions.
- Support means a guarantee that
    * the chosen version works for that InvenioRDM version
    * the bugs related to the intersection of the Python and InvenioRDM versions will be addressed
    * the features of that version can be used in InvenioRDM-specific code
    * the deprecated features of that version do not have to be supported in InvenioRDM-specific code
- The anointed version is made publicly known in the release notes and documentation.
- The anointed version is itself officially supported by the Python community.
- Best effort is made to keep the same anointed version for a number of major InvenioRDM versions. A different anointed version can only be selected for a new major InvenioRDM version.
- All Invenio and InvenioRDM modules are compatible with the anointed version for its lifespan.
- That lifespan starts on a major InvenioRDM release and ends when the last InvenioRDM release that had it as its anointed version is retired (reached end-of-life).
- Invenio modules and InvenioRDM instances *may* be compatible with other versions of Python. This policy approach focuses on establishing inclusive guarantees as opposed to exclusive ones.
    * by modules adopting/dropping Python version features, some versions will naturally be excluded
- Overtime, this policy of one anointed Python version will be used for modules in the wider Invenio ecosystem (not just InvenioRDM one) in order to simplify processes across all modules.

The rest of this document details these core policy points and their consequences in terms of continuous integration, the release of new Python versions, explicit requirement markers, transition periods, etc.


## Detailed design

### Terms

**Official version**

An official and actively supported (by Python community) version of Python. Such versions and their lifespan can be consulted here: https://devguide.python.org/versions/ . Non-CPython interpreters may work, but those are not guaranteed by this policy.

> [!NOTE]
> InvenioRDM and Python are on different release cadences with new Python versions typically releasing annually early October while InvenioRDM releases annually late July.

**Anointed version**

A chosen official version of CPython for which the InvenioRDM community makes the above summarized support guarantees for the lifespan of its associated InvenioRDM version.

**InvenioRDM-specific modules**

These are modules "exclusive" to InvenioRDM

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

Any use of these modules outside InvenioRDM is (obviously) possible but not supported.

**InvenioRDM-ecosystem modules**

These are modules used by InvenioRDM and under the `inveniosoftware` organization, but not exclusive to it. Most Invenio modules fall under this category. For example, `invenio-base` is such a module. It is also used in "raw" Invenio installations and in InvenioILS, thus making it non-exclusive to InvenioRDM.

For modules under `inveniosoftware` but not used by InvenioRDM, they are typically deprecated or exclusive to raw Invenio installations or InvenioILS. For non deprecated ones, the longer term goal would be to align them with this policy, but at time of RFC writing, they are unaffected by it.

**All modules**

When talking about "modules" in general, we refer to the combination of the two above groups unless otherwise noted (not literally all `inveniosotware` modules typically).

### Continuous integration (CI) tests

Continuous integration tests are the main way how this policy is explicitly manifested/enforced.

The **minimum** Python version in the test grid of a module *must* be:

- the smallest active anointed version if the module is InvenioRDM-specific, OR
- the smallest officially supported Python version at least (could be higher but no higher than the anointed version) otherwise

Any active anointed versions *must* explicitly be in the test grid of all InvenioRDM-specific and InvenioRDM-ecosystem modules. This ensures support.

The **maximum** Python version in the test grid of a module *should* be the latest officially supported Python version.
We don't make this a "must" as it is extra effort, and may burden the project with supporting compatibility with deprecated features early on. However, it is useful to identify and address breaking changes sooner and make the transition to a new anointed version easier because gradual.

In practice, because a new major InvenioRDM version is in development for some time, there might often be 2 anointed versions in the CI grid: one for the current one and one for the upcoming or previous one.

Another practical consideration to take into account is that most modules use the common `workflows/tests-python.yml` GitHub Action workflow. Defining the default Python versions for tests in that workflow to be the ones for an ecosystem module will correctly cover those modules (majority of modules), but not InvenioRDM-specific ones. For InvenioRDM-specific modules, their Python versions for tests will be overwritten in their own call to the shared workflow (unless defining a separate `workflows/tests-python-rdm.yml` that re-uses the latter with the appropriate set of python versions is possible). The overhead cost of such bifurcated support is what we want to drop in the longer run by aligning InvenioRDM-ecosystem modules to fully follow the anointed Python version approach.

> [!IMPORTANT]
> At the time of writing, there is a caveat to that approach whereby Python version 3.9 must still be tested since major CERN instances still use it. Once they are not using it, the normal application of this RFC would resume.

#### Examples

Let's say for InvenioRDM major version 14 released in July 2026, Python 3.14 is the anointed version. The CI grid for the following packages would be (disregarding the caveat about Python 3.9 above):

| Timeline/Context | Module | Python versions in test grid | Reasoning |
| ---------------- | ------ | ---------------------------- | --------- |
| InvenioRDM v14 Release | invenio-app-rdm | [3.14] | InvenioRDM-specific module<br> active anointed version is 3.14<br> highest official version is 3.14 |
| InvenioRDM v14 Release | invenio-app | [3.10, 3.14] | InvenioRDM-ecosystem module<br> active anointed version is 3.14<br> lowest official version is 3.10<br> |
| Python 3.15 release<br> 3.10 deprecation | invenio-app-rdm | [3.14, 3.15]<br> (3.15 is an option, and could have not been there) | InvenioRDM-specific module<br> active anointed version is 3.14<br> highest official version is 3.15 |
| Python 3.15 release<br> 3.10 deprecation | invenio-app | [3.11, 3.14, 3.15]<br> (3.15 an option, not a must) | InvenioRDM-ecosystem module<br> active anointed version is 3.14<br> lowest and highest official versions are now 3.11 and 3.15 |
| InvenioRDM v15 Release<br> anointed version kept the same | invenio-app-rdm | [3.14, 3.15] | InvenioRDM-specific module<br> active anointed version is 3.14<br> highest official version is 3.15 |
| Python 3.16 release<br> 3.11 deprecation | invenio-app-rdm | [3.14, 3.16] | InvenioRDM-specific module<br> active anointed version is 3.14<br> highest official version is now 3.16 |

Cases where different anointed versions overlap are covered in the "Transitions" section.

### Meta markers (requires-python / python_requires and co.)

`requires-python` (`pyproject.toml`) or equivalent *should* be present in modules. It should at least reflect the smallest official Python version. Ideally it would reflect the minimum required Python version for the module.

If the smallest official Python version is 3.11, then something of the form:

```toml
# pyproject.toml
[project]
# ...
requires-python = ">=3.11"
```

is expected. If features specific to Python 3.14 are used, then it would ideally be `requires-python = ">=3.14"`.
Because of the very large number of InvenioRDM-specific and InvenioRDM-ecosystem modules, this leniency is established.

The policy skews towards an affirmative approach as opposed to a negative one. These markers may exclude versions that a module could stil be compatible with if the minimum anointed version was chosen. The markers have a cost to their maintenance: whenever official versions change, they have to change. Adding more specific constraints can become an additional burden. If a developer wants to use a non-anointed Python version (within the `requires-python` range), they will have to attempt to run or install the code under that version to know if it is compatible.

Historically, the `classifiers` meta-marker has not accurately reflected which Python version any module is compatible with. As such we keep it that way to also save on maintenance effort. Removal of any minor Python versions is preferred
in fact.

### Docker images

For each active anointed version there *must* be a base Invenio(RDM) docker image supporting that Python version.

This is a core objective of this support policy: produce an Invenio(RDM) docker image that is guaranteed to work and off-the-shelf. Adopters with no Python version preferences should be confident that this image will work.

### Transitions

Two sequential major versions of InvenioRDM share a period of time when both are supported (see [RDM support policy](https://inveniordm.docs.cern.ch/releases/maintenance-policy/)), and as such, there will come a time when two different anointed versions are supported. In these cases, both are part of the CI test grid.

The decision to anoint a new Python version is made during development, so it will show up in the test grid before an official release. However, this version will usually be one that is or has already been in the test grid, so this typically doesn't bring new concerns.

Semver leaves room for debate about the version assignment implication of dropping and adopting language versions. For us, changes in anointed version are only done on major releases, so the major bump covers the change of supported language. In between those (in development), if supporting a new language version requires additions, a module should be minor bumped. If not, support can be claimed and not result in a version change. When an official version has reached its end-of-life and no changes were needed, a version change can also be skipped.

#### Examples

| Timeline/Context | Module | Python versions in test grid | Reasoning |
| ---------------- | ------ | ---------------------------- | --------- |
| InvenioRDMv17 in development<br> with 3.17 as upcoming new anointed version | invenio-app-rdm | [3.14, 3.17] | InvenioRDM-specific module<br> active anointed version is 3.14<br> upcoming anointed version is 3.17 |
| ... | ... | ... | ... |
| 5 months after InvenioRDMv17 release<br> Python 3.18 out | invenio-app-rdm | [3.14, 3.17]<br> (no 3.18 because  optional) | InvenioRDM-specific module<br> active anointed versions are 3.14 + 3.17 |
| 5 months after InvenioRDMv17 release<br> Python 3.18 out | invenio-app | [3.13, 3.14, 3.17, 3.18] (3.18 because optional) | InvenioRDM-ecosystem module<br> active anointed versions are 3.14 + 3.17<br> lowest and highest official versions are 3.13 and 3.18<br> |


## How we teach this

We can document the gist of it in the docs in the [Maintenance Policy](https://inveniordm.docs.cern.ch/releases/maintenance-policy/) section.

By having the appropriate workflows, this will also be conveyed to developers.


## Drawbacks

This lays out potentially long support times for some Python versions (couple years) which means accounting for deprecations for a long time and sometimes not using helpful newer features. But, we get stability and set ourselves up to work within our available development resources in exchange.

By not enforcing the anointed version as a `requires-python` (or equivalent) constraint, we don't make it "statically" fully known to adopters which versions won't work. Multiple independent adopters would potentially perform same work to come to same conclusions. This can be mitigated by mentioning in the release notes if it is already known that other Python versions will not work. The choice to do so has the advantage of not cutting off modules from being used by still valid official Python versions.

There is ambiguity as to how/when to decide to add a new official version to the CI, and when in development to choose the new anointed version. The policy doesn't remove the need for discussion around this topic. A posited rule of thumb is to touch base among maintainers and community partners at telecon once a year around November-December (after new Python version out). Aiming around keeping the same anointed version for 3 years or so is respectable. A positive take on this lack of determinism is that this approach is flexible to future contexts.

Other drawbacks were inlined.


## Alternatives

Alternatives were inlined.


## Discussion points / Unresolved questions

When will InvenioRDM-ecosystem modules shift to using the anointed version as their minimum test versions as well?
Communicating the change should be done sooner rather than later, since a period of time will have to elapse before that change can be applied.


## Implied action items

- Updating `requires-python` and such from modules
- Removing `classifiers` containing Python versions if any at will
- Changing the CI test grids and potentially creating a separate test workflow for InvenioRDM-specific module
- Settling on Python 3.14 for InvenioRDMv14 to start the policy.


## Resources/Timeline

This should be adopted as soon as possible. Guillaume is available to perform all implied actions.
