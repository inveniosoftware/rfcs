- Start Date: 2021-08-18
- RFC PR: [#<PR>](https://github.com/inveniosoftware/rfcs/pull/<PR>)
- Authors: Martin Fenner
- State: DRAFT

# <RFC Data Citation Best Practices>

## Summary

Implement the 11 data citation best practices as described in Fenner _et al._ 2019 in invenioRDM.

Fenner, M., Crosas, M., Grethe, J. S., Kennedy, D., Hermjakob, H., Rocca-Serra, P., … Clark, T. (2019). A data citation roadmap for scholarly data repositories. Scientific Data, 6(1). https://doi.org/10.1038/s41597-019-0031-8

## Motivation

The best practices described in the paper are an output of the [Force11 DCIP project](https://www.force11.org/group/dcip) and reflect the work of a broad community of data repositories and other actors (with a focus on life sciences because of the grant funding by NIH).

## Detailed design

### Guidelines for repositories (1-5 required, 6-9 recommended, 10-11 optional)

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

Most of the recommendations are implemented by invenioRDM, two of them by using DataCite DOIs. What is missing is mainly documentation for end users, and proper handling of givenName and familyName, which creates issues with citation and bibtex formatting. HTTP link headers is a nice optional feature that should be added.

The following is the DCIP recommendation for documentation:

> The repository must provide documentation about how data should be cited, how metadata can be obtained, and who to contact for more information. This documentation should follow the recommendations in this document, the DCIP Data Citation Primer, community recommendations provided by a number of organizations, but should also address the specifics of that particular data repository.

| No  | Label                           | Status                                                                                                          |
| --- | ------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| 1   | Persistent identifier           | Implemented by using DataCite DOIs.                                                                             |
| 2   | Granularity                     | Implemented by using versions and part/isPartOf relationships.                                                  |
| 3   | Landing page                    | Implemented in invenioRDM.                                                                                      |
| 4   | PID in landing page             | Implemented using `citation_doi` meta tag and schema.org/JSON-LD. Consider adding the `DC.identifier` meta tag. |
| 5   | Documentation for data citation | Documentation for how to cite data in invenioRDM is currently missing.                                          |
| 6   | Metadata required for citation  | Implemented using schema.org/json-ld. Should add support for givenName and familyName.                          |
| 7   | Schema.org/json-ld              | Implemented. Should add support for givenName and familyName.                                                   |
| 8   | HTML meta tags                  | Implemented. Consider using some addition tags such as `DC.identifier`.                                         |
| 9   | BibTeX                          | Implemented. Names are not formatted correctly (should John Doe not Doe, John).                                 |
| 10  | Content negotiation             | Implemented by using DataCite DOIs. Consider implementing in the invenioRDM API.                                |
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

We have resourced five weeks of work to implement these recommendations. Given the current implementation status this is achievable, and the work should be completed by October 2021.
