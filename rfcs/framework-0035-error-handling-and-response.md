- Start Date: 2020-07-15
- RFC PR: [#35](https://github.com/inveniosoftware/rfcs/pull/35)
- Authors: Lars Holm Nielsen, Zach Zacharodimos
- State: DRAFT

# Error handling principles for REST API error responses in Invenio

## Summary

Error response bodies are not subject to content negotiation and  thus always JSON formatted. Further a simple method for error handling and returning JSON formatted responses is presented. The method relies on Flask error handlers and extension of the HTTPException.

## Motivation

Clear and consistent error handling throughout Invenio REST APIs is highly important for the clients accessing our REST APIs, as well as for properly logging errors and tracking errors. It's further important for a clean code architecture.

## Detailed design

### Scope

The RFC is about general error handling principles in Invenio-based applications focusing primarily on the REST APIs. It's documenting the reasoning and high-level implementation of error handling in Invenio. It's not supposed to be prescriptive about how users of Invenio Framework would like to do their error handling, only about the approach used in Invenio modules and applications. There might be good reasons to diverge from the general principles in specific cases. However, when diverging it's important to cross-check with this RFC, to understand why it's necessary to diverge, and what are the implications.

### Content negotiation in Invenio REST API

The Invenio REST APIs makes use of HTTP Content Negotiation in order to render a record in multiple different formats.

**Example of content negotiation with succcessful response**
*Request*

```http
GET /records/1 HTTP/1.1
Accept: application/xml
```

*Response*

```http
HTTP/1.1 200 OK
Content-Type: application/xml
...

<record>
    ...
</record>
```

Notice, the Accept and Content-Type header. If you ask for ``application/json`` instead, you could get a JSON-formatted HTTP response body instead of XML-formatted.


### Should error responses use content negotiation?

A key question is if an error response should also be subject to HTTP Content Negotiation? For instance, if the following HTTP request results in a HTTP 404 Not Found error, what does the response looks like:

*Request*

```http
GET /records/1 HTTP/1.1
Accept: application/xml
```

*Response example 1 (with content negotiation)*

The client asked for XML so an XML response is returned.

```http
HTTP/1.1 404 Not Found
Content-Type: application/xml
...

<error>
    <code>400</code>
    <message>The requested resource was not found.</message>
</error>
```

*Response example 2 (without content negotiation)*

The client asked for XML, but instead a JSON response is returned for all errors.

```http
HTTP/1.1 404 Not Found
Content-Type: application/json
...

{
    "code": 404,
    "message": "The requested resource was not found."
}
```


**Error response with content negotiation (example 1)**

At a theoretical level, the above example may seem logical that a client requests XML, and gets an error response in XML. However for other formats, this becomes less clear. For instance, some formats may not easily encode errors - e.g. how would an error response look like for CSV or BibTeX?

Similar, at an architecture-level, an application deals with a wide range of errors, and decoupling the error from the serialization of the response body, means that we need very clear definition of errors so that the error serializers have a limited set of errors as input to deal with, otherwise it becomes an overwhelming task to write an error serializer. Currently, this does not seem feasible in Invenio code base.

Also, at an architecture-level, errors may come from many sources, . Take e.g. a 404 Not Found error. This one can come from the service layer or from the Flask routing layer. This would mean error handling would need to be global for the application in order to have consistent serialization of both errors.

At a client-level, the most important is having a machine-readable format for the error, and especially it's important for Single-Page applications in order to display errors. The response will always have a content type defined, and thus it's reasonable straightforward to handle errors in a different format that what was requested, especially if it is only for errors.

**Error response without content negotiation (example 2)**

Having only a single response format for errors simplifies all of above mentioned issues with the content negotiation. Serializers needs only to deal with how to serialize a successful response and not a plethora of errors, and thus becomes easier to write. Having only one format, means errors can easily be generated from multiple different sources and produce the same error (e.g. 404 from service vs routing layer), and clients can easily decode errors and know it's always the same format.

Because of above reasons, it means that *error response bodies are not subject to content negotiation and thus always JSON formatted*

### Who is responsible for rendering the HTTP response body for an error?

Having established that error response are not subject to content negotiation, the next question is who is responsible for rendering the HTTP response body for an error.

IMPORTANT: As a prerequisite, please read [error handling in Flask](https://flask.palletsprojects.com/en/1.1.x/errorhandling/#error-handlers).

**Sources of errors**

The following are usual sources of errors (i.e. exceptions being raised):

1. Flask layer, for instance routing errors.
1. View layer, for instance ``abort(400)`` or ``raise BadRequest()``.
1. Service/data source layer, for instance Elasticsearch ``RequestError`` or ``PIDDeleteError``.

A bit oversimplified, you can say that Flask/view layer errors usaully  result in HTTP errors such as 40x errors (Bad Request, Not Found).

Service/data source layer errors will usually result in 500 Internal Server Error *unless they are handled*.

For all above type of errors, we'd like to create a machine readable error response.

**View error handling**

A Flask view can simply catch exceptions, and render a response. This can either be done per view or via special decorators.

```python
@app.route('/')
def view1():
    try:
        # ...
    except Exception:
        return 400, ...
```

The view layer error handling is best suited for very custom error handling from the service layer. E.g implementation of a custom API with custom reposnses.

The issue with view layer error handling is that only errors from the view and service/data source layer is being caught. Any error in the Flask layer (routing, request processing functions etc) will escape the try/except-clause (because they happen outside the function).

Example of a 404 error:

1. A routing error will result in a 404 error from the Flask layer.
2. An object that doesn't exist in the database, will likely result in a 404 error from the service layer.

A view layer can only catch the latter. Thus, if we render an error response in the view, we still have to deal with error response for Flask layer errors.

**Flask's error handling/rendering**

Flask and Werkzeug's request dispatching system will catch all errors happening in the three layers described. The error handling is pretty sophisticated, and is explained below.

First, Flask will look for a user-provided error handler. If found, the handler is responsible for making a response. If not found, but the exception is an HTTPException (or subclass thereof), the HTTPException will simply be returned, because an HTTPException can self-render. If it's not an HTTPException, the error is simply reraised and results in an Internal Server Error. The following pseudo-code is an example of the handling:

```python
def handle_user_exception(e):
    # If it's an HTTPException, use different method:
    if isinstance(e, HTTPException):
        return handle_http_exception(e)

    handler = find_error_handler(e)
    if handler is None:
        # No handler was found, and it's not and HTTPException.
        raise # <----- reraise
    # A handler was found, use it to return response
    return handler(e)

def handle_http_exception(e):
    handler = find_error_handler(e)
    if handler is None:
        # No handler was found, but it is an HTTPException.
        return e # <----- return
    # A handler was found, use it to return response
    return handler(e)
```

The key in understanding the difference between an normal exception and an HTTPException is that an HTTPException is considered a valid response by Flask's ``make_reponse``. This allows it to be returned by the ``handle_http_exception()`` function.

**Recap**

Source of errors (exceptions raised) is three layers (Flask, View, Service). For handling these errors (i.e. making an HTTP error response) we can use Flask and Werkzeug's error handling system. The view layer is not well-suited for this because it doesn't catch all errors.

Above basically establish that either

1. an *exception* can render itself into an HTTP error response, or
2. an *error handler* can render an exception into a HTTP error response.

### Error handling in Flask REST API applications

Following is a proposal for how to implement the error handling in REST API **application**:

**JSON-formatted HTTP Exception**

First we propose to create a new subclass of ``HTTPException``, that will render the response body as JSON. In simplified pseudo-Python  this would be something like:


```python
class HTTPJSONException(HTTPException):
    def get_body(self, environ=None):
        body = dict(
            status=self.code,
            message=self.get_description(environ=environ),
        )
        return json.dumps(body)):
```

**Flask application**

Next, we install an error handler on the Flask application for ``HTTPExceptions``:

```python
from Flask import Flask

app = Flask(__name__)

@app.errorhandler(HTTPException)
def handle_http_exception(e):
    return HTTPJSONException(code=e.code, description=e.description)
```

This will ensure that all errors generated at the Flask layer will have a proper JSON-formatted error response.

**Blueprints**

Last, for blueprints containing the views of the REST API, we propose to add error handlers that able to map service errors to ``HTTPJSONException``:

```python
# A function that creates an error handler.
# The error handler, maps the exception into an HTTPException.
def create_error_mapper(exc):
    def handler(e):
        return HTTPJSONException(code=e.code, description=e.description)
    return handler

# A list of service errors, and their handlers
error_map = {
    ServiceError: create_error_mapper(HTTPJSONException),
    # ...
}

# Install handlers on blueprints.
def register_error_map(blueprint, error_map):
    for exc_or_code, handler in error_map.items():
        blueprint.register_error_handler(exc_or_code, handler)
```

#### Usage

The above error handler technique, can the be used with the existing exceptions and Flask idoms:


```python
@blueprint.route('/')
def view():
    if True:
        abort(400)
    raise BadRequest()

```

Service errors, needs to be mapped, but will otherwise also be caught and reply with a JSON-formatted response body:

```python
error_map = {
    ServiceError: create_error_mapper(HTTPJSONException)
}

@blueprint.route('/')
def view():
    raise ServiceError()


register_error_map(blueprint, error_map)
```

More specific REST API errors, can then also be created and raised directly without further handling:

```python
class ValidationError(HTTPJSONException):
    code = 400
    description = "My description"


@blueprint.route('/')
def view():
    raise ValidationError()

```

### Existing Invenio implementation

**Flask application error handler**

Currently Invenio-REST defines the Flask application error handlers per HTTP code. This should like be changed to a single handler for all HTTP exceptions.

We could simply start with an implementation of this in Flask-Resources, and at somepoint look to migrate away the Invenio-REST implementation.

**HTTPJSONException**

Invenio-REST currently defines a ``RESTException`` which is similar to ``HTTPJSONException``. We propose to add ``HTTPJSONException`` in Flask-Resources and let all new modules use this exception.

In principle there's no problem in having both RESTException and HTTPJSONException as this is one of the strengths of the approach. That code in other modules can dependent on different implementations, but still produce the same error response to a client. This ensures better isolation of code.

**Blueprint error handlers**

These will be implemented in the Flask-Resources.

## How we teach this

Should be documented as part of the Flask-Resources doucmentation, possibly with a chapter in main Invenio documentation.

## Drawbacks

Not applicable (already detailed above)

## Alternatives

Not applicable (already detailed above)

## Unresolved questions

None

## Resources/Timeline

Needed for a decision on implementation choice in Flask-Resources.
