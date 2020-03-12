- Start Date: YYYY-MM-DD
- RFC PR: [#<PR>](https://github.com/inveniosoftware/rfcs/pull/<PR>)
- Authors: Lars Holm Nielsen, Nicola Tarocco, Alex Ioannidis, Guillaume Viger

# Data flow

## Summary

This is just a document to talk about data flow. I really don't know if an RFC is the
ideal format, but I just needed a place to start putting some of the discussion down!

## Motivation

- As a developer, I want to know where to place data flow code and what to expect as input and output,
  so my code plays nice with others, is clearer and there are less bugs.

## Detailed design

Perhaps a pedantic view of the data flow, but it's my representation of
what I perceive the situation to be.

### Data IN
Problem out to solve: where to place code and what form of record are you manipulating

**Converts1**
from submitted xml / json / supported format (external world) -> loaded_json_dict

Note: here Python does boolean conversion, but *not* date(time) conversions

```
|   |   |
V   V   V
```

**Validates**
loaded_json_dict -> validated_json_dict

Done via marshmallow schema
:warning: marshmallow schema CAN do conversion (it **is** one of its purposes)
and mutation: it can convert Date-string to Date(time) object or remove loaded fields or modify them.

Right now, I am leaning towards not, because there is a "conversion" and "mutation" step down
the line: it separates roles cleanly to have this layer purely validate.

```
|   |   |
V   V   V
```

**Converts2**
validated_json_dict -> converted_json_dict

Case of interest: date(time) strings into date(time) objects; is this the only case?

Having a date object is convenient in order to conform its serialization/display afterwards.
BUT maybe it can be achieved on the fly and is not needed internally.
If that's the case and datestrings are the only need for conversion, this layer could be discarded.


```
|   |   |
V   V   V
```

**Mutates for storage**

To DB *and then* to ES
   DataCite to DB is essentially this whole flow again
   DB to DataCite is part of the Data OUT flow

*DB/jsonschema*
converted_json_dict + context -> db_ready_json_dict

Use of the context here to generalize outside of request flow.
Addition of PID / files and so on to the converted_json_dict or the DB entry
If some fields need additional utility fields, this layer would be the place to put them.

If there was a Date(time) object it would have to be serialized back (other strike against their conversion...)

```
|   |   |
V   V   V
```

*Elasticsearch*
no guaranteed ordering on the indexing hooks (indexing_hook number represents ordering that just so happened)

db_ready_json_dict -> indexing_hook1 -> es_ready_json_dict_up_to_1
es_ready_json_dict_up_to_1 -> indexing_hook2 -> es_ready_json_dict_up_to_2
                    ...
loaded_json_dict_up_to_N-1 -> indexing_hookN -> es_ready_json_dict_up_to_N

*DataCite (other external systems)*

### Data OUT
(This part is especially hazy to me. You can read most of these steps with question marks :) )

Problem out to solve: duplication/uniformity of data access:
  API served from ES, jinja templates served from DB but want uniformity
  DataCite served from DB but with different serialization than API or jinja2

**External Serialization**
db_retrieved_json_dict -> DataCite_xml

db_retrieved_json_dict should be equivalent to db_ready_json_dict

**Internal serialization (API/jinja)**

parameters + context (DB) -> db_retrieved_json_dict -> uniformity process -> jinja2
parameters + context (ES) -> es_retrieved_json_dict -> uniformity process -> API

es_retrieved_json_dict should be equivalent to es_ready_json_dict_up_to_N
uniformity process is at the individual record level
uniformity process is probably best served as the dump of the marshmallow schema
to keep input and output consistent, but then it might be in charge of mutating
the output which would blur lines again...

### Conclusions

resource controller we talked about would deal with the **validates** layer all the way to
the part of the **internal serialization** that deals with retrieval.
That last uniformity process is part coalescing / part serialization...

I am pretty sure this is another rehash of what we talked about, but it's good to have multiple takes on these to hammer out details or other perspectives :sweat_smile:

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Invenio patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Invenio documentation must be
re-organized or altered? Does it change how Invenio is taught to new users
at any level?

> How should this feature be introduced and taught to existing Invenio
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Invenio,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

Most of the above ;)