# Web API Design Guidelines

Guidelines for designing good RESTful APIs.

## Table of Content

- [Overview](#overview)
- [Endpoints](#endpoints)
- [Responses](#responses)
- [Requests](#requests)
  - [GET](#get)
  - [POST](#post)
  - [PATCH](#patch)
  - [PUT](#put)
  - [DELETE](#delete)
  - [POST for Update](#post-for-update)
  - [PUT for Create](#put-for-create)
- [HTTP Status Code Summary](#http-status-codes-summary)
- [Root URL](#root-url)
- [Entry Point Response](#entry-point-response)
- [Versioning](#versioning)
- [Documentation](#documentation)
- [Timestamps](#timestamps)
- [Hypermedia](#hypermedia)
  - [Resource Reference](#resource-reference)
  - [Collection Reference](#collection-reference)
- [Expands](#expands)
- [Searches](#searches)
  - [Attribute Search](#attribute-search)
  - [Filter Search](#filter-search)
  - [Global Search](#global-search)
- [Pagination](#pagination)
  - [Parameters](#parameters)
  - [Pagination Object](#pagination-object)
- [Sorting](#sorting)
- [Count](#count)
- [Partial Representations](#partial-representations)
- [Overriding the HTTP Verbs](#overriding-the-http-verbs)
- [Async or Long-Lived Operations](#async-or-long-lived-operations)
- [Chatty APIs](#chatty-apis)
- [3 Seconds Policy](#3-seconds-policy)
- [Health Checks](#health-checks)
- [Monitoring Policy](#monitoring-policy)

## Overview

This document serves as a guideline for all API development in the company. Please read the document and provide [feedback](https://github.com/cebroker/guidelines/issues) based on your own experience building APIs. This is not intended to be a final document but a collaborative team effort.

> **Note:** This is not a Web API tutorial. It assumes you have a good understanding on REST API concepts. You won't find languages, frameworks, nor any other underlaying technologies references here.

## Endpoints

The key principles of REST involve separating your API into logical resources.
An Endpoint is a URL within the API which points to a specific resource or collection of resources.

For **naming resources**, use plural *nouns* not *verbs* in the URLs. Use nouns that belong to the project domain so they make more sense to developers.

Following URLs smell like bad  [RPC](https://en.wikipedia.org/wiki/Remote_procedure_call). `DON'T DO THIS`:

```
/getAllUsers
/getInactiveUsers
/searchUsers
/createUser
/updateUser
/validateUserName
/deleteUser
/deleteLicenseFromUser
```

Instead, use a combination of HTTP verbs and plural nouns to access or manipulate the resources. These are some good examples:

ENDPOINT | DESCRIPTION
------- | ---
`GET` /users | Retrieves a collection of users
`GET` /users/123 | Retrieves a specific user
`POST` /users | Creates a new user
`PUT` /users/123 | Updates user #123
`PATCH` /users/123 | Partially update user #123
`DELETE` /users/123 | Deletes user #123

In the next sections, we are going to explain the proper use of these methods and expected results according to the status of the operation.

## Responses

Use simple JSON objects for results and camelCase for naming fields.

```
GET /users/123
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{
    "href": "/users/123",
    "id": 123,
    "firstName": "Jane",
    "lastName": "Doe",
    "age": 25,
    "ssn": 999111888
}
```

Every accessible resource has a canonical unique URL that should be provided as part of the response in the `href` attribute. This is critical for linking. See [hypermedia](#hypermedia).

## Requests

As we said before, resources are manipulated using HTTP requests where the method or HTTP verb has a specific meaning.

VERB | DESCRIPTION
--- | ---
HEAD | Can be issued against any resource to get just the HTTP header info.
GET | Used for retrieving resources.
POST | Used for creating resources.
PATCH | Used for updating resources with partial JSON data. A PATCH request may accept one or more of the attributes to update the resource. PATCH is a relatively new and uncommon HTTP verb, so resource endpoints must also accept POST requests for partial update.
PUT | Used for replacing resources or collections.
DELETE | Used for deleting resources.

In case a method is not supported for the resource, we must return a `405 Method Not Allowed` error, including in the response an `Allow` header containing a list of valid methods for the requested resource.

### GET

Use `GET` for retrieving resources and collections. As we explained before, every accessible resource has a canonical unique URL which represents it. This is applicable for both instances resources and collection resources.

#### Collections

A collection represents a set of resources. If the number of resources is small and the growing rate is low, we can return a simple array with the entire collection.

```
GET /users
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

[
  {
    "href": "/users/1",
    "id": 1,
    "firstName": "John",
    "lastName": "Doe",
    "age": 25,
    "ssn": 999111888
  },
  {
    "href": "/users/2",
    "id": 2,
    "firstName": "Scott",
    "lastName": "Tiger",
    "age": 36,
    "ssn": 999111777
  },
  {
    "href": "/users/3",
    "id": 3,
    "firstName": "Jane",
    "lastName": "Doe",
    "age": 21,
    "ssn": 999111666
  },
]
```

But if the collection is large or can grow drastically in the future, we must return a paging object instead. See [pagination](#pagination) for details.

#### Resources

A resource represents a single instance of an object.

```
GET /users/1
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{
  "href": "/users/1",
  "id": 1,
  "firstName": "John",
  "lastName": "Doe",
  "age": 25,
  "ssn": 999111888
}
```

When trying to access a resource that doesn't exist, we must return the HTTP Status Code `404 Not Found` with an empty body.

```
GET /users/1789
HTTP/1.1 404 Not Found
Content-Type: application/json; charset=utf-8
```

Check the [HTTP Status Codes Summary](#http-status-codes-summary) to see other possible responses.

#### Sub-resources

A sub-resource represents a relation between two resources. So for example, to get the addresses of a user we can do this:

```
GET /users/1/addresses
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

[
  {
    "href": "/users/1/addresses/881"
    "id": 881,
    "streetAddress": "1600 Pennsylvania Avenue NW",
    "city": "Washington",
    "state": "DC"
  },
  {
    "href": "/users/1/addresses/882"
    "id": 882,
    "streetAddress": "1st Street SE",
    "city": "Washington",
    "state": "DC"
  }
]
```

Use sub-resources to express relations as a preferred method always that makes sense. Otherwise use a search pattern instead.

```
GET /addresses?user=1
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

### POST

Use `POST` for creating resources. The URL of the created resources must be set  in a Location header as part of the response.

```
POST /users
HTTP/1.1 200 OK
Location: https://api.example.com/v1/users/109
```

You can optionally return the body of the resource created too. It can save a new call to the API in case additional data be created sever side as part of the operation.

### PATCH

Use `PATH` for partially updating resources. A PATCH request may accept one or more of the attributes to update the resource.

```
PATCH /users/1
HTTP/1.1 200 OK
```

If the resource to be updated doesn't exist, it must return a `404 Not Found` error.

> **Important**: PATCH is a relatively new and uncommon HTTP verb, so resource endpoints must also accept [POST requests for partial update](#post-for-update).

### PUT

Use `PUT` for replacing resources. It requires the entire object be provided.

```
PUT /users/1
HTTP/1.1 200 OK
```

If the resource to be replaced doesn't exist, a new resource must be created by using the given identifier. See [PUT for Create](#put-for-create).

PUT is [idempotent](http://restcookbook.com/HTTP%20Methods/idempotency/), it means it can be called many times and the result should be the same.

Do not use this method for partial updates. See [PATCH](#patch).

### DELETE

Use `DELETE` for deleting resources.

```
DELETE /users/1
HTTP/1.1 204 No Content
```

If some content is returned as a result of the operation, use `200 OK` status code instead.

The DELETE method is idempotent. This implies that the server must return response code `200 OK` or `204 No Content` even if the server deleted the resource in a previous request.

You can also use this method for deleting entire collections. For example, you can do following call to remove all the addressed of a user:

```
DELETE /users/1/addresses
HTTP/1.1 204 No Content
```

> Note: After a deletion, the resource affected should not be accessible anymore through `GET`, even if you are using soft deletion. In case you still needing to access the resource then you must use `PATCH` for update the deletion flag.

### `POST` for Update

The API **must** support using `POST` for partially updating a resource. The reason for this is `PATCH` method is not fully supported and `PUT` cannot be used for partial updates.

```
POST /users/1
HTTP/1.1 200 OK
```

Basically, it must operate in the same exactly way `PATCH` does. So you must provide the URL of an existing resource, return `404 Not Found` if the resource doesn't exist or `200 OK` if the resource was successfully updated.

> **Important:** you should never create a resource with `POST` if the URL of the resource is specified. Use `PUT` instead.

### `PUT` for Create

Use PUT when you allow the client to specify the resource identifier of the newly created resource. But remember, since PUT is [idempotent](http://restcookbook.com/HTTP%20Methods/idempotency/), you must send all possible values.

```
PUT /users/1
HTTP/1.1 200 OK
```

In this case, we don't have to return the Location header for the newly created resource since it must be created in the URL provided.

If you don't know the resource identifier then use `POST` for creating the new resource.

> To learn more about `POST` and `PUT` uses check this [Stack Overflow answer](http://stackoverflow.com/questions/630453/put-vs-post-in-rest#answer-630475) and this [Stormpath blog post](https://stormpath.com/blog/put-or-post)

## HTTP Status Codes Summary

We should try to use the most appropriate [HTTP status code](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes) for any particular result we return back.

HTTP Status Code | Description
--- | ---
200 | `OK` Standard response for successful HTTP requests
201 | `Created` Successfully POST method when a new resource was created. The URL of newly created resource must be returned
400 | `Bad Request` The client did something wrong. e.g. missing or unaccepted parameters
401 | `Unauthorized` Actually means Unauthenticated. Authentication is required and has failed or has not yet been provided
403 | `Forbidden` Actually means Unauthorized. The resource exists but the user cannot access it
404 | `Not Found` The requested resource could not be found
405 | `Method Not Allowed`  The method specified is not allowed for the resource
500 | `Internal Server Error` The server did something wrong

You can find an extensive list of codes [here](http://www.restapitutorial.com/httpstatuscodes.html).

## Root URL

It’s important that the root entry point into the API is as simple as possible, as a long complex URL will appear daunting and can turn developers away. If your application is huge, or you anticipate it becoming huge, putting the API on its own subdomain, is a good choice. This can allow for some more flexible scalability down the road.

```
https://api.example.com/v1
```

But if you anticipate your API will never grow to be that large, or you want a much simpler application setup (e.g. you want to host the website and API from the same framework), placing your API beneath a URL segment at the root of the domain works as well.

```
https://example.com/api/v1
```

In both cases, the API must be exclusively served over `HTTPS`. This is important to protect both our site and our users from attack.

## Entry Point Response

It’s a good idea to have content at the root of the API. Hitting the root of GitHub’s API returns a listing of endpoints, for example. We must provide information which a lost developer would find useful, e.g., how to get to the developer documentation for the API.

```
GET https://api.example.com/v1
HTTP/1.1 200 OK
```

```json
{
  "name": "example",
  "version": "1.2.3",
  "author": "CE Broker Inc.",
  "description": "The API for saving the world.",
  "documentation": "https://api.example.com/docs",
  "repository": "https://github.com/cebroker/example",
  "end-points": {
    "user_url": "https://api.example.com/v1/users{/id}",
    "address_url": "https://api.example.com/v1/users/{user}/addresses{/id}"
  }
}
```

## Versioning

An API is a published contract between a client and a server. If you make changes to the API and these changes break backwards compatibility, you will break things for your clients and they will resent you for it. Changes that break the contract could be for example removing or renaming an endpoint or a field from the results.

However things will always change. So in case you need to add changes that break the contracts, expose a new version of the API. This is why it is so important the API supports versioning since day 1. If you don't want the API gods to get really mad at you, don't you ever even think about releasing an API without versioning.

There are several ways to do versioning, but we agreed we should put the version of the API at the end of the base URL like in the examples above.

## Documentation

An API is only as good as its documentation. The docs should be easy to find and accessible. Most developers will check out the docs before attempting any integration effort. When the docs are hidden inside a PDF file or require signing in, they're not only difficult to find but also not easy to search.

These are our rules of thumbs:

1. Write the documentation as plain HTML or Markdown.
1. Do not use automatic documentation generators.
1. Expose the content in a website accessible to unauthenticated developers.
1. Include endpoint reference, object model, and authorization guide.
1. Show examples of complete request/response cycles.

[Spotify](https://developer.spotify.com/web-api),
[GitHub](https://developer.github.com/v3/gists/#list-gist) and [Stripe](https://stripe.com/docs/api) do a great job with this.

## Timestamps

Timestamps are returned in [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) format as [Coordinated Universal Time](https://en.wikipedia.org/wiki/UTC_offset) (UTC) with zero offset: YYYY-MM-DDTHH:MM:SSZ

```json
{
  "createdAt": "2013-07-10T18:02:24.343Z"
}
```

## Hypermedia

*Optional. Implement only when needed.*

All resources may have one or more attributes linking to sub-resources. This will allow clients to get additional information about a resource without having to construct the URLs themselves. We recommend all clients to use those attributes. If sub-resource URLs change, client won't be affected because the new URL will be provided to them by the API, no clients code needs
to change. Linking is fundamental to scalability.

### Resource Reference

Let's say a user belongs to an organization. Then, in the resource model we have to define that reference. In order to do that we should specify the URL to the referenced resource with a `href` attribute.

```
GET /users/123
HTTP/1.1 200 OK

{
    "href": "/users/123",
    "firstName": "Jane",
    "lastName": "Doe",
    "age": 25,
    "organization": {
        "href": "https://api.example.com/v1/organizations/a5c2"
    }
}
```

### Collection Reference

But what when the reference is actually a collection? Let's consider for example the addresses of a user. Basically it follows the same principle than instance references, we should provide the URL to the collection by using an href attribute.

```
GET /users/123
HTTP/1.1 200 OK

{
    "href": "/users/123",
    "firstName": "Jane",
    "lastName": "Doe",
    "age": 25,
    "addresses": {
        "href": "https://api.example.com/v1/users/123/addresses"
    }
}
```

## Expands

One important thing is how to load referenced resources without having to perform several calls to the API. This is handled by expansion. For example, in order to get all user's information including his/her organization we can do this:

```
GET /users/123?expand=organization
HTTP/1.1 200 OK

{
    "href": "/users/123",
    "firstName": "Jane",
    "lastName": "Doe",
    "age": 25,
    "organization": {
        "href": "https://api.example.com/v1/organizations/a5c2",
        "name": "CE Broker Inc."
        "address": {
          "href": "https://api.example.com/v1/organizations/a5c2/address"
        }
    }
}
```

## Searches

Sometimes we don't need the whole set of data, we are just interested in a set of data that meets certain criteria. We called that a resource's search and we use `?` to help with those variations.

### Attribute search

We perform searches by specific attributes of a resource.

For example:
*Get a list of all active users who live in Cartagena.*

```
GET /users?status=active&city=cartagena
```

Wildcards could be also supported.

*Get a list of all users whit a "cebroker.com" email address.*

```
GET /users?email=*cebroker.com
```

This is a list of possible wildcards:
- Starts with: `?email=joe*`
- Ends with: `?email=*company.com`
- Contains: `?email=*foo*`

We could also support search by ranges:

```
GET /users?createdAt=[2015-02-01,2015-05-31]
```

### Filter search

For more generic searches, meaning that you give a developer the possibility of looking for any value that match, we can use a filter search. For filter search, you must use the parameter `q`, which stands for *query*.

```
GET /users?q=some+value
```

### Global search

If want to provide the ability for developers to do a global search across multiple resources, we should do what the search gods at Google do:

```
GET /search?q=where+are+my+pants
```

Wait but, `search` is not a noun! Well, we are not RESTafarians. This is one of those occasions where we don't use nouns. This is not quite common however. At the time this guideline was written, any global search had been implemented.

## Pagination

By default and for large collections, all searches should return the first set of results, unless the client requests an specific set of data by sending specific values to: `offset` and `limit` parameters. Here are some examples:

Request | Result
------ | ---
`GET` /users | Returns first page of results with default limit (25) and offset (0)
`GET` /users?offset=50 | Returns third page of results with default page size
`GET` /users?limit=30 | Returns first page of results with page size of 30
`GET` /users?offset=30&limit=10 | Returns fourth page of results with page size of 10

### Parameters

Name | Description | Default Value | Min Value | Max Value
--- | --- | --- | --- | ---
offset | Number of rows to skip | 0 | 0 | N/A
limit | Number of rows to return | 25 | 1 | 100

### Pagination Object

As a result of a paginated request we must return a pagination object matching the following structure:

Field | Description
--- | ---
href | the URL to the collection.
offset |  number of elements skipped.
limit | max number of elements returned.
size | total size of collection.
first | url to first set of elements. *Optional*
previous | url to previous set of elements. `null` if first. *Optional*
next | url to next set of elements. `null` if no more. *Optional*
last | url to last set of elements. *Optional*
items | current set of elements returned.

Following is a example of paginated collection of users:

```json
{
    "href": "/users",
    "offset": 0,
    "limit": 25,
    "size": 3,
    "first": "/users/offset=0",
    "previous": null,
    "next": null,
    "last": "/users?offset=0",
    "items": [
      {
        "href": "/users/1",
        "id": 1,
        "firstName": "John",
        "lastName": "Doe",
        "age": 25,
        "ssn": 999111888
      },
      {
        "href": "/users/2",
        "id": 2,
        "firstName": "Scott",
        "lastName": "Tiger",
        "age": 36,
        "ssn": 999111777
      },
      {
        "href": "/users/3",
        "id": 3,
        "firstName": "Jane",
        "lastName": "Doe",
        "age": 21,
        "ssn": 999111666
      }
    ]
}
```

## Sorting

In order to get a list of results following certain order, the `sort` parameter should be specified:

```
GET /users?sort=firstName
```

The api may support sorting by multiple fields like so:

```
GET /users?sort=firstName,age
```

To perform a descending sorting use a hyphen as a prefix of the field:

```
GET /users?sort=-age
```

## Count

Sometimes we need to know the size of collection without actually getting the data. To support this return a custom header with the total number of items: `X-Total-Count`

Then use `HEAD` to get headers without the body. Like so:

```
HEAD /users
X-Total-Count: 1251
```

It also covers searches for free:

```
HEAD /users?status=active&city=cartagena
X-Total-Count: 300
```

With this technique you avoid creating new non-resources endpoints and having to add extra search capabilities on it. So **Don't do this**:

```
/users/count
/users/count?status=active&city=cartagena
```

## Partial Representations

*Optional. Implement only when needed.*

What if developers don't need all the attributes we have defined for a given resource? well, they should define the list of fields they need by using the `?fields` parameter.

Let's say we are only interested in the user's name and the zip code for all his/her addressees. We got you covered:

```
GET /users?status=active&fields=name,address.zip

GET /users/123?fields=name,address.zip
```

## Overriding the HTTP Verbs

*Optional. Implement only when needed.*

Some HTTP clients can only work with simple GET and POST requests. To increase accessibility to these limited clients, the API needs a way to override the HTTP method. Although there aren't any hard standards here, the popular convention is to accept a request header `X-HTTP-Method-Override` with a string value containing one of PUT, PATCH or DELETE.

Note that the override header should only be accepted on POST requests. GET requests should never change data on the server!

## Async or Long-Lived Operations

Sometimes operations performed against the API could take a really long time to be completed. For those kind of operations we shouldn't wait to the operation is completed to return a response. Instead we must return and `202 Accepted` HTTP response along with a URL in the Location header where we can check the result of the operation in a subsequent step.

```
POST /emails
{
 “from”: me@somewhere.com,
 “subject”: “Hi!”
 “body”: “...”
}
```

```
204 Accepted
Location: /emails/23Sd932sSl
{
 “status”: “queued”
}
```

Later, we can check the progress by using the given location:

```
GET /emails/23Sd932sSl
Expires: 2014-09-29T18:00:00.000Z
{
 “status”: “sent”
}
```

The information in the response body could include the percentage of progress of the current operation.

## Chatty APIs

Building RESTful APIs can sometimes lead to developer making dozens of calls just to present the information they need to build a simple UI or application, resist the temptation to create end points that return composite results right away! First, concentrate in designing and implementing the API following this guideline, and then, evaluate if adding composite end points will make a big percentage of your developers happy. If so, why not? but, first things first, a very well design API comes first, then, it's probably OK to create 'shortcuts'.

## 3 Seconds Policy

API end-points should never take longer than 3 seconds to respond. If for some for reason, the underlaying technology doesn’t provide the required level of performance, then you have to consider using something else. We now have several alternatives, just pick the right thing.

## Health Checks

Health checks determine the availability of the service by checking the status of downstream components the API relies on. We use these responses to make operational decisions for caching or failover which are essemtial to our goal of providing highly available solutions.

API's Health Check endpoint: `/health`

Health Check Response Format for HTTP APIs uses the JSON format and has the media type "application/health+json". Its content consists of a single mandatory root field ("status") and several optional fields:

- status. Required. Indicates whether the service status is acceptable or not. Possible values:
  - "pass": healthy.
  - "fail": unhealthy.
  - "warn": healthy, with some concerns.

The value of the status field is case-insensitive and is tightly related with the HTTP response code returned by the health endpoint. For "pass" status, HTTP response code in the 2xx-3xx range must be used.  For "fail" status, HTTP response code in the 4xx-5xx range must be used.  In case of the "warn" status, endpoints must return HTTP status in the 2xx-3xx range, and additional information should be provided, utilizing optional fields of the response.

- version. Optional. Public version of the service.
- output. Optional. It is raw error output, in case of "fail" or "warn" states.  This field should be omitted for "pass" state.
- service. Optional. It is a unique identifier of the service, in the application scope. e.g. <product>.<division>.<api-name>
- description. Optional. It is a human-friendly description of the service.
- instance. Optional. It is a unique identifier of the service's instance (e.g host name).
- checks. Optional. It is an object that provides detailed health statuses of additional downstream systems and endpoints which can affect the overall health of the main API.
  - The key identifying an element in the object should be a unique string within the details section.  It MAY have two parts: "{componentName}:{measurementName}". Component name is a human-readable string. Measurement name is the name of the measurement type (possible values: `utilization`, `responseTime`, `connections`, `uptime`) e.g mongodb-courses:connections
  - componentId.  It is a unique identifier of an instance of a specific sub-component/dependency of a service.
  - observedValue. Optional. could be any valid JSON value, such as: string, number, object, array or literal
  - observedUnit. Required if observedValue is present. Clarifies the unit of measurement in which observedUnit is reported.
  - status. Optional. Indicates whether the component status is acceptable or not.
  - time. Optional.  It is the date-time, in [ISO8601](https://en.wikipedia.org/wiki/ISO_8601) format, at which the reading of the observedValue was recorded.
  - output. Optional. Component's raw error output, in case of "fail" or "warn" states.  This field should be omitted for "pass" state.

### Health Check Response: Example

```
 GET /health HTTP/1.1
     Host: api.component.product.com
     Accept: application/health+json

     HTTP/1.1 200 OK
     Content-Type: application/health+json
     Cache-Control: max-age=3600
     Connection: close

      {
         "status":"pass",
         "version":"1",
         "output":"",
         "service":"ec.services.abc",
         "description":"health for ABC service",
         "instance":"api-host-0002",
         "checks":{
            "cassandra:responseTime":[
               {
                  "componentId":"node-0001",
                  "componentType":"datastore",
                  "observedValue":250,
                  "observedUnit":"ms",
                  "status":"pass",
                  "time":"2018-01-17T03:36:48Z",
                  "output":""
               }
            ],
            "cpu:utilization":[
               {
                  "componentId":"api-host-0002",
                  "componentType":"system",
                  "observedValue":85,
                  "observedUnit":"percent",
                  "status":"warn",
                  "time":"2018-01-17T03:36:48Z",
                  "output":""
               }
            ],
            "memory:utilization":[
               {
                  "componentId":"api-host-002",
                  "node":1,
                  "componentType":"system",
                  "observedValue":8.5,
                  "observedUnit":"GiB",
                  "status":"warn",
                  "time":"2018-01-17T03:36:48Z",
                  "output":""
               }
            ]
         }
      }
```

## Monitoring Policy

New APIs should be shipped with monitoring capabilities.

One thing is shipping new API end-points with an optimal level of performance and other is keeping it working that way. For this reason, we need to keep an eye on our API calls and see how they are performing. We need to start tracking response time, identifying slowness and kill them. This will ensure our APIs continue working as we expect as the time passes.
