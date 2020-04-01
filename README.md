# Invenio RFCs

The Invenio RFC process (Request For Comments) is a **communication tool** with
the purpose to:

- coordinate the design process
- document design decisions
- produce consensus among Invenio stakeholders

The RFCs are not meant to be a heavy long process, but rather an agile process
to aid communication between geographically dispersed teams as well as to
document Invenio development so that we can avoid knowledge loss when people
leave and ease knowledge transfer when people joins.

Invenio RFCs is not an official approval process which you might known from
other RFC processes.

### TL;DR

- [Request a new RFC](https://github.com/inveniosoftware/rfcs/issues/new/choose)

#### Quick links

- [Pending RFC requests](https://github.com/inveniosoftware/rfcs/labels/Proposal%3A%20Pending)
- [Work-In-Progress RFCs](https://github.com/inveniosoftware/rfcs/labels/Proposal%3A%20Accepted)
- [Completed RFCs](https://github.com/inveniosoftware/rfcs/tree/master/rfcs)

### Process overview

1. **Request RFC (focus on scope):** Before starting to write a new RFC, you first have to request the approval of Invenio architects by [opening an issue](https://github.com/inveniosoftware/rfcs/issues/new/choose). This is to aid scoping the RFC, avoid duplication as well as same everyone time..
2. **Write RFC (focus on content):** If the request is accepted by architects, you start collaborative writing of the RFC using the [template](https://github.com/inveniosoftware/rfcs/blob/master/0000-template.md) and CodiMD. This part of the process focuses on the *content*, and an architect is assigned to support you in writing the RFC. The goal is that most discussions and alignment on a solutions happens in the writing phase.
3. **Review RFC (focus on quality):** Once the RFC is complete, you submit a pull-request with the new RFC for final review. The review focuses on quality of RFC, not the content of the RFC (should already have been agreed upon in the writing phase).
4. **Merge RFC:** RFC is merged into repository (RFCs does not need to be complete to be merged, as long as unresolved questions have been listed in the RFC and quality has passed the review).

You'll notice that there's no explicit approval step for the acceptance of an RFC. This is because the goal is to a) document and b) produce consensus on designs. The [Invenio Governance](https://inveniosoftware.org/governance/) applies for the decision making and handling of disagreements.

### When to write a RFC?

You need to write a RFC to make substantial changes to Invenio. A substantial
change could be:

- Adding/removing larger features and/or modules.
- Changing existing features/APIs.
- Changes of design patterns, idiomatic usage or conventions.

You do not need a RFC for:

- Modules/features you develop privately (you only need it, if you want it as
  an official Invenio module).

If in doubt, just ask on [Gitter](http://gitter.im/inveniosoftware/invenio).

### Step 1: Request an RFC

- [Open an issue](https://github.com/inveniosoftware/rfcs/issues/new/choose).
  - Document the 1) Motivation 2) Summary of proposed changes and 3) Resources

##### For architects:

- Label and assign the issue
  - All new RFCs should have the "Proposal: Pending" label and a label for the product (e.g. "RDM").
- Review the request
- **If rejected**:
  - Add a justification to the comments of the issue.
  - Change the label to "Proposal: Rejected".
  - Close the issue.
- **If accepted**
  - Change the label to "Proposal: Accepted".
  - Create a [new CodiMD](https://codimd.web.cern.ch/new) filling it with the [RFC template](0000-template.md)
  - Add link to CodiMD from issue (``- [RFC work-in-progress](...)``])
  - Assign the point-of-contact architect

### Step 2: Write the RFC

Following is optional. It is just advices for writing the RFC in an collaborative and efficient manner:

- Choose an editor (person) being responsible for this phase (e.g. the architect or the person opening the RFC request)
- **Brainstorming phase:**
  - Fill the template with unstructured bullet points and high-level outline.
  - Try to clearly define scope - what's included, and what's excluded.
  - Try to identify multiple options for solutions.
  - What issues should be addressed?
  - Identify possible stakeholders, and include them in the discussions.
- **Reading phase (moderated by editor):**
  - Add a "Questions" subsection to each section.
  - Read the RFC and add questions/comments to the questions sections (prefix each question with ``<name>: ...``). Be clear purpose and support it with examples.
  - Purpose is to identify sections that needs further discussion.
- **Discussion phase (moderated by editor):**
  - Expect discussion on semantics, naming and scope to possibly be long discussions (i.e. take these discussions first, subsequent discussions will be much faster).
  - Identify discussions points and list them in the document
  - Meet live to discuss discussion points
    - Moderator takes notes as bullet points for each discussion point/question.
    - Moderator must ensure everybody is explicitly asked about their opinion.
    - Conclusion:
      - Ask for preferred solution: Once sufficient discussion has taken place, the moderator asks each person
        for their preferred solution. Goal is to identify if there is consensus or disagreement.
      - Propose conclusion: The moderator looks for a consensus solution an proposes this solution.
      - Ask explicitly everyone if they agree
      - If consensus is not possible, the conclusion can be TBD and perhaps needs more research, and/or ask for input from non-designated architects.
  - Meet live to discuss all questions
    - Use same procedure as for discussion points.
- **Cleaning phase:**
  - Clean up the RFC document - it should be readable and coherent for a third-party which was not part of the discussions.
  - Write up the summary focusing on explaining a third-party about the gist of the RFC.
- **Reviewing phase:**
  - Ask for input from the non-designated architects and other stakeholders.

You can jump around between phases.

**Disagreement resolution**

> Fight for what you believe, but gracefully accept defeat.

We expect everyone (no matter their role and experience level) to adhere to our [Code of Conduct](https://inveniosoftware.org/governance/) and be open, considerate and respectful during discussions. Search for consensus and be ready to change your mind!

Please do your outmost to not have unresolvable disagreements! The more senior your are, the more responsible you are to not have unresolvable disagreements.

In case all attempts to reach consensus have failed, and really only as a very very last resort, the architects can resolve the conflict by taking a decision. This decision should be properly documented, and can be escalated according to the [Invenio Governance](https://inveniosoftware.org/governance/).

### Step 3: Review the RFC

#### How to submit a pull request with a RFC?

This is used in the step 3 (see "Process overview") to submit the already written RFC.

- Fork this repository.
- Copy ``0000-template.md`` to ``rfcs/<product>-0000-<my-title>.md``.
- Fill in the RFC.
- Submit a pull request
- Update your pull request:
    - Rename your ``<product>-0000-<my-title>.md`` to ``<product>-<pull request id with leading zeros>-<my-title>.md`` (e.g. ``rdm-0012-persistent-identifiers.md``).
    - Add a link to the pull request inside the RFC.

#### Reviewing checklist

- Comment on quality of the RFC, not the chosen solutions (this was already done in step 2)!
- Can the RFC be understood by an experienced third-party that didn't participate in the discussions?
- Is the RFC coherent?
- Is the unresolved questions properly documented?
- Is there sufficient resources to implement the RFC?

### Step 4: Merge the RFC

As soon as the RFC has reached sufficient quality level and consensus it can be merged into this RFC repository. The RFC does not need to be fully completed to be merged, as long as unresolved questions have been listed in the RFC.

### RFC States

- **Draft**: The RFC has reached sufficient quality to be merged, some discussions has happened, but there's open questions
and it's not ready yet to be implemented.
- **Ready**: The design is ready, enough discussions has happened to reach a reasonable consensus and quality of RFC is good.
- **Implemented**: The RFC has been implemented in the community or code.

---

The Invenio RFC process owes it's initial inspiration to the
[Ember RFC process](https://github.com/emberjs/rfcs).
