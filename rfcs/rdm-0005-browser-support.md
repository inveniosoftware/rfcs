- Start Date: 2019-10-02
- RFC PR: [5](https://github.com/inveniosoftware/rfcs/pull/5)
- Authors: Guillaume Viger

# Browser support policy

## Summary

InvenioRDM commits to supporting secure, evergreen browsers that are themselves
supported by their vendors.

## Terminology

Evergreen browser: A browser that keeps itself up-to-date without prompting the user.

## Motivation

- As an InvenioRDM core or extension/plugin developer, I want to take advantage
of modern Web APIs such as flexbox, websockets, File API, modern Javascript (ES2015+)...
so that I can provide more powerful features at reduced development and maintenance
(polyfills...) cost.

- As an InvenioRDM hoster, running an application that requires evergreen secure
browsers benefits me, since it ensures my users are running secure browsers
which reduces my surface of attack and reduces my support costs, because evergreen
browsers continually fix their bugs and security holes.

## Detailed design

The following RFC outlines a support policy for a set of browsers and their versions.
It primarily serves the purpose to clarify for developers if a
certain feature can be used or not, and to reassure hosters that they can rely
on having bugs fixed for supported browsers.

The policy is by no means a guarantee that all supported browsers are working
with Invenio, as this would require extensive automated testing for all
browsers/versions over all Invenio modules which is not feasible at this point.

### Supported browsers

Browsers with a market share of at least 5% are supported. Currently that
includes:

- Google Chrome
- Mozilla Firefox
- Microsoft Edge / Internet Explorer
- Apple Safari

Only auto-updating browsers (evergreen browsers) are supported in
order to ensure a secure environment for Invenio to run in.

### Supported versions

Only browser versions supported by their vendor are supported by Invenio.

Both desktop and mobile versions of the browsers should be supported.

Special note on Internet Explorer: Only IE v11 is supported.

### What does it mean for a browser to be supported?

- Best effort is made to ensure that all client-side code (HTML/CSS/JavaScript)
  work in all supported browsers. This usually translates into supporting the
  earliest supported version. Use https://caniuse.com/ to check if a specific
  feature is supported.

- Bugs affecting any supported version will be fixed (resources permitting).

Specifically, if a browser is supported, it does not mean that Invenio is *guaranteed* to
work with the browser, just that we do our best to ensure that Invenio works
with that browser, and that bugs affecting the browser are fixed if discovered. This
is because it is unfeasible at this stage to rigorously test all browser
versions.

### How do I know if I can use a certain feature?

The service https://caniuse.com is your friend. In general, use features widely
adopted by all browser platforms and that have been around for more than 1 release.
Doing so, this browser support policy will come naturally.

### What does it mean for a browser not to be supported?

Bugs affecting unsupported browsers may not be fixed, and no effort is made to
test if Invenio works with the specific browser. In many cases however, an
unsupported browser is likely to still work with Invenio as we strive to use
widely adopted features only.

## How we teach this

A version of the browser support policy should be added to
[Invenio documentation](https://invenio.readthedocs.io/en/latest/) next to the
"Maintenance policy") as the official reference.

Emphasis should be placed on a TLDR at the top of the policy that points to
https://caniuse.com to allow developers to quickly determine if a specific
feature can be used.

## Drawbacks

Some of the stakeholders' environments may not be amenable to use evergreen
browsers. For instance, in the bio-medical domain, there are researchers that
use their own laptop with full autonomy of software, but there are also
research coordinators and staff librarians or clinicians that use organizational
IT resources that are locked down to specific Windows versions and browsers.

To confirm the usefulness of this support policy, we could get feedback from
current hosters about their current browser demographics, current trends and any
upcoming institutional IT policies they might be aware of. A report containing
current marketshare as of current month with differential with respect to same
month last year and any extra information could be produced by each organizations.
Global report trends with same numbers should also be found to compare against
general trends.

If the trends show much lower adoption of evergreeen browsers, we should revise
our support policy.

## Alternatives

We support a wider range of browsers at development effort and support cost.

## Unresolved questions

See drawbacks for adoption percentages.
