- Start Date: 2021-08-18
- RFC PR: [#<PR>](https://github.com/inveniosoftware/rfcs/pull/<PR>)
- Authors: Martin Fenner
- State: DRAFT

# <RFC Data Citation Best Practices>

## Summary

Implement the 11 data citation best practices as described in [Fenner et al. 2019](https://doi.org/10.1038/s41597-019-0031-8) in invenioRDM.

## Motivation

The best practices described in the paper are an output of the Force11 DCIP project and reflect the thinking of a broad community of data repositories and other actors (with a focus on life sciences because of the grant funding by NIH).

## Detailed design

### Guidelines for repositories (1-5 required, 6-9 recommended, 10-11 optional, from https://doi.org/10.1038/s41597-019-0031-8)

1. All datasets intended for citation must have a globally unique persistent identifer that can be expressed as an unambiguous URL.
2. Persistent identifers for datasets must support multiple levels of granularity, where appropriate.
3. The persistent identifer expressed as an URL must resolve to a landing page specifc for that dataset, and that landing page must contain metadata describing the dataset.
4. The persistent identifer must be embedded in the landing page in machine-readable format.
5. The repository must provide documentation and support for data citation.
6. The landing page should include metadata required for citation, and ideally also metadata facilitating discovery, in human-readable and machine-readable format.
7. The machine-readable metadata should use schema.org markup in JSON-LD format.
8. Metadata should be made available via HTML meta tags to facilitate use by reference managers.
9. Metadata should be made available for download in BibTeX and/or another standard bibliographic format.
10. Content negotiation for schema.org/JSON-LD and other content types may be supported so that the persistent identifer expressed as URL resolves directly to machine-readable metadata.
11. HTTP link headers may be supported to advertise content negotiation options

### Implementation status

| No  | Label                           | Status                                                                                                          |
| --- | ------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| 1   | Persistent identifier           | Implemented by using DataCite DOIs.                                                                             |
| 2   | Granularity                     | Implemented by using versions and part/isPartOf relationships.                                                  |
| 3   | Landing page                    | Implemented in invenioRDM.                                                                                      |
| 4   | PID in landing page             | Implemented using `citation_doi` meta tag and schema.org/JSON-LD. Consider adding the `DC.identifier` meta tag. |
| 5   | Documentation for data citation | Documentation for how to cite data in invenioRDM is currently missing.                                          |
| 6   | Metadata required for citation  | Implemented using schema.org/json-ld. Should add support for givenName and familyName.                          |
| 7   | Schema.org/json-ld              | Implemented. Should add support for givenName and familyName.                                                   |
| 8   | HTML meta tags                  | Implemented.                                                                                                    |
| 9   | BibTeX                          | Implemented. Names are not formatted correctly (should John Doe not Doe, John).                                 |
| 10  | Content negotiation             | Implemented via DataCite                                                                                        |
| 11  | HTTP link headers               | Not yet implemented.                                                                                            |

## Example

> Show a concrete example of what the RFC implies. This will make the consequences of adopting the RFC much clearer.

> As with other sections, use it if it makes sense for your RFC.

## How we teach this

> What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing Invenio patterns, or as a wholly new one?

> Would the acceptance of this proposal mean the Invenio documentation must be re-organized or altered? Does it change how Invenio is taught to new users at any level?

> How should this feature be introduced and taught to existing Invenio users?

## Drawbacks

> Why should we _not_ do this? Please consider the impact on teaching Invenio, on the integration of this feature with other existing and planned features, on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still TBD? Use it as a todo list for the RFC.

## Resources/Timeline

> Which resources do you have available to implement this RFC and what is the overall timeline?
