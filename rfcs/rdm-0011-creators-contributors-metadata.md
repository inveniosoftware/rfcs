- Start Date: 2019-10-22
- RFC PR: [#11](https://github.com/inveniosoftware/rfcs/pull/11)
- Authors: Tom Morrell, Guillaume Viger

# Contributors / Creators Schema Definition

## Summary

This RFC defines the JSON schema field(s) holding the
contributors, creators, authors... of a bibliographic record. Because defining
the whole schema is a big effort, we have broken it off into constituent parts.


## Motivation

- As an InvenioRDM hoster, I want to benefit from a common and already defined
  bibliographic schema, so that I don't have to make my own.
- As an InvenioRDM hoster, I want to use a schema that is compatible with
  DataCite schema (and others), so that I can inter-operate with other metadata
  formats.
- As an InvenioRDM developer, I want to code to a common metadata
  schema, so that I don't repeat code and so that I benefit from and to others'
  work.
- As a researcher, I want to cite a record with its authors/creators,
  so that my citation follows standards.
- As a researcher, I want a DOI for my record which is only possible
  if the equivalent `Creator` field is passed to the minting authority (DataCite).
- As an InvenioRDM hoster, I want to provide a generic name field, so that
  international authors and contributors can enter their full name in their own
  way.
- As an InvenioRDM hoster, I want to support person and organization identifiers,
  so that I can disambiguate between entities.


## Detailed design

Creators and contributors can be people or organizations. Their schema structure
will be reused. Their definition in the `"definitions"` section is:

```json
{
  "definitions": {
    "identifier": {
      "type": "object",
      "description": "A generic identifier.",
      "additionalProperties": true,
      "type": "object",
      "properties": {
        "scheme": {
          "type": "string",
          "description": "Kind of identifier.",
          "enum": [
            "ads",
            "ark",
            "arxiv",
            "bibcode",
            "bioproject",
            "biosample",
            "doi",
            "ean13",
            "ean8",
            "eissn",
            "ensembl",
            "genome",
            "gnd",
            "hal",
            "handle",
            "igsn",
            "isbn",
            "isni",
            "issn",
            "istc",
            "lissn",
            "lsid",
            "orcid",
            "pmcid",
            "pmid",
            "purl",
            "refseq",
            "ror",
            "sra",
            "uniprot",
            "upc",
            "url",
            "urn",
            "w3id"
          ]
        },
        "value": {
          "type": "string"
        }
      },
      "required": ["type", "value"]
    }
  },
  "person_or_org": {
    "type": "object",
    "description": "A person or organization.",
    "additionalProperties": false,
    "properties": {
      "affiliations": {
        "type": "array",
        "description": "Affiliation(s) (if person) for the purpose of the specific record.",
        "uniqueItems": true,
        "items": {
          "type": "string"
        }
      },
      "email": {
        "type": "string",
        "description": "Contact email for the purpose of this specific record.",
        "format": "email"
      },
      "full_name": {
        "type": "string",
        "description": "Full name of person or organization. Personal name format: family, given."
      },
      "ids": {
        "type": "array",
        "description": "List of IDs related with the person or org. (e.g., ORCID, RORid)",
        "uniqueItems": true,
        "items": { "$ref": "#/definitions/identifier" }
      },
      "nameType": {
        "type": "string",
        "description": "Is this an organization or a person?",
        "enum": ["Organizational", "Personal"]
      }
    },
    "required": [
      "full_name"
    ]
  }
}
```

The proposed metadata field definition is as follows:

```json
"stakeholders": {
  "description": "Contributors in order of importance.",
  "minItems": 1,
  "type": "array",
  "items": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "stakeholder": { "$ref": "#/definitions/person_or_org" },
      "role": {
        "type": "string",
        "description": "Role the stakeholder has in this record.",
        "enum": [
          "Author",
          "ContactPerson",
          "DataCollector",
          "DataCurator",
          "DataManager",
          "Distributor",
          "Editor",
          "HostingInstitution",
          "Producer",
          "ProjectLeader",
          "ProjectManager",
          "ProjectMember",
          "RegistrationAgency",
          "RegistrationAuthority",
          "RelatedPerson",
          "Researcher",
          "ResearchGroup",
          "RightsHolder",
          "Sponsor",
          "Supervisor",
          "WorkPackageLeader",
          "Other"
        ],
        "type": "string"
      }
    },
    "required": ["stakeholder", "role"]
  }
}
```

See #Alternatives below for the other take on this field.


## How we teach this

Once implemented and used, this schema will be self explanatory. The RFC will
be a good location to understand the thinking that went into the decisions.


## Drawbacks

Less of a drawback, but more of an implication, is that various InvenioRDM
modules that depend on a record schema will have to be amended to use this
schema. If adopted by other services, this schema will require migrations.

Although the generic approach is flexible, when it comes to defining a record
**mapping** from this schema, there might be some more effort involved than
in using a more set and explicit approach.


## Alternatives

The main alternative to the definition above is to split the `stakeholders`
field into multiple fields like DataCite already does:

```json
"creators": {
  "type": "array",
  "description": "Creators in order of importance.",
  "items": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "creator": { "$ref": "#/definitions/person_or_org" },
    }
  }
},
"contributors": {
  "type": "array",
  "description": "Contributors in order of importance.",
  "items": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "contributor": { "$ref": "#/definitions/person_or_org" },
      "role": {
        "type": "string",
        "description": "Role the contributor has in this record.",
        "enum": [
          "ContactPerson",
          "DataCollector",
          "DataCurator",
          "DataManager",
          "Distributor",
          "Editor",
          "HostingInstitution",
          "Other",
          "Producer",
          "ProjectLeader",
          "ProjectManager",
          "ProjectMember",
          "RegistrationAgency",
          "RegistrationAuthority",
          "RelatedPerson",
          "Researcher",
          "ResearchGroup",
          "RightsHolder",
          "Sponsor",
          "Supervisor",
          "WorkPackageLeader",
          "Other"
        ]
      }
    },
    "required": ["contributor", "role"]
  }
}
```

## Unresolved questions

Is there a convention for field order? For properties fields, alphabetic order
seems good. For other fields, having the `type` first helps parse the field I find.

Using `ref`, should we wrap them or use them directly via `one_of`. For instance,
should the pattern be:

```json
"<plural form>": {
  "type": "array",
  "description": "Many of this type.",
  "items": {
    "type": "object",
    "properties": {
      "<singular form>": { "$ref": "<definition>" }
    }
  }
}
```

OR

```json
"<plural form>": {
  "type": "array",
  "description": "Many of this type.",
  "items": {
    "one_of": [{ "$ref": "<definition>" }]
  }
}
```

?

Where should `owners` fit in this design (if at all)? The meaning of `owners`
is already overloaded: original uploader, contact person, maintainer or
copyright holder? Other than via `contributor`, rights ownership can't be
assigned to the `owner` or the `creator` right now.
