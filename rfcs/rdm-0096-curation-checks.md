RFC: Automated curation checks
===

- Start Date: 2024-12-16
- RFC PR: [#93](https://github.com/inveniosoftware/rfcs/pull/93)
- Authors: Alex Ioannidis

# Summary

A new mechanism for configuring and running curation checks on records to:

- allow communities to configure curation checks, run automatically on submitted records with feedback to submitters and curators.
- provide a flexible datamodel and API, that will allow internal (i.e. part of the Invenio application) and external (e.g. 3rd-party services) checks to be implemented.

# Motivation

InvenioRDM currently provides a basic set of validation checks in the form of required fields on the deposit form. These checks are only useful to guiding users into creating "valid" records ready for publishing.

The alternative approach for validating records is to go through a community submission workflow, where a **(human) curator** reviews the record and decides whether it is ready for publishing. Curators can directly edit the record, or provide feedback to the submitter.

To ease the curation process, we propose a new mechanism for configuring and running automated curation checks on records. This mechanism will allow communities to configure their own set of curation checks, that will run automatically on records either on submission, or at a later stage. Submitters and curators will be able to see the results of these checks and act on them.

## User stories

- As a community manager, I want to be able to configure a set of curation checks that will be run on records submitted to my community, so that I can ensure that records meet the quality standards of my community.
- As a community manager, I want to be able to run a set of curation checks on records that are already part of my community, so that I can ensure that records meet the quality standards of my community (and address them if necessary).
- As a submitter, I want to be able to see the results of the curation checks run on my record, so that I can address any issues before a human curator reviews it.
- As a curator, I want to be able to see the results of the curation checks run on a record, so that I can decide whether the record is ready for publishing.
- As a developer, I want to be able to implement custom curation checks on my instance, so that I can take advantage of external services or custom logic to validate records.

# Detailed design

High-level concepts:

- **Check**: a reusable definition of validation logic that can be run on a record. It encapsulates metadata, logic, and parametrization. Examples:
  - *Metadata has to comply with the following rules:*
      - *Authors must have persistent identifiers of type XYZ*
      - *If record is of type XYZ, then license must be of type ABC*
  - *Non-proprietary file types should be used*
- **Check Config**: concrete configuration of a *Check* in a community that will run on record submissions. This is an "instance" of a *Check*, with specific parameters, stored in the database. Examples:
  - *Run the "Metadata checks" check with two configured rules:*
      - *Authors must have persistent identifiers of type `ORCiD` or `GND`*
      - *If Resource type is `Dataset` require a `Creative Commons` license*
  - *Run the "File types" check for `tabular data` and `image` files*
- **Check Run**: a single run of a *Check Config* on a **record revision**, tracking the execution of the parametrized *Check*. It contains the status and result of the check.


## Datamodel

### Check

- A Check encapsulates logical conditions that can be run against a record, producing a result. It's a Python class containing the metadata, logic, and parameters of a check.
- Checks are stored in a global registry and have a unique identifier.

```python
class Check:

    id: str

    # Display fields
    name: str
    description: str

    params: marshmallow.Schema

    # If `False` the check will run as a Celery task
    sync: bool = True

    # Main logic of the check
    def run(self, record: Record): pass

    # External or background checks can be cancelled/retried
    can_cancel: bool = False
    def cancel(self): pass

    can_retry: bool = False
    def retry(self): pass

    # External checks might need their status to be "refreshed"
    can_refresh: bool = False
    def refresh(self): pass

    # External checks need a callback to update their status/result
    can_update: bool = False
    def update(self, status: str, result: dict): pass


# Glbal checks registry
type ChecksRegistry = dict[str, Check]
```

### Check Config

- A Check Config is a concrete configuration of a Check in a community.
- `severity` tells if the check is required (`error`), recommended (`warn`), or informational (`info`).
- Check Configs are persisted in the database.

```python
class CheckConfig(Model):

    id: PK[UUID]
    community_id: FK[Community.id]
    check_id: Enum[Check.id]
    params: dict
    severity: Enum["info", "warn", "error"]
    order: int
    enabled: bool
```

### Check Run

- A Check Run is a single run of a Check Config for a record or draft revision.
- Check Runs are persisted in the database.

```python
class CheckRun(Model):

    id: UUID
    config_id: FK[CheckConfig.id]
    # FIXME: Should we store a "snapshot" of the config params, severity, etc.?
    #        What if the community changes the config after the check has run?

    # These describe "what" we're running the check on
    record_id: UUID
    is_draft: bool
    revision_id: int

    # These keep track of the execution state of the check
    status: Enum["pending", "done", "error"]
    state: dict
    result: dict
```

### Result schema

Check Run results are stored as JSON objects following a base schema that accomodates different result types from the implemented validation checks. We want though to be able to evolve the schema over time, e.g. to provide rich UI/UX to guide users with suggestions.

The schema loosely follows that of the [Language Server Protocol](https://microsoft.github.io/language-server-protocol/), which is used by code editors for providing a unified API that different programming languages can target with their tooling (i.e. compilers, linters, etc.). We can standardize results to contain the following types of items:

- Diagnostics: a message that describes an issue found by the check
- Actions: a suggestion for how to fix the problem (e.g. set a field's value)

A client can interpret these results and display them to the user, and potentially even take actions.

```json
{
  "diagnostics": [
    {
      // Top-level message describing the issue
      "message": "Creative Commons license is required for datasets.",
      "severity": "error",
      // Path to the field that has the issue
      "error_path": ["metadata.rights"],
      // We include the "offending" values, so that the UI knows when a check can
      // be considered "addressed". In this case, the value of either the
      // resource type or the rights field must be updated.
      "values": [
        {"path": "metadata.resource_type.id", "value": "dataset"},
        {"path": "metadata.rights.0.id", "value": "mit"},
      ]
    }
  ],
  "actions": [
    {
      "title": "Fill in Creative Commons license",
      "command": "set_field_options",
      "arguments": ["metadata.rights.0.id", ["cc-by-4.0", "cc-by-sa-4.0", ...]]
    }
  ]
}
```

## Lifecycle

### "Raw" notes on check objects lifecycle

Checks can be triggered in the following manner:

- A draft is submitted to community (i.e. a draft review)
- A published record is submitted to a community
  - This workflow has a slight issue… the checks will run on the record at its current revision, and if they yield any results, the submitter has to then edit the record, which will create a draft of it…
    - One idea would be to “move”/convert the `CheckRun` objects to `is_draft: true` and `revision_id: <draft's revision>` on Edit
    - This enforces a "preference" to have checks on the draft (if there is one)
- A record that is already part of a community is edited, and its draft then has to be of course have checks run, so that it can still comply at least with the required checks of the community.
- A community curator wants to perform some sort of “bulk run” to check if all the records in their community pass the configured checks

At some point check runs on a draft will reach a satisfactory state, both for the submitter and community curators.

- If the submitter reached this point, they will need to notify the curators about it. That could either be via a comment on the inclusion/review request, or via some sort of “request review” event that would notify the community curators.
- Once the community curators review the draft and accept its changes, they will then “Accept” the request and the record will be included in the community as usual.
- In the case where the record is already part of the community, there is now community curator that will review the record though. In that case the “required”/error-severity checks will “block” the record owner from publishing the record.
  - We have to take into account also the scenario where communities have already accepted records in them, that never had any checks run on them (i.e. before the checks feature was enabled in the system)

In all of the above workflows the `CheckRun` rows must be created/modified and possibly deleted accordingly.

- To keep the `CheckRun`s table from growing indefinitely, we want to clean-up some of the checks (e.g. from older revisions)
- It's still interesting to keep the last successful check even after a record is published/accepted in a community. That would allow to display e.g. some sort of “badge”/checkmark that this record passes all the community's checks

### Trigger points

- **Draft Review request**: When a draft submitted to a community, the configured curation checks are triggered to run automatically.
- **Record Inclusion request**: Published records submitted to a community undergo the same checks as drafts. Any detected issues are logged, and feedback is provided to both submitters and curators. Re-editing challenges are managed by ensuring the system tracks changes and runs checks incrementally on modified fields.
- **Record Updates/Edits**: Updates to records in a community trigger checks on the modified fields or affected sections of the record. This ensures that all updates comply with the community's quality standards without reprocessing the entire record.
- (Out of scope of this RFC) **Bulk Runs**: Bulk runs allow curators to apply curation checks to multiple records
  at once. This feature is designed for efficiency, providing aggregated diagnostics and
  actions to streamline resolution. It is particularly useful for retroactive checks or
  batch updates.

### Check lifecycle

> TODO: Expand on the points, and maybe add a diagram
> - **Initial State:** Checks are queued or initiated based on a trigger.
> - **Execution:** Logic for running checks on records, including multiple attempts or retries.
> - **Resolution:**
>   - What constitutes a "satisfactory state."
>   - Submitter notifying curators.
>   - Curators reviewing and accepting changes.

## Service Layer

> TODO: Describe the service layer

## Presentation Layer

### REST API

#### Create a community check config

```http
POST /api/communities/<community_id>/checks HTTP/1.1
Content-Type: application/json

{
  "check_id": "author-pids",
  "params": {"types": ["orcid", "gnd"]},
  "severity": "error",
  "order": 1,
  "enabled": true
}

HTTP/1.1 201 Created

{
  "id": "<check_config_id>",
  "check_id": "author-pids",
  "params": {"types": ["orcid", "gnd"]},
  "severity": "error",
  "order": 1,
  "enabled": true
}
```

#### List community check configs

```http
GET /api/communities/<community_id>/checks HTTP/1.1

HTTP/1.1 200 OK

[
  {
    "id": "<check_config_id>",
    "check_id": "author-pids",
    "params": {"types": ["orcid", "gnd"]},
    "severity": "error",
    "order": 1,
    "enabled": true
  },
  {
    "id": "<check_config_id>",
    "check_id": "file-types",
    "params": {"types": ["tabular", "image"]},
    "severity": "warning",
    "order": 2,
    "enabled": true
  }
]
```

#### Accessing checks on a draft record

```http
GET /api/records/<record_id>/draft/checks HTTP/1.1

HTTP/1.1 200 OK

{
    "hits": [
        {
            "id": "<check_run_id>",
            "config_id": "<check_config_id>",
            "revision_id": 3,
            "name": "Author identifiers",
            "status": "finished",
            "result": {
                "diagnostics": [
                    {
                        "message": "Author 'Nielsen, Lars Holm' has ORCiD identifier.",
                        "severity": "info",
                        "action": {}
                    },
                    {
                        "message": "Author 'Ioannidis, Alex' has an ORCiD identifier.",
                        "severity": "info",
                        "action": {}
                    }
                ],
            },
            "links": {
                "self": "/api/records/<record_id>/draft/checks/<check_run_id>",
            }
        },
        {
            "id": "<check_run_id>",
            "config_id": "<check_config_id>",
            "revision_id": 3,
            "name": "File types",
            "status": "failed",
            "result": {
                "diagnostics": [
                    {
                        "message": "File 'data.xlsx' is a proprietary file type. Use '.csv' instead.",
                        "severity": "error",
                        "action": {
                            "message": "Replace file 'data.xlsx' with 'data.csv'.",
                        }
                    }
                ],
            },
            "links": {
                "self": "/api/records/<record_id>/draft/checks/<check_run_id>",
            }
        }
    ]
}
```

#### To get checks processed for a specific community (Important for UI)

```http
GET /api/records/<record_id>/draft/<community_id>/checks HTTP/1.1

HTTP/1.1 200 OK

{
    "hits": [
        {
            "id": "<check_run_id>",
            "config_id": "<check_config_id>",
            "revision_id": 3,
            "name": "Author identifiers",
            "status": "finished",
            "result": {
                "diagnostics": [
                    {
                        "message": "Author 'Nielsen, Lars Holm' has ORCiD identifier.",
                        "severity": "info",
                        "action": {}
                    },
                    {
                        "message": "Author 'Ioannidis, Alex' has an ORCiD identifier.",
                        "severity": "info",
                        "action": {}
                    }
                ],
            },
            "links": {
                "self": "/api/records/<record_id>/draft/<community_id>/checks/<check_run_id>",
            }
        },
        {
            "id": "<check_run_id>",
            "config_id": "<check_config_id>",
            "revision_id": 3,
            "name": "File types",
            "status": "failed",
            "result": {
                "diagnostics": [
                    {
                        "message": "File 'data.xlsx' is a proprietary file type. Use '.csv' instead.",
                        "severity": "error",
                        "action": {
                            "message": "Replace file 'data.xlsx' with 'data.csv'.",
                        }
                    }
                ],
            },
            "links": {
                "self": "/api/records/<record_id>/draft/<community_id>/checks/<check_run_id>",
            }
        }
    ]
}
```


### UI Mockups

**Draft review submission flow**

> [name=Karolina Przerwa] Are we planning to run checks also on records which are not inside a community? To encourage filling metadata
> [name=Alex] I see two variations of this:
> - "Global checks" applied on an instance-level, not associated with a community at all. This is similar to the discussions we had about "global/top-level collections"
> -

> [name=Karolina Przerwa] All the mockups seem to have checks under "Conversation tab" is this intentional? Why do we need "Checks" tab? I imagined in the conversation we would have something like GH, "All check have passed" - overview of the status, rather than listing every check
> [name=Alex] Correct, the box in the "Conversation" tab is meant to be a quick overview of the checks (similar to the GitHub PR box).
> There's a missing mockup from the RFC for the "Checks" tab, that basically has much more details about each check (see point `2a` below).

1. After selecting a community, filling in metadata and submit draft for review:

![](https://codimd.web.cern.ch/uploads/upload_d71151f986248f956c9b1394421d7a8e.png)

2. After checks have finished running

![](https://codimd.web.cern.ch/uploads/upload_42870ed90f67da065f9b9c937f6a2ced.png)

2a. Checks tab shows details for each check run

> TODO: See what kind of information we need to display and how...
>       It's much more efficient/useful on the deposit form directly

3. Going back to the deposit form to see check results

![](https://codimd.web.cern.ch/uploads/upload_ade817fe6d0c93c4237c583d6397ef82.png)

4. After editing fields that were affected by checks (but not clicking "Save draft" yet):

![](https://codimd.web.cern.ch/uploads/upload_9965f96584798c972c9f36547170b69a.png)

5. After clicking "Save draft" (note the "Checks" sidebar box):

> [name=Karolina Przerwa] Checks are more important than "Delete", should be placed above the button
> [name=Alex] Agree :+1:, has been fixed in the latest mockups

![](https://codimd.web.cern.ch/uploads/upload_2abeb5743421c1d09b257eeb07bdff3a.png)

6. All required checks passing:

![](https://codimd.web.cern.ch/uploads/upload_0afadf058a48e5e2ec6d1054a182a3b6.png)


## Deposit form changes

### Errors display

- Replace the top banner list of errors with a "summary" with anchor links to the sections where there are errors/warnings
    - We need a mapping/listing of what field paths are under which section, e.g.:
        - Basic info: `pids.*`, `metadata.creators`, `metadata.title`, `metadata.resource_type`, etc.
- Each section has a summary of its errors/warnings
- Each field has to be able to display errors of different "severity" (fail, warn, info)
    - This requires expanding the schema of the `errors` field of draft REST API responses to add a `severity` field
- Displayed errors that are coming from community checks, should have contextual info that links to the "Checks" tab
    - This requires expanding the schema of the `errors` field of draft REST API responses... `ui_context_icon` vs `community`

```json
{
   "errors": [
       {
           "field": "metadata.funding",
           "messages": ["At least one EC-funded award should be present."],
           "severity": "info",  // failure, warning, info
           "context": {
               "community": "<community_id>",
               "check": "<check_id>",
           }
       },
       {
           "field": "metadata.creators",
           "messages": ["Authors must use persistent identifiers (ORCiD, GND, etc.). 'Nielse, Lars Holm' and 'Ioannidis, Alex' are missing ORCiDs."],
           "severity": "warning",  // failure, warning, info
           "context": {
               "community": "<community_id>",
               "check": "<check_id>",
           }
       }
   ],

   "ui": {
      "communities": {
          "<community_id>": {...slug, title, etc...},
      },
      "checks": {
          "<check_id>": {...title, links, description, etc...},
      }
   }
}
```

**Configuring checks as a community manager/curator**

> TODO: Add mockups

**Display of checks on record landing page**

> TODO: See if necessary, and add mockups

# Example

## Check - Author identifiers

A check that checks if authors have persistent identifiers of a given type.

```python
class AuthorPIDs(Check):

    id = "author-pids"
    name = "Author identifiers"
    description = "Authors must have persistent identifiers"

    params = {
        "types": fields.List(
          fields.Str(validate=validate.OneOf(["orcid", "gnd"]))
        )
    }

    def run(self, record: Record):
        diagnostics = []

        creators = record["metadata"]["creators"]
        for creator_idx, creator in enumerate(creators):
            identifiers = creator.get("person_or_org", {}).get("identifiers", [])
            for id_idx, identifier in enumerate(identifiers):
                scheme = identifier.get("scheme")
                if scheme not in self.params["types"]:
                    name = creator["person_or_org"]["name"]
                    diagnostics.append({
                        "message": f"Author '{name}' has no identifier.",
                        "severity": "error",
                        "path": [f"metadata.{creator_idx}.person_or_org.identifiers.{id_idx}"],
                        "values": [{
                            "path": f"metadata.{creator_idx}.person_or_org.identifiers.{id_idx}.scheme",
                            "value": scheme
                        }]
                    })

        self.check.result = {"diagnostics": diagnostics, "actions": []}
        self.check.status = "done"
```

## Check - FAIR Tool score

A check that checks the external FAIR Tool score of the record.

```python
class FAIRToolScore(Check):

    id = "fair-tool-score"
    name = "FAIR Tool score"
    description = "Check the external FAIR Tool score of the record"

    params = {
        "threshold": fields.Float(validate=validate.Range(min=0, max=1))
    }

    # This check is external, so will be executed in a Celery task
    sync = False

    def run(self, record: Record):
        resp = self.fair_tool_client.evaluate(record)
        self.check.state = {"request_id": resp["id"]}

    can_refresh = True
    def refresh(self):
        diagnostics = []

        result = self.fair_tool_client.refresh(self.check.state["request_id"])
        if result["status"] == "done":
            diagnostics = result[...]
        return {"diagnostics": diagnostics, "actions": []}
```

## Workflow

A concrete example of a record being submitted to a community that has configured four *Check Configs*:

- Checks in registry
  - `AuthorPIDs` - Authors must have persistent identifiers of type XYZ
    - Parameters: `types: List[str]`
  - `FundingInfo` - Require award from funder XYZ
    - Parameters: `funder: str`
  - `LicenseType` - Require license of type XYZ
    - Parameters: `types: List[str]`
  - `FileTypes` - Non-proprietary file types should be used
    - Parameters: `types: List[str]`
  - `FAIRToolScore` - Check the external FAIR Tool score of the record
    - Parameters: `threshold: float`
- Community
  - Check Config A: `AuthorPIDs(params: ["orcid", "gnd"], severity: "error")`
  - Check Config B: `FundingInfo(params: ["ec"], severity: "error")`
  - Check Config C: `FileTypes(params: ["tabular", "image"], severity: "warning")`
  - Check Config D: `FAIRToolScore(params: 0.80, severity: "warning")`
- Record (submitted to Community)
  - Check A
    - Run `A(rev: 1, status: success,  result: {...})`
  - Check B
    - Run `B(rev: 1, status: failed,  result: {...})`
    - Run `B(rev: 4, status: failed,  result: {...})`
  - Check C
    - Run `C(rev: 1, status: failed,  result: {...})`
    - Run `C(rev: 4, status: failed,  result: {...})`
  - Check D
    - Run `D(rev: 1, status: failed,  result: {...})`
    - Run `C(rev: 4, status: failed,  result: {...})`


# How we teach this

> What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing Invenio patterns, or as a wholly new one?

> Would the acceptance of this proposal mean the Invenio documentation must be re-organized or altered? Does it change how Invenio is taught to new users at any level?

> How should this feature be introduced and taught to existing Invenio users?

# Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Invenio, on the integration of this feature with other existing and planned features, on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

# Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## `CheckRun` reproducibility

For an "active" check run, we are currently storing the `config_id` FK to the `CheckConfig` table and the record's `revision_id`. This is a problem if the community changes the configuration of the check after the check has run. That could be while the record is still in draft, or even after it has been published.

There are a few ways to address this:

- **Store a snapshot of the configuration in the `CheckRun`**
  - ✅ This would allow us to keep the original configuration of the check, even if the community changes it later.
  - ❌ We would have to store a lot of redundant data, as the `CheckConfig` is already stored in the database once.
  - ❌ We would have to keep track of the "snapshot" and update it if the community changes the configuration.
- **"Version" the `CheckConfig` with a `revision_id` and store that in the `CheckRun`**
  - ✅ This would be a more lightweight solution, as we would only store the `revision_id` of the `CheckConfig`, and not the whole configuration.
  - ✅ When a community changes a `CheckConfig` we can easily query for `CheckRun`s that were run with an outdated `CheckConfig` (and possibly re-run them).
  - ❌ We won't have the exact "snapshot" of the configuration for older `CheckRun`s (i.e. a reproducibility issue, *"With what config did this check result come from?"*).

For now we will not address this issue in this RFC, since it's a more complex problem that requires more thought and UX design. At the moment, if a comunity changes a `CheckConfig`, only new `CheckRun`s will be affected by the change.

## Checks for non-record entities

Currently, the checks are designed to run on records/drafts. However, there are other entities in InvenioRDM that could benefit from checks, such as communities, users, etc. Since that would be a relatively new concept without any prior use-cases, we decided to keep the scope of this RFC to record checks only.

In general, extending the existing model to support "polymorphic" behavior over different entities would be possible, but it would require a more complex datamodel and API design.

# Unresolved questions

### Lars' comments Jan 27

There's a one central part missing I think, and related to this mockup:

![](https://codimd.web.cern.ch/uploads/upload_5715877f40370c53e660040c6a0a01df.png)

In my mind we have in this interface

1) 3 checks - metadata, required approvals, file format
2) The *metadata check* is a generic entitiy, that's applied to community through the *check config* (this is also what you say).
3) *Check run* is the result of the application of the *metadata check config* to the given record (also what you say)

> My intent behind the `Rule` vs. `Check` naming, was to differentiate between something that's a "template"/"building-block" vs. something that is stored in the database.
> Given the confusion, I will change to `Check`, `CheckConfig`, `CheckRun` (for clarity, I've renamed `Rule` to `Check` and will refer to it as such below)
> [name=Alex]

The community will have defined a set of metadata **RULES** (this is not the same *rule* as you have). Each rule (e.g. "Scientific articles must provide journal information") will be checked during the "metadata check run"

Here's how the rule could be created:

![](https://codimd.web.cern.ch/uploads/upload_459269db503df819e5e7288529c04463.png)

Overall I think the conceptual model you have proposed will work, but you have checks of e.g. funding info/author each as a separate check, where I see that as a combined metadata check, that checks against the metadata rules.

Either way, we'll need some sort of easy way to create, store and apply metadata rules that does not involved deploying code.

> Agree on building towards the more flexible "Metadata check" UI; we'll need to have a good datamodel for it though, which I felt grew the scope quite a lot.
>
> Conceptually, I saw that defining `Check`s as code, would allow to gradually accomodate:
> 1. basic "hardcoded" type of checks (like "Author identifiers", "Funding info", etc.)
> 2. complex checks dependent on related entities or data processing, e.g. "File formats", "Required approvals", ...
> 3. external checks, like 3rd-party FAIR Score evaluation
> 4. more modular/structured user-defined cases, like the "Metadata check" component. This would be the "advanced" UI, that would transparently replace "1. basic hardcoded checks" from above as its generalization, since it would be trivial to model them under it.
>
> I could see already starting with modeling the "Metadata check" as a `Check` whose parameters are basically a container for smaller rules like "Journal information", "Required open access", "Author identifiers", ...
> [name=Alex]

### Call with Lars/Nico/Alex - Jan 29

- (Lars) Test concurrency (e.g. uploading files, editing the form at the same time)
- How to reconcile with form display/customization config vs. checks
    - What if we have multiple communities? Should requests
- (Lars) for the future we'll need possibly "auto-accept" after all checks are passing
    - Possibly via "events"? We have to be aware of sync vs. async checks?
- (Lars) What about storing the results of checks (e.g. FAIR score), so that it can be displayed on the record?
    - Should it go in a separate table after the publish
- Conflicting checks between two communities.
- Notifying vs. enforcing/auto-filling information that checks require (e.g. specific grant)

### Nico's comments Jan 29

- we can basically use such checks to enforce some mandatory fields per community, without the need of making this customizable at community level. This will be extremely useful for CDS
- it might be nice (or not!) to allow community managers to have access to the global registry of checks, and re-use/clone the check in the community without having to re-define them... like a catalog to pick from. (if that, we might be missing another DB table)
- `CheckRun`: the `config_id` param is indeed tricky as you have highlighted it. It might be handy if you need to tune the rules on the fly, but at the same time, each run might not be idempotent.
- `/api/records/<record_id>/draft/checks` this endpoint can be tricky in case that the record is in multiple communities... or will it contain the sum of all checks from all communities? In Zenodo, if I remember correctly, you have some records included in lot of communities?
- about the last comment from Lars on the data model: I am wondering if it is simply a community manager choice to define 1, 2 or 3 different distinct rules for metadata checks, or to define 1 rule with multiple fields checks inside (single vs combined).
- the "require approvals" check would be very powerful for CDS, but it might be tricky... this is the work that Zac started now, to understand what is missing in CDS/InvenioRDM to be able to bulk migrate many collaborations that have reviews workflows. More on this in the coming weeks. It feels like not covered in the RFC for the moment, right?

### Jose's comments Jan 29

- I think we need to be carefully not to have 2 ways of dealing with mandatory fields. Otherwise it can be weird that, I click submit, I get an error in the form, I correct it, I click submit...it goes through, but then i get a message from the curator that I didn't add the award information...if i were the user i would say "why the form allowed me to go throuhg?"

### Karolina's comments 6th of Feb

I've submitted them in between the text

# Resources/Timeline

Zenodo team: 18 PMs over 6 weeks
