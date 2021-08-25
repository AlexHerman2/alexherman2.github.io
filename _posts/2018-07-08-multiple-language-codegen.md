---
title: "Generating SDKs from a Swagger Spec for Multiple Languages"
layout: post
---

## Outline
 - Introduction
   - Why would we do this?
     - To make it easier for consumers of your API to get started
     - To support polyglot development
   - What we're going to cover
     - How to set up a gradle file to generate Java and Ruby SDKs
     - Creating wrapper code in Java and Ruby
     - Writing a unit test in JUnit/RSpec

## What we're going to cover

## Prerequisites
 - A Java development environment set up (We're not using anything super cutting edge so, Java 7 or above should be fine)
 - Ruby installed

There was a time when I thought that the way to be a good developer was to pick 1 language and learn everything about it: warts and merits, trivia and idioms, toolchains and ecosystem. I thought that the best developers knew everything about "their" language and to reach their level, you should ignore everything but "your" language.

Then I met someone who thought the same way, but was a couple decades deeper in his career than I was. Its out that following this path you don't become a great developer, you become "the such-and-such guy". And the person I met had become "the Visual Basic guy". He knew all the ins and outs of his tools and had made interesting things. But, when something truly new needed to be built... no one called the "Visual Basic guy". His job was to fix the bugs in things that were too important to let break, but not important enough to invest in. I decided to revisit my thinking. Maybe it's better to be proficient at a lot of things, instead of an absolute expert in 1.


An API is, fundamentally, a contract between two machines to allow them to communicate with each other. Language, library, protocol: anything is on the table as long as both parties know what’s going on. 

With that in mind, I’ve found that often it’s best to define the API contract first, before writing any code at all. Until you get into the nitty-gritty details of paths, payloads, and methods, it’s highly unlikely that team writing the API and the team consuming the API are on the same page. Beyond that, thinking about error responses early in the design process gets us off the happy path and considering edge cases.

And, if things go south, at least you can point to the API contract for some level of plausible deniability.

## Enter Swagger
Swagger, er, OpenAPI now I guess, is a way to define an API (including paths, payloads, methods, parameters…the whole shebang). This approach allows the APIs to be defined in such detail that they can be tested, mocked, or used to generate code.

## Get a Swagger Editor spun up
There a a bunch of online Swagger editors, but I prefer to have a local instance running (for the illusion of control). Easiest way to do that is with the Swagger Editor Docker image.

``` bash
docker run -p 8080:8080 swaggerapi/swagger-editor
```

Be patient while it downloads a bunch of layers. Once it’s good to go head over to `localhost:8080` in your browser. You should see the Swagger Editor running with the Pet Store example definition loaded in the editor.

## Define your API
For this example, we’re going to be defining a simple demo API that provides a full set of CRUD operations for a Magic Eight Ball style fortune teller. Why bother with something like this? Well, I figured it would be straight forward enough as to be easy to adapt to whatever you’re working on but still comprehensive enough to be meaningful. Also, I’m planning on actually making this API and using it for a few other demos so, win-win.

First off, let’s set some basic information about our entire API:
``` yaml
swagger: "2.0"
info:
  description: "An example API that provides a magic eight ball style fortune."
  version: "1.0.0"
  title: "A Magic Eightball"
 
basePath: "/eight-ball"
```

The first line defines which version of the Swagger spec we’re going to use. I’m using 2.0 here even though it’s not the most recent version because, eventually I want to use this to generate code via the Swagger Codegen project and (for now, at least), it only supports the 2.0 spec. We’re also defining some basic information about our API, version, title, description, and base path. Base path is, essentially just a prefix in front of all the paths we define later on.

### Define the Model Schemas

One of the coolest parts of Swagger is the ability to reuse payload schema definitions. In this example, we’ll define 2 payloads: a `NewFortune` that includes wraps the fortune text, and a `FortuneResponse` that is returned by our GET request and includes both the fortune text and the ID of the fortune itself.
``` yaml
definitions:
  FortuneResponse:
    type: "object"
    properties:
      id:
        type: "integer"
      text:
        type: "string"
  NewFortune:
    type: "object"
    properties:
      text: 
        type: "string"
```

### Define the End Points that Don’t Include ID in the Path
In this example API, we’re going to have two end points that don’t have the fortune ID as part of the path (that is, the path is just `/eight-ball`): A POST end point to add new fortunes, and a GET that returns a random fortune (like, when you shake a magic eight-ball).

``` yaml
paths:
  /:
    get:
      tags:
        - "magic-eight-ball"
      summary: "Gets a random fortune"
      operationId: "getRandomFortune"
      produces:
        - "application/json"
      responses:
        200:
          description: "A fortune"
          schema: 
            $ref: "#/definitions/FortuneResponse"
    post:
      tags:
        - "magic-eight-ball"
      summary: "Create a new fortune"
      operationId: "createFortune"
      consumes:
        - "application/json"
      parameters:
        - name: "fortune"
          in: "body"
          description: "Fortune to be added"
          required: true
          schema:
            $ref: "#/definitions/NewFortune"
      responses:
        200:
          description: "New fortune created"
```

Here we’re defining the path (in this case `/`), and two methods (GET and POST). We’ve tagged each method with `magic-eight-ball` (just to make it look nicer in the Swagger UI), added a summary describing what the end point does and an operationId that CodeGen will use to name the methods it creates (but, we don’t have to worry about that right now).

Let’s look at our GET method a bit. We’ve defined the response payload type as `application/json`. And, we’ve defined the 200 response: a `FortuneResponse` object that we’ve defined earlier (that’s what that `$ref` bit is doing).

We’re doing something similar for the POST method, but instead of defining the response, we’re defining the request payload. We do this in the `parameters` block, where a `required` payload in `body` of the request should match the schema of the `NewFortune` we defined earlier. We’ve also defined a 200 response, but in this case we don’t return a payload.

### Define the End Points that Include ID in the Path
And finally, we need to add the two end points that include the ID in the path: a PUT to update an existing fortune, and a DELETE to, well, delete it. Both of these are defined with the same `paths` block as the previous 2 end points

``` yaml
/{id}:
    parameters:
      - name: "id"
        in: "path"
        description: "ID of fortune"
        required: true
        type: "integer"
        format: "int64"
    put:
      tags:
        - "magic-eight-ball"
      summary: "Update an existing fortune"
      operationId: "updateFortune"
      consumes:
        - "application/json"
      parameters:
        - name: "fortune"
          in: "body"
          description: "Updated fortune data"
          required: true
          schema:
            $ref: "#/definitions/NewFortune"
      responses:
        200:
          description: "Existing fortune updated"
        404:
          description: "ID not found"
    delete:
      tags:
        - "magic-eight-ball"
      summary: "Delete an existing fortune"
      operationId: "deleteFortune"
      responses:
        200:
          description: "Fortune deleted"
        404:
          description: "ID not found"
```

Since both end points require the id as a path parameter, we defined that right off the bat. Notice how this is similar to how the body payload was defined above, but the main difference is that the `in` field is set to `path` here and the `name` field matches the name of the path parameter on the first line (`id`). This will apply the path parameter definition to both of the following end points.

The PUT end point takes the same request body as the POST method, except in this case it will update the fortune text instead of creating a new entry in our database. But, what happens if the ID provided isn’t available in the database? Well, in that case we should return a 404. Notice how we’ve defined two distinct response code: 200 and 404 in the end point.

And finally, the DELETE end point. This takes no request payload and returns no response payload, it either succeeds (with a 200) or it can’t find the fortune to delete (giving a 404). Easy peasy.

### The Whole Thing
Putting it all together, we end up with a nice, easily readable document that clearly defines what our API expects and what our API makes available. It’s concrete enough to allow a team building an app that consumes our API to begin their work in parallel with ours, but not so concrete that it locks us in to any one language or implementation.

``` yaml
swagger: "2.0"
info:
  description: "An example API that provides a magic eight ball style fortune."
  version: "1.0.0"
  title: "A Magic Eightball"
 
basePath: "/eight-ball"

paths:
  /:
    get:
      tags:
        - "magic-eight-ball"
      summary: "Gets a random fortune"
      operationId: "getRandomFortune"
      produces:
      - "application/json"
      responses:
        200:
          description: "A fortune"
          schema: 
            $ref: "#/definitions/FortuneResponse"
    post:
      tags:
        - "magic-eight-ball"
      summary: "Create a new fortune"
      operationId: "createFortune"
      consumes:
        - "application/json"
      parameters:
        - name: "fortune"
          in: "body"
          description: "Fortune to be added"
          required: true
          schema:
            $ref: "#/definitions/NewFortune"
      responses:
        200:
          description: "New fortune created"
  /{id}:
    parameters:
      - name: "id"
        in: "path"
        description: "ID of fortune"
        required: true
        type: "integer"
        format: "int64"
    put:
      tags:
        - "magic-eight-ball"
      summary: "Update an existing fortune"
      operationId: "updateFortune"
      consumes:
        - "application/json"
      parameters:
        - name: "fortune"
          in: "body"
          description: "Updated fortune data"
          required: true
          schema:
            $ref: "#/definitions/NewFortune"
      responses:
        200:
          description: "Existing fortune updated"
        404:
          description: "ID not found"
    delete:
      tags:
        - "magic-eight-ball"
      summary: "Delete an existing fortune"
      operationId: "deleteFortune"
      responses:
        200:
          description: "Fortune deleted"
        404:
          description: "ID not found"

definitions:
  FortuneResponse:
    type: "object"
    properties:
      id:
        type: "integer"
      text:
        type: "string"
  NewFortune:
    type: "object"
    properties:
      text: 
        type: "string"
```
