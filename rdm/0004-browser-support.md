- Start Date: 2019-10-02
- RFC PR: [5](https://github.com/inveniosoftware/rfcs/pull/5)
- Authors: Guillaume Viger

# Support Evergreen Browsers

## Summary

InvenioRDM commits to supporting secure, evergreen browsers that are themselves
supported by their vendors.

## Terminology

Evergreen Browser: A browser that keeps itself up-to-date without prompting the user.

## Motivation

- As an InvenioRDM core or extension/plugin developer, I want to take advantage
of modern Web APIs such as flexbox, websockets, File API, modern Javascript (ES2015+)...
so that I can provide more powerful features at reduced development and maintenance
(polyfills...) cost.

- As an InvenioRDM hoster, running an application that requires evergreen secure
browsers benefits me since it ensures my users are running secure browsers
which reduces my surface of attack and reduces my support costs because evergreen
browsers continually fix their bugs and security holes.


## Detailed design

Microsoft Edge, Google Chrome, Mozilla Firefox are
supported. Browsers from derivatives should work by consequence.

Apple Safari is updated with Apple's OS'es. The latest available major
version along with previous 2 major versions should be supported. For example,
if Safari 12 is the currently available version, then Safari 11 and 10 are
supported.

To account for mobile browsers that sometimes lag behind their desktop
equivalent and to prevent usage of bleeding edge features that are
not widely-adopted, the Edge, Chrome and Firefox user base can be assumed to
be at least running the fifth browser version prior to the current one. For
example, if Chrome 80 is the current version, then features in Chrome 75 that
are still around are fair-game. If there is a known bug in Chrome 76 fixed in
Chrome 77, a fix should still be provided (although time heals all wounds clearly).

"Support" here translates to:

- Only features available from the earliest covered browsers are used.
- Bugs affecting any covered versions have to be addressed.

For IE10 and less and other non-supported browsers, a warning
message should be displayed and recommended browsers suggested.


## How we teach this

Browser support is a frequently asked question and has been implicit so far.
Our stance should be publicly placed in the FAQ section. This is just making
explicit an implicit design decision. The FAQ entry is targeted at users and
hosters.

This fact should also be re-stated in the plugin/extension documentation, so
that third-party developers know the features they can rely on.

In talking and writing, the support window strategy can be referred as
"rolling support". Because the strategy is relative, hardcoded versions
don't have to be documented and updated overtime. For developers it is also
more convenient to determine what features they can use.

## Drawbacks

Some of the stakeholders' environments may not be amenable to use evergreen
browsers. For instance, in the bio-medical domain, there are researchers that
use their own laptop with full autonomy of software, but there are also
research coordinators and staff librarians or clinicians that use organizational
IT resources that are locked down to specific Windows versions and browsers.

To confirm the usefulness of this support policy, we should get feedback from
current hosters about their current browser demographics, current trends and any
upcoming institutional IT policies they might be aware of. A report containing
current marketshare as of current month with differential with respect to same
month last year and any extra information should be produced by each organizations.
Global report trends with same numbers should also be found to compare against
general trends.

If the trends show much lower adoption of evergreeen browsers, we should revise
our support policy.


## Alternatives

We support a wider range of browsers at development effort and support cost.


## Unresolved questions

See drawbacks for adoption percentages.
