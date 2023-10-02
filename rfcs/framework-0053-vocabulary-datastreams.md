- Start Date: 2021-10-14
- RFC PR: [#53](https://github.com/inveniosoftware/rfcs/pull/53)
- Authors: Pablo Panero, Alex Ioannidis, Dimitris Frangiadakis
- State: DRAFT

# Vocabulary datastreams

## Summary

Vocabularies are a specialized type of records used to represent controlled sets of values or "_vocabularies_". These come from many different _sources_ and in many different formats (XML, JSON, specific raw data, etc.). In addition, they are vast (up to tens of millions). Therefore, we need flexible and configurable method to "automate" their processing (ingestion, parsing and dumping).

_Vocabulary datastreams_ enable the instance administrator to configure different ways of reading (ingest), transforming (parsing) and dumping (writing) vocabularies, or even create custom ones.

## Motivation

### Load/Update
    
- As an administrator I want to **load/update a vocabulary from a local file**.
- As an administrator I want to **load/update a vocabulary from a remote file**.
- As an administrator I want to **load/update only a specific set of entries from a vocabulary** (e.g. load only CERN ORCiD names).
- As a administrator I want to be able to **transparently load generic and custom (contrib) vocabularies**.
- As an administrator I want to **load/update nested vocabularies**
- As an administrator I want to **load/update using bulk creation/indexing**
    
### Delete
    
- As an administrator I want to **delete a specific entry or set of entries** from a vocabulary
- As an administrator I want to **delete a vocabulary**
- As an administrator I want to **delete all vocabularies**
    
### Convert

- As an administrator I want to **convert vocabularies to a (common) file format to store them** (and potentially decrease file size).

### List
    
- As an administrator I want to be able to **list all imported vocabularies**
- As an administrator I want to be able to **search for a vocabulary**. Do we forsee the list of vocabularies containing more than 50-100 entries?
    
### Dump

- As an administrator I want to be able to **dump a vocabulary** (or many).

### Developer
  
- As a developer I want to **implement an new data source** (reader)
- As a developer I want to **implement a new way of parsing or (pre)processing a data entry before it is used** (transformer)
- As a developer I want to **implement a new way to capture the result of a data importing pipeline** (writer).
    
... and be able to plug it in the normal pipeline (e.g. ORCiD from tar file and ORCiD from REST API).

    
## Detailed design

There are three main objects:

- A **reader**, which will get the information and yield items one by one
- One or more **transformers**, which will apply one or more operations to the received entry.
- One or more **writers**, which will dump the entries to the target

These three objects are glued together in a datastream, which will:

- Iterate through all the entries the reader yields
- Apply all the transformations (one by one, in order of definition) to an entry
- Write the entry using all the writers
    
### Readers
    
As mentioned above, a reader would iterate through the entries available on the origin and will *yield* them one by one.
    
A reader has only one required parameter, `origin`, which represents the entry point of the source (e.g. a filepath, an API url, stdin).

```python
class BaseReader:

    def __init__(self, origin, *args, **kwargs):

    def read(self, *args, **kwargs):
```

Some examples of readers could be a `YamlReader`, `JSONLDreader`, `OrcidHTTPReader`.
    
```python
class YamlReader(BaseReader):

    def read(self):
        with open(self._origin) as f:
            data = yaml.safe_load(f) or []
            for entry in data:
                yield StreamEntry(...)
```
    
### Transformers

A transformer is the way to modify the entries once they have been read. There is almost no limits to what kind of transformations can be applied, but the end result will always be a dictionary that ressembles a vocabulary record of some type. Note that the return value will be the passed `stream_entry` with a modified `entry` property. Raises `TransformerError` in case of errors.
    
```python
class BaseTransformer:

    def apply(self, stream_entry, *args, **kwargs):
        stream_entry.entry = self._do_stuff()
        return stream_entry
```

An example transformer could be `OrcidXMLTransformer`, which would transform a full ORCiD XML record into a dictionary compatible with the `names` vocabulary.
    
```python
class OrcidXMLTransformer(XMLTransformer):

    def apply(self, stream_entry, **kwargs):
        transform_from_xml(stream_entry.entry)
        ...
        return stream_entry
```
    
### Writers
    
A writer is in charge of dumping an entry, already parsed and formatted into a target. Returns a _new_ `StreamEntry` to avoid having conflicts with consecutive writers. Raises `WriterException` in case of errors.
    
```python
class BaseWriter:

    def write(self, stream_entry, *args, **kwargs):
        return StreamEntry(result)
```

An example of a writer could be a `YamlWriter` to dump a yaml file, or a `ServiceWriter` to write entries to an Invenio instance.

```python
class ServiceWriter(BaseWriter):

    def __init__(self, service_or_name, identity, *args, **kwargs):

    def write(self, stream_entry, *args, **kwargs):
        """Writes the input entry using a given service."""
        try:
            result = StreamEntry(
                self._service.create(self._identity, stream_entry.entry)
            )
        except ValidationError as err:
            raise WriterError([{"ValidationError": err.messages}])
```

### Filtering
    
Filtering can be applied at any of the previous three levels, when using custom objects.
    
In the case of custom readers, it would suffice with not yielding a specific entry. For example:

```python
class FilteredReader(BaseReader):

    def read(self):
        if condition_ok:
            yield StreamEntry(...)
```

In the case of transformers, since they are executed in order one after the other, the flow needs to be stopped so subsequent transformers are not executed and the entry is dropped. For example:
 
```python
class FilteredTransformer(BaseTransformer):

    def apply(self, stream_entry, *args, **kwargs):
        if not condition_ok:
            raise TransformerError(...)
```

Finally, for writers it would suffice with not writing (reserving raising errors to notify of actual errors). It would be done in a similar fashion than readers:

```python
class FilteredWriter(BaseWriter):

    def write(self, stream_entry, *args, **kwargs):
        if condition_ok:
            write()
```

### Datastream

A datastream will loop through the read entries (`process`), and one by one apply the transformers (`transform`). Once transformed, if the entry is not filtered (_ignored/skipped_) it will write it to all writers.

The `transformed` operation will break the processing of an entry if an error is found (i.e. this entry will not reach the writers). On the other hand, an error in one writer will not block an entry from being written somewhere else.

For example, if we imagine a pipeline of four transformers, failing on the second means that the third might not get the assumed input. On the other hand, failing to write to an invenio instance is independent from writing to a jsonld file (assuming those two writers are the configured ones).

In addition, no compatibility checks are performed and it is left to the administrator to configure readers, transformers and writers in a way that make sense (e.g. do not apply the `ORCiDXMLTransformer` if the reader does not yield XML records).

```python
class BaseDataStream:

    def __init__(self, reader, writers, transformers=None, *args, **kwargs):
        ...
    def filter(self, entry, *args, **kwargs):
        ...
    def process(self, *args, **kwargs):
        ...
    def transform(self, entry, *args, **kwargs):
        ...
    def write(self, entry, *args, **kwargs):
        ...
    def total(self, *args, **kwargs):
        # Not implemented yet (decide default val when not available)
        # Who keeps track of index (datastream or reader)
        ...
```

In order to have a consistent way to pass entries from one method to the other, and track errors, `StreamEntry` object is used. This object is used internally by the datastream.


```python

class StreamEntry:
    """Entry object for streams processing."""

    def __init__(self, entry, errors=None):
        """Constructor."""
        self.entry = entry
        self.errors = errors or []
```
    
### Fixtures

NOTE: This should not go to the RFC, this is to show that, although implementation details vary slightly, vocabularies can be loaded in the same fashion as `invenio-rdm-records` does it today.

A fixture is another layer of encapsulation on top of datastreams. In this case is used to ease the data importing process in new instances.

    
```python
class BaseFixture:

    def _load_vocabulary(self, config, delay=True, **kwargs):
        # create and loop over a datastream

    def _create_vocabulary(self, id_, pid_type, *args, **kwargs):

    def load(self, *args, **kwargs):
        for vocabulary in ...:
            # create vocabulary
            # load vocabulary

```

## Example
 
### CLI usage

(_Currently implemented_)
  
#### Import (would be similar/the same for updating)
    
Contrib or exisiting vocabularies have a default configuration for importing (e.g. for the names is `TarReader`, `OrcidXMLTransformer` and `ServiceWriter`). However, a full configuration file can be passed using the `-f` parameter. Otherwise, only the (tar) file origin would be needed.
    
``` bash
invenio vocabularies import -v names -f ./app_data/vocabularies-future.yaml
    
invenio vocabularies import -v names -0 ./app_data/orcid-dump.tar.gz    
```
    
#### Convert
    
``` bash
invenio vocabularies convert -v names -o "/path/to/orcid_dump.tar.gz" -t names.yaml
```
    
#### Other
   
**Delete**
    
Deleting a single vocabulary item is already possible (implemented), but in the vast majority of cases what we are looking for is to remove the whole voabulary. For that a "delete_all" method needs to be implemented and propagated from `invenio-records-resources`.
    
```
invenio vocabularies delete -v names
invenio vocabularies delete -v names 0000-0000-0000-0000
```
   
**Search/list**
    
```
invenio vocabularies list
invenio vocabularies search vocab_to_search_for
```
    
**Export**

Under the hood it would be implemented as the _convert_ command, using a `DBReader` (implicit). This command exists to avoid having different way to achieve "export/dump", and having a better developer UX.

```
invenio vocabularies export resourcetype <dst_file_path>
```
    
### REST API usage

```http
POST /api/vocabularies/tasks HTTP/1.1
Accept: application/json

{
    "reader": {
        "type": "tar",
        "args": {
            "regex": ".xml$",
            "source": {"path": "/path/to/orcid-dump-2021.tar"},
            
            # or a FileInstance?
            "source": {"file_id": "f1601293-d717-42ac-b666-5480044a0b92"}
        }
    },
    "transformers": [
        {"type": "orcid-xml"}
    ],
    "writers": [{
        "type": "service",
        "args": {
            "service_or_name": "rdm-names",
            "identity": system_identity,
            "delay": true
        }
    }],
}
```

The operations will be async by default. In the future we can potentially accept a parameter for `async=false` and if there are task persistance it can be monitored (e.g. query for status/cancel/pause).
    
```http
HTTP/1.1 202 Accepted
Content-Type: application/json
```   

## How we teach this

- Terminology: ETL?
- How? It is similar to ETL, having full examples (e.g. _ORCiD names_ implementation, _Funders/Grants_ implementation...)

## Drawbacks

- Need to migrate the old `Fixtures` implementation in `invenio-rdm-records`.
- Unknown use cases that might not fit the implementation (e.g. combination async/sync and complex workflows.)
    
## Alternatives

The old `Fixtures` implementation in `invenio-rdm-records`.

## Unresolved questions

- How do we enforce order/wait for completion (e.g. dependent vocabualaries - names might link affiliations -)?
- How do we implement persistence?
- How do we monitor/log progress?
- Bulk operations (import, update, delete, etc.). Pending on a service level API in `invenio-records-resources`.

## Resources/Timeline

Zenodo team: commited for v8 ([product-rdm#62](https://github.com/inveniosoftware/product-rdm/issues/62))