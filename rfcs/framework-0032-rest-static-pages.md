- Start Date: 2020-04-06
- RFC PR: [#<PR>](https://github.com/inveniosoftware/rfcs/pull/32)
- Authors: Karolina Przerwa

# REST API for Invenio-Pages

## Summary

Add REST API `invenio-pages` module, in order to display static pages in Single Page Apps (SPA).

## Motivation

Use case: As a developer I would like to retrieve a static page using REST API to display them in my SPA application.

This will allow to CRUD pages from Invenio CLI or Admin web interface and display changes on the fly, without the need of deploying the SPA application. Since pages are meant to be public, there is no need to implement access restrictions.

## Detailed design

Create a new extention in `invenio-pages` to expose a new REST endpoint. The endpoint will implement only the `GET` method to retrieve a single page from the database.
    
Retrieving the list of available pages is currently out of scope.

To improve performamce, the GET method will return the ETag with the version of the page to allow browser caching.
    
Proposed payload:

```json
{
  title: "test",
  content: "<p>test</p> ",
  id: "1",
  description: "test",
  url: "/about",
  links: {
     self: "https://localhost:5000/api/pages/1"
  }
}
```
    
### Content with HTML

The content of a page can contain HTML tags. After some research, there is no clear recommendation if HTML tags should be encoded or not, and opinions diverge.
The most sensible solution is that the backend should not make assumption how APIs will be consumed, so it is probably better to not encode HTML tags.

Content with HTML might expose the site to XSS vulnerabilities. Ideally, vulnerable HTML code (such as javascript or img for example) should be sanitizied when inserting or updating content, and not when retrieving it. On top of that, the creation/edition of content can be performed only by admin users, which should limit the risk factor.
That's why input sanitization is not in the scope of this change.

## Example

The easiest example is a `Contact Us` static page: it is often displayed in the homepage, in the footer, and it might contain useful info such as access and opening hours. It can happen that users want to change the content of such page from time to time.

In a SPA, the easiest way to achieve this is to use and store the content in `invenio-pages`. This allows power users to simply go to the admin panel and modify the content: the modification will be immediately applied and visible.

## How we teach this

The feature will be used in `invenio-app-ils` SPA and it can be demonstrated there.
    

## Drawbacks

Since listing of pages will not be implememnted, there is no easy way from a SPA to retrieve what pages are available. Therefore, creation or deletion of pages will require code changes in the SPA. This is partially a limitation, given that creation of new pages will very probably require code changes in a SPA: the user will have to decide where to display the link to reach such new page.
    

## Alternatives

Several alternatives have been explored: 
- static pages could be hardcoded in the SPA, which reduces software complexity but it requires code changes and deployment if the static page content changes.
- static pages could be just config or stored in a simple JSON files, but it will have similar drawbacks as explained above.

In the end, adding REST APIs to `invenio-pages` sounded the most reasonable choice.


## Unresolved questions

TBD: POST method, PUT/PATCH, DELETE to achieve the full control of pages through SPA

Potentially the pages could be a record to optimise the implementation and keep it consistent.