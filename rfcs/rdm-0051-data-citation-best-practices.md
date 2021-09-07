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

Most of the recommendations are implemented by invenioRDM, two of them by using DataCite DOIs. What is missing is mainly documentation for end users, and proper handling of givenName and familyName, to resolve issues with citation and bibtex formatting. HTTP link headers is a nice optional feature that should be added.

The following is the DCIP recommendation for documentation:

> The repository must provide documentation about how data should be cited, how metadata can be obtained, and who to contact for more information. This documentation should follow the recommendations in this document, the DCIP Data Citation Primer, community recommendations provided by a number of organizations, but should also address the specifics of that particular data repository.

| No  | Label                           | Status                                                                                                          |
| --- | ------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| 1   | Persistent identifier           | Implemented by using DataCite DOIs.                                                                             |
| 2   | Granularity                     | Implemented by using versions and part/isPartOf relationships in DataCite metadata.                             |
| 3   | Landing page                    | Implemented in invenioRDM.                                                                                      |
| 4   | PID in landing page             | Implemented using `citation_doi` meta tag and schema.org/JSON-LD. Consider adding the `DC.identifier` meta tag. |
| 5   | Documentation for data citation | Documentation for how to cite data in invenioRDM is currently missing.                                          |
| 6   | Metadata required for citation  | Implemented using schema.org/json-ld.                                                                           |
| 7   | Schema.org/json-ld              | Implemented. Re-evaluate the schema.org metadata that should be exported.                                       |
| 8   | HTML meta tags                  | Implemented. Consider using some additional tags such as `DC.identifier`.                                       |
| 9   | BibTeX                          | Implemented. Need to evaluate implementation.                                                                   |
| 10  | Content negotiation             | Implemented by using DataCite DOI content negotation. Add content negotiation in the invenioRDM API.            |
| 11  | HTTP link headers               | Not yet implemented.                                                                                            |

## Example

> Show a concrete example of what the RFC implies. This will make the consequences of adopting the RFC much clearer.

Implementing the RFC will lead to better discovery of content hosted in inveniRDM, e.g. by Google Dataset Search.

## How we teach this

> What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing Invenio patterns, or as a wholly new one?

No new terminology is needed, this RFC extends existing patterns.

> Would the acceptance of this proposal mean the Invenio documentation must be re-organized or altered? Does it change how Invenio is taught to new users at any level?

This RFC proposes better documentation for how to cite data. The focus for the documentation change is on end users.

> How should this feature be introduced and taught to existing Invenio users?

This is an optional feature.

## Drawbacks

> Why should we _not_ do this? Please consider the impact on teaching Invenio, on the integration of this feature with other existing and planned features, on the impact of the API churn on existing apps, etc.

This RFC increases the complexity of metadata for personal names. Conversion of already existing name metadata to support givenName and familyName is not trivial.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

This follows the path DataCite has taken regarding givenName and familyName and the lessons learned. It is an achievable goal but the transition is not trivial.

## Unresolved questions

Content negotiation (#10) and http link headers (#11) are optional in the published Data Citation recommendations. This RFC suggests to implement them.

## Resources/Timeline

We have resourced five weeks of work to implement these recommendations. Given the current implementation status this is achievable, and the work should be completed by October 2021.
