- Start Date: 2021-03-25
- RFC PR: [#<PR>](https://github.com/inveniosoftware/rfcs/pull/<PR>)
- Authors: Alex Ioannidis, Georgios Lignos, Zachos Zacharodimos
- State: DRAFT

# RDM Communities

## Old resources/discussion

- Community datamodel: https://codimd.web.cern.ch/WhwckpbxSGWTpGgniWNayA
- Initial brainstorming: https://codimd.web.cern.ch/-RvwMK4fRq6eSJjCXYtc_A
- Members REST API endpoints/payload https://codimd.web.cern.ch/hgnIp1tiR3SkXcNbyXwIFg
- Sub-communities REST API: https://codimd.web.cern.ch/vytKIvGwTpihxdNdKPamfg
- GitHub orgs PATCH: https://docs.github.com/en/rest/reference/orgs#update-an-organization

## Summary

We introduce into InvenioRDM the concept of "Communities", in order to facilitate the concept of grouping records for organization and management purposes. Communities are not meant to be just collections of records, but also introduce a layer of collaborative features which can be used to serve more complex curation workflows in a repository.

For the scope of this RFC, we're interested in bootstrapping the `invenio-communities` module to allow CRUD operations on Communities through the programmatic and REST APIs, but also UI pages.

## Motivation

- As a user, I want to be able to have a Community page that represents my institution/project/event/topic, so that I can provide information about its scope, purpose, connection to other systems, etc.
- As a user, I want to be able to search for existing Communities, in order to flexibly discover and filter new institutions/projects/events/topics, so that I can know if one I'm interested in already exists.
- As a Community owner, I want to be able to make changes to my Community's page, in order to correct/add information, so that they can quickly appear publicly for other users to see.
- As a Community owner, I want to be able to completely remove/delete my Community, so that it's not visible anymore publicly, in other to avoid other users finding and seeing it.
- As a Community owner, I want to be able to have a unique identifier/slug for my Community, so that I can reliably store and depend on it on my side, in order to use it in REST APIs.

## Detailed design

The module `invenio-communities` will be directly integrated as a dependency and via configuration in `Invenio-App-RDM`.

### Programmatic API

#### Data layer

Communities have very similar requirements/components to records. They have Metadata (e.g. title, description, etc.), Access options (e.g. visibility/protection), a unique PID, and can be associated with files (e.g. for the community's logo). For these reasons a Community can be easily modeled by extending the existing `invenio_records.models.RecordMetadata` model and `invenio_records.api.Record` API classes, and systemfields like the `PIDField`, the `AccessField`, and the `FilesField`. 

**invenio_communities/communities/records/models.py**

```python
class CommunityMetadata(db.Model, RecordMetadataBase):

    __tablename__ = 'communities_metadata'
		
		# For files support
    bucket_id = db.Column(UUIDType, db.ForeignKey(Bucket.id))
    bucket = db.relationship(Bucket)


class CommunityFileMetadata(db.Model, RecordMetadataBase, FileRecordModelMixin):

    __tablename__ = 'communities_files'

```

**invenio_communities/communities/records/api.py**

```python
class CommunityFile(FileRecord):

    model_cls = models.CommunityFileMetadata
    record_cls = LocalProxy(lambda: Community)


class Community(Record):
 
    pid = PIDField(
        'id', 
        provider=CommunitiesIdProvider, 
        # We manually handle the PID creation
        create=False,
        # On community deletion we want to PID to be automatically destroyed
        delete=True,
    )
    schema = ConstantField('$schema', 'local://communities/communities-v1.0.0.json')

    model_cls = models.CommunityMetadata

    index = IndexField("communities-communities-v1.0.0", search_alias="communities")

    # Custom implementation to model community-specific options 
    access = CommunityAccessField()

    bucket_id = ModelField(dump=False)
    bucket = ModelField(dump=False)
    files = FilesField(
        store=False,
        file_cls=CommunityFile,
        # Don't delete, we'll manage in the service
        delete=False,
    )
```

##### Support for editable PID

A community's unique persistent identifier has to be specified by the user during creation, and can also be changed. Thus the default `RecordIdProvider` which generates the usual "cool PIDs" cannot be used. For that we reason we defined a custom PID provider, which uses the community's (input) metadata to manage the PIDs value. 

To fully implement this functionality we also need an additional `PIDComponent` in the Service layer (see below) responsible for validation and business logic.

#### Service layer

We can re-use many of the services, like the `RecordService`, and `FilesService`. We do not need any of the draft and parent/versioning functionality, since changes to a community are applied immediately and to the same entity. For the `FilesService` we're using a "nested" service (similar to how this is done in the RDM records/drafts services).

In addition to the original service methods, we also implement:

- a custom "search"-like method, for filtering search results down to a specific user;
- a method for renaming the community, since in the future this action might have side-effects (e.g. for reindexing records that belong to the community)
- methods for managing the community's logo file which provide a simplified API for updating a single file (named `"logo"`) via the `FilesService`. For creating/updating the logo, the original 3-step process (initialize, upload content, commit) has been compacted into one single step (since the logo's file size is small).

```python
class CommunityService(RecordService):

    @property
    def files(self):
        """Community files service."""

    def search_user_communities(self, identity, params=None, es_preference=None, **kwargs) -> RecordList:
        """Search for records matching the querystring."""

    def rename(self, id_, identity, data, revision_id=None, raise_errors=True) -> RecordItem:
        """Rename a community."""

    def read_logo(self, id_, identity) -> FileItem:
        """Read the community's logo."""

    def update_logo(self, id_, identity, stream, content_length=None) ->  FileItem:
        """Update the community's logo."""

    def delete_logo(self, id_, identity) -> FileItem:
        """Delete the community's logo."""

```

#### Presentation/resources layer

Again, the existing `RecordsResource` can be extended to handle the basic CRUD operations. We only need to wire-up the new service layer methods to their appropriate routes, trying always to reuse existing helpers/decorators:

```python
    @request_search_args
    @response_handler(many=True)
    def search_user_communities(self):
        """Perform a search over the user's communities."""

    @request_headers
    @request_view_args
    @request_data
    @response_handler()
    def rename(self):
        """Rename a community."""

    @request_view_args
    def read_logo(self):
        """Read logo's content."""

    @request_view_args
    @request_stream
    @response_handler()
    def update_logo(self):
        """Upload logo content."""

    @request_view_args
    def delete_logo(self):
        """Delete logo."""

```

### REST API

#### Create a Community

`POST /api/communities`

**Parameters**

| Name     | Type   | Location | Description                                                  |
| -------- | ------ | -------- | ------------------------------------------------------------ |
| `accept` | string | header   | - `application/json` (default)<br />- `application/vnd.inveniordm.v1+json` |

**Request**

```http
POST /api/communities HTTP/1.1
Content-Type: application/json

{
  "access": {
    "visibility": "public",
    "member_policy": "open",
    "record_policy": "open"
  },
  "id": "my_community_id",
  "metadata": {
    "title": "My Community",
    "description": "This is an example Community.",
    "type": "event",
    "curation_policy": "This is the kind of records we accept.",
    "page": "Information for my community.",
    "website": "https://inveniosoftware.org/",
    "funding": [
      {
        "funder": {
          "name": "European Commission",
          "identifier": "00k4n6c32",
          "scheme": "ror"
        },
        "award": {
          "title": "OpenAIRE",
          "number": "246686",
          "identifier": ".../246686",
          "scheme": "openaire"
        }
      }
    ],
    "organizations": [
      {
        "name": "CERN",
        "identifiers": [
          {
            "identifier": "01ggx4157", 
            "scheme": "ror"
          }
        ]
      }
    ]
  }
}
```

**Response**

```http
HTTP/1.1 201 CREATED
Content-Type: application/json

{   
  "access": {
    "owned_by": [
      {
        "user": <user_id>
      }
    ],
    "record_policy": "open",
    "member_policy": "open",
    "visibility": "public"
  }
  "id": "my_community_id",
  "updated": "2021-04-29T14:24:02.830457+00:00",
  "revision_id": 1,
  "created": "2021-04-29T14:24:02.806782+00:00",
  "links": {
    "self": "{scheme+hostname}/api/communities/my_community_id",
    "self_html": "{scheme+hostname}/communities/my_community_id",
    "settings_html": "{scheme+hostname}/communities/my_community_id/settings",
    "logo": "{scheme+hostname}/api/communities/my_community_id/logo",
    "rename": "{scheme+hostname}/api/communities/my_community_id/rename"
  },
  "metadata": {
    "title": "My Community",
    "type": "event",
    "curation_policy": "This is the kind of records we accept.",
    "description": "This is an example Community.",
    "page": "Information for my community.",
    "website": "https://inveniosoftware.org/",
    "funding": [
      {
        "award": {
          "number": "246686",
          "title": "OpenAIRE",
          "identifier": ".../246686",
          "scheme": "openaire"
        },
        "funder": {
          "identifier": "00k4n6c32",
          "scheme": "ror",
          "name": "European Commission"
        }
      }
    ],
    "organizations": [
      {
        "name": "CERN",
        "identifiers": [
          {
            "identifier": "01ggx4157",
            "scheme": "ror"
          }
        ]
      }
    ]
  }
}
```

#### Update a Community

`PUT /api/communities/<comid>`

**Parameters**

| Name     | Type   | Location | Description                                                  |
| -------- | ------ | -------- | ------------------------------------------------------------ |
| `comid`  | string | path     | Identifier of the community, e.g.  `my_community`            |
| `accept` | string | header   | - `application/json` (default)<br />- `application/vnd.inveniordm.v1+json` |

**Request**

```http
PUT /api/communities/<comid> HTTP/1.1
Content-Type: application/json

{
  "access": {
    "visibility": "public",
    "member_policy": "open",
    "record_policy": "open"
  },
  "id": "my_community_id",
  "metadata": {
    "title": "My Updated Community",
    "description": "This is an example Community.",
    "type": "event",
    "curation_policy": "This is the kind of records we accept.",
    "page": "Information for my community.",
    "funding": [
      {
        "funder": {
          "name": "European Commission",
          "identifier": "00k4n6c32",
          "scheme": "ror"
        },
        "award": {
          "title": "OpenAIRE",
          "number": "246686",
          "identifier": ".../246686",
          "scheme": "openaire"
        }
      }
    ],
    "organizations": [
      {
        "name": "CERN",
        "identifiers": [
          {
            "identifier": "01ggx4157", 
            "scheme": "ror"
          }
        ]
      }
    ]
  }
}
```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{   
  "access": {
    "owned_by": [
      {
        "user": <user_id>
      }
    ],
    "record_policy": "open",
    "member_policy": "open",
    "visibility": "public"
  }
  "id": "my_community_id",
  "updated": "2021-04-29T14:35:53.365430+00:00",
  "revision_id": 2,
  "created": "2021-04-29T14:24:02.806782+00:00",
  "links": {
    "self": "{scheme+hostname}/api/communities/my_community_id",
    "self_html": "{scheme+hostname}/communities/my_community_id",
    "settings_html": "{scheme+hostname}/communities/my_community_id/settings",
    "logo": "{scheme+hostname}/api/communities/my_community_id/logo",
    "rename": "{scheme+hostname}/api/communities/my_community_id/rename"
  },
  "metadata": {
    "title": "My Updated Community",
    "type": "event",
    "curation_policy": "This is the kind of records we accept.",
    "description": "This is an example Community.",
    "page": "Information for my community.",
    "website": "https://inveniosoftware.org/",
    "funding": [
      {
        "award": {
          "number": "246686",
          "title": "OpenAIRE",
          "identifier": ".../246686",
          "scheme": "openaire"
        },
        "funder": {
          "identifier": "00k4n6c32",
          "scheme": "ror",
          "name": "European Commission"
        }
      }
    ],
    "organizations": [
      {
        "name": "CERN",
        "identifiers": [
          {
            "identifier": "01ggx4157",
            "scheme": "ror"
          }
        ]
      }
    ]
  }
}
```

#### Get a Community

`GET /api/communities/<comid>`

 **Parameters**

| Name     | Type   | Location | Description                                                  |
| -------- | ------ | -------- | ------------------------------------------------------------ |
| `comid`  | string | path     | Identifier of the community, e.g.  `my_community`            |
| `accept` | string | header   | - `application/json` (default)<br />- `application/vnd.inveniordm.v1+json` |

**Request**

```http
GET /api/communities/<comid> HTTP/1.1
Accept: application/json
```

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{   
  "access": {
    "owned_by": [
      {
        "user": <user_id>
      }
    ],
    "record_policy": "open",
    "member_policy": "open",
    "visibility": "public"
  }
  "id": "my_community_id",
  "updated": "2021-04-29T14:24:02.830457+00:00",
  "revision_id": 1,
  "created": "2021-04-29T14:24:02.806782+00:00",
  "links": {
    "self": "{scheme+hostname}/api/communities/my_community_id",
    "self_html": "{scheme+hostname}/communities/my_community_id",
    "settings_html": "{scheme+hostname}/communities/my_community_id/settings",
    "logo": "{scheme+hostname}/api/communities/my_community_id/logo",
    "rename": "{scheme+hostname}/api/communities/my_community_id/rename"
  },
  "metadata": {
    "title": "My Community",
    "type": "event",
    "curation_policy": "This is the kind of records we accept.",
    "description": "This is an example Community.",
    "page": "Information for my community.",
    "website": "https://inveniosoftware.org/",
    "funding": [
      {
        "award": {
          "number": "246686",
          "title": "OpenAIRE",
          "identifier": ".../246686",
          "scheme": "openaire"
        },
        "funder": {
          "identifier": "00k4n6c32",
          "scheme": "ror",
          "name": "European Commission"
        }
      }
    ],
    "organizations": [
      {
        "name": "CERN",
        "identifiers": [
          {
            "identifier": "01ggx4157",
            "scheme": "ror"
          }
        ]
      }
    ]
  }
}
```

#### Rename a Community

`POST /api/communities/<comid>/rename`

**Parameters**

| Name     | Type   | Location | Description                                                  |
| -------- | ------ | -------- | ------------------------------------------------------------ |
| `accept` | string | header   | - `application/json` (default)<br />- `application/vnd.inveniordm.v1+json` |

**Request**

```http
POST /api/communities/<comid>/rename HTTP/1.1
Content-Type: application/json

{
  "access": {
    "visibility": "public",
    "member_policy": "open",
    "record_policy": "open"
  },
  "id": "new_community_id",
  "metadata": {
    "title": "My Updated Community",
    "description": "This is an example Community.",
    "type": "event",
    "curation_policy": "This is the kind of records we accept.",
    "page": "Information for my community.",
    "funding": [
      {
        "funder": {
          "name": "European Commission",
          "identifier": "00k4n6c32",
          "scheme": "ror"
        },
        "award": {
          "title": "OpenAIRE",
          "number": "246686",
          "identifier": ".../246686",
          "scheme": "openaire"
        }
      }
    ],
    "organizations": [
      {
        "name": "CERN",
        "identifiers": [
          {
            "identifier": "01ggx4157", 
            "scheme": "ror"
          }
        ]
      }
    ]
  }
}
```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "access": {
    "owned_by": [
      {
        "user": <user_id>
      }
    ],
    "record_policy": "open",
    "member_policy": "open",
    "visibility": "public"
  }
  "id": "new_community_id",
  "updated": "2021-04-29T14:24:02.830457+00:00",
  "revision_id": 1,
  "created": "2021-04-29T14:24:02.806782+00:00",
  "links": {
    "self": "{scheme+hostname}/api/communities/my_community_id",
    "self_html": "{scheme+hostname}/communities/my_community_id",
    "settings_html": "{scheme+hostname}/communities/my_community_id/settings",
    "logo": "{scheme+hostname}/api/communities/my_community_id/logo",
    "rename": "{scheme+hostname}/api/communities/my_community_id/rename"
  },
  "metadata": {
    "title": "My Community",
    "type": "event",
    "curation_policy": "This is the kind of records we accept.",
    "description": "This is an example Community.",
    "page": "Information for my community.",
    "website": "https://inveniosoftware.org/",
    "funding": [
      {
        "award": {
          "number": "246686",
          "title": "OpenAIRE",
          "identifier": ".../246686",
          "scheme": "openaire"
        },
        "funder": {
          "identifier": "00k4n6c32",
          "scheme": "ror",
          "name": "European Commission"
        }
      }
    ],
    "organizations": [
      {
        "name": "CERN",
        "identifiers": [
          {
            "identifier": "01ggx4157",
            "scheme": "ror"
          }
        ]
      }
    ]
  }
}
```

#### Delete a Community

`DELETE /api/communities/<comid>`

**Parameters**

| Name     | Type   | Location | Description                                                  |
| -------- | ------ | -------- | ------------------------------------------------------------ |
| `comid`  | string | path     | Identifier of the community, e.g.  `my_community`            |
| `accept` | string | header   | - `application/json` (default)<br />- `application/vnd.inveniordm.v1+json` |

**Request**

```http
DELETE /api/communities/<comid> HTTP/1.1
Accept: application/json

```

**Response**

```http
HTTP/1.1 204 NO CONTENT
Content-Type: application/json

{}
```

#### Search Communities

`GET /api/communities`

**Parameters**

| Name     | Type    | Location | Description                                                  |
| -------- | ------- | -------- | ------------------------------------------------------------ |
| `q`      | string  | query    | Search query used to filter results based on [ElasticSearch's query string syntax](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html#query-string-syntax). |
| `sort`   | string  | query    | Sort search results (default: newest).                       |
| `size`   | integer | query    | Specify number of items in the results page (default: 10).   |
| `page`   | integer | query    | Specify the page of results.                                 |
| `type`   | string  | query    | Specify community type as one of organization, event, topic or project. |
| `accept` | string  | header   | - `application/json` (default)<br />- `application/vnd.inveniordm.v1+json` |

**Request**

```http
GET /api/communities HTTP/1.1
Accept: application/json
```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{ 
  "sortBy": "newest"
  "links": {
    "self": "{scheme+hostname}/api/communities?{params}",
    "next": "{scheme+hostname}/api/communities?{params}"
  },
  "aggregations": {
    "domain": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": []
    },
    "type": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "organization",
          "doc_count": 21
        },
        {
          "key": "event",
          "doc_count": 19
        },
        {
          "key": "topic",
          "doc_count": 13
        },
        {
          "key": "project",
          "doc_count": 10
        }
      ]
    }
  },
  "hits": {
    "hits": [
      {
        "access": {
          "owned_by": [
            {
              "user": <user_id>
            }
          ],
          "record_policy": "open",
          "member_policy": "open",
          "visibility": "public"
        }
        "id": "my_community_id",
        "updated": "2021-04-29T14:24:02.830457+00:00",
        "revision_id": 1,
        "created": "2021-04-29T14:24:02.806782+00:00",
        "links": {
          "self": "{scheme+hostname}/api/communities/my_community_id",
          "self_html": "{scheme+hostname}/communities/my_community_id",
          "settings_html": "{scheme+hostname}/communities/my_community_id/settings",
          "logo": "{scheme+hostname}/api/communities/my_community_id/logo",
          "rename": "{scheme+hostname}/api/communities/my_community_id/rename"
        },
        "metadata": {
          "title": "My Community",
          "type": "event",
          "curation_policy": "This is the kind of records we accept.",
          "description": "This is an example Community.",
          "page": "Information for my community.",
          "website": "https://inveniosoftware.org/",
          "funding": [
            {
              "award": {
                "number": "246686",
                "title": "OpenAIRE",
                "identifier": ".../246686",
                "scheme": "openaire"
              },
              "funder": {
                "identifier": "00k4n6c32",
                "scheme": "ror",
                "name": "European Commission"
              }
            }
          ],
          "organizations": [
            {
              "name": "CERN",
              "identifiers": [
                {
                  "identifier": "01ggx4157",
                  "scheme": "ror"
                }
              ]
            }
          ]
        }
      },
      {
        ...
      },
      {
        ...
      }
    ]
    "total": 64
  }
}
```

Each hit looks like a community above.

#### Search User Communities

Same as `GET /api/communities` but with the authenticated user's communities in the search results.

`GET /api/user/communities`

**Parameters**

| Name     | Type    | Location | Description                                                  |
| -------- | ------- | -------- | ------------------------------------------------------------ |
| `q`      | string  | query    | Search query used to filter results based on [ElasticSearch's query string syntax](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html#query-string-syntax). |
| `sort`   | string  | query    | Sort search results (default: newest).                       |
| `size`   | integer | query    | Specify number of items in the results page (default: 10).   |
| `page`   | integer | query    | Specify the page of results.                                 |
| `type`   | string  | query    | Specify community type as one of organization, event, topic or project. |
| `accept` | string  | header   | - `application/json` (default)<br />- `application/vnd.inveniordm.v1+json` |


**Request**

```http
GET /api/user/communities HTTP/1.1
Accept: application/json
```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{ 
  "sortBy": "newest"
  "links": {
    "self": "{scheme+hostname}/api/communities?{params}",
    "next": "{scheme+hostname}/api/communities?{params}"
  },
  "aggregations": {
    "domain": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": []
    },
    "type": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "organization",
          "doc_count": 3
        }
      ]
    }
  },
  "hits": {
    "hits": [
      {
        "access": {
          "owned_by": [
            {
              "user": <user_id>
            }
          ],
          "record_policy": "open",
          "member_policy": "open",
          "visibility": "public"
        }
        "id": "my_community_id",
        "updated": "2021-04-29T14:24:02.830457+00:00",
        "revision_id": 1,
        "created": "2021-04-29T14:24:02.806782+00:00",
        "links": {
          "self": "{scheme+hostname}/api/communities/my_community_id",
          "self_html": "{scheme+hostname}/communities/my_community_id",
          "settings_html": "{scheme+hostname}/communities/my_community_id/settings",
          "logo": "{scheme+hostname}/api/communities/my_community_id/logo",
          "rename": "{scheme+hostname}/api/communities/my_community_id/rename"
        },
        "metadata": {
          "title": "My Community",
          "type": "event",
          "curation_policy": "This is the kind of records we accept.",
          "description": "This is an example Community.",
          "page": "Information for my community.",
          "website": "https://inveniosoftware.org/",
          "funding": [
            {
              "award": {
                "number": "246686",
                "title": "OpenAIRE",
                "identifier": ".../246686",
                "scheme": "openaire"
              },
              "funder": {
                "identifier": "00k4n6c32",
                "scheme": "ror",
                "name": "European Commission"
              }
            }
          ],
          "organizations": [
            {
              "name": "CERN",
              "identifiers": [
                {
                  "identifier": "01ggx4157",
                  "scheme": "ror"
                }
              ]
            }
          ]
        }
      },
      {
        ...
      },
      {
        ...
      }
    ]
    "total": 3
  }
}
```

Each hit looks like a community above, belongs to the authenticated user.

#### Update Community Logo

`PUT api/communities/<comid>/logo`

**Parameters**

| Name           | Type   | Location | Description                                                  |
| -------------- | ------ | -------- | ------------------------------------------------------------ |
| `comid`        | string | path     | Community name                                               |
| `content-type` | string | header   | Should always be `application/octet-stream`.                 |
| `accept`       | string | header   | - `application/json` (default)<br />- `application/vnd.inveniordm.v1+json` |

**Request**

```http
PUT api/communities/<comid>/logo HTTP/1.1
Content-Type: application/octet-stream

<...file binary data...>
```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "bucket_id": "d3c493fd",
  "checksum": "md5:96d6f2e7e1f705ab5e59c84a6dc009b2",
  "created": "2021-04-26 10:52:23.945755",
  "file_id": "d2a7adb5",
  "key": "logo",
  "metadata": None,
  "mimetype": "application/octet-stream",
  "size": file_size,
  "status": "completed",
  "storage_class": "S",
  "updated": "2021-04-26 10:52:24.562652",
  "version_id": "b95ead95",
  "links": {
    "self": "{scheme+hostname}/api/communities/<comid>/logo"
  }
 }
```

#### Get Community Logo
{scheme+hostname}
`GET api/communities/<comid>/logo`

**Parameters**

| Name    | Type   | Location | Description    |
| ------- | ------ | -------- | -------------- |
| `comid` | string | path     | Community name |

**Request**

```http
GET api/communities/<comid>/logo HTTP/1.1

```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: application/octet-stream

<...file binary data...>
```

#### Delete Community Logo

`DELETE api/communities/<comid>/logo`

**Parameters**

| Name    | Type   | Location | Description    |
| ------- | ------ | -------- | -------------- |
| `comid` | string | path     | Community name |

**Request**

```http
DELETE api/communities/<comid>/logo HTTP/1.1

```

**Response**

```http
HTTP/1.1 204 NO CONTENT
Content-Type: application/octet-stream

```


#### Error Responses of Community

`POST /api/communities`

**Parameters**

| Name     | Type   | Location | Description                                                  |
| -------- | ------ | -------- | ------------------------------------------------------------ |
| `accept` | string | header   | - `application/json` (default)<br />- `application/vnd.inveniordm.v1+json` |

**Request**

```http
POST /api/communities HTTP/1.1
Content-Type: application/json

{
  "id": "comm_id",
}
```

**Response**

```http
HTTP/1.1 400 BAD REQUEST
Content-Type: application/json

{
  "errors": [
    {
      "field": "metadata",
      "messages": [
        "Missing data for required field."
      ]
    },
    {
      "field": "access",
      "messages": [
        "Missing data for required field."
      ]
    }
  ],
 "message": "A validation error occurred.",
 "status": 400
}
```

### Mock-ups/views

See mockups at https://github.com/inveniosoftware/mockups/tree/master/rdm/communities.

## Example

The actual code implementation for communities is currently at https://github.com/inveniosoftware/invenio-communities.

## How we teach this

Explaining the core concept of communities should be simple given their user-stories, and connection to real world entities. As a standalone feature at the moment they do not provide much functionality, so future additions (records and members integration), will probably provide better points for understanding the purpose of communities.

## Drawbacks

For the current implementation there are no obvious drawbacks in terms of overall modeling. The excessive reuse of existing APIs might be an issue in case configuration or functionality changes over time. 

## Alternatives

No alternatives were considered during the design.