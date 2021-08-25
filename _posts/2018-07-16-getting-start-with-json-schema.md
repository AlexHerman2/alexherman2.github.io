---
layout: post
title: "Getting Started with JSON Schema"
---

# Getting Started with JSON Schema
When I use API Blueprint to define an API contract, I’ve found the JSON Schema tends to be more expressive than the MSON format more deeply embedded into the spec.  What follows is a quick tutorial outlining how to define a complex JSON payload using JSON Schema.

Let’s say we have a reasonably complex JSON payload. Something like this:
```json
{
  "name": "Example",
  "date_created": "2018-07-01T01:12:23Z",
  "url": "https://example.com/users?id=12345",
  "permissions": [
    "CAN_READ",
    "CAN_WRITE"
  ],
  "profile": {
    "display_name": "Alex",
    "favorite_food": "pizza"
  }
}
```

So, what we have here is a JSON payload with an object, an array, a string, a date formatted string, and a url formatted string. All in all, this is a pretty complex payload. Let’s start with the simpler items and then work our way out to the more complex ones.

First, let’s get the boilerplate out of the way. We need to identify which version of the JSON Schema spec we’ll be working against. So start with:
```json
{
  "$schema": "http://json-schema.org/draft-04/schema#"
}
```
And, we know that our root is an object, so let’s add that:
```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {

  },
  "required": [],
  "additionalProperties": false
}

```
So, what did we just add here? We’ve carved out a place for us to define all the fields in our root object (`properties`), a place for us to define which fields are required, and we’ve let the Schema parser know that any properties outside the ones we’ve defined (well, are going to define) are not allowed.

So far, so good. Now, let’s start with the three string fields. The first one can be any string, the second must be a date-time, and the last one a url. To do this, we’ll add these properties to the root object.
```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "name": {
      "type": "string"
    },
    "date_created": {
      "type": "string",
      "format": "date-time"
    },
    "url": {
      "type": "string",
          "format": "uri"
    }
  },
  "required": ["name"],
  "additionalProperties": false
}
```
So, what we’ve done here is call out all three string fields. In JSON-Schema every field definition is an object, but simpler fields have simpler definition objects (not surprisingly). Notice how our more constrained fields have the additional “format” property. JSON-Schema spec has a number of string formatters available, but “date-time” and “uri” are the ones I end up using most often.

Also, note that “name” was added to the required array. What this indicates is that any field defined in the “properties” object is optional _except_ name. Since this is an array, we can add as many required fields as, um, required.

Let’s get a little deeper and add the “permissions” array.
```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "name": {
      "type": "string"
    },
    "date_created": {
      "type": "string",
      "format": "date-time"
    },
    "url": {
      "type": "string",
          "format": "uri"
    },
    "permissions": {
      "type": "array",
      "items": {
        "type": "string",
        "enum": ["CAN_READ", "CAN_WRITE", "IS_ADMIN"]
      },
      "minItems": 1
    }
  },
  "required": ["name"],
  "additionalProperties": false
}
```

So, now we’re starting to get more interesting. We’ve defined the “permissions” property in the same way we’ve defined the other properties. This time, however, the type is “array” and there’s an additional field “items”. In this “items” field we can define the data type inside the array. In our case it’s just simple strings, but we can define integers, objects, other arrays, whatever we need. We’re starting to see the cool, recursive nature of JSON-Schema here. Objects can be nested inside of other objects, collected into arrays, or flattened into whatever we want and the syntax largely stays the same.

Notice how we’ve limited the possible values of the strings within “permissions” array. This essentially makes it an array of enums. We’ve also specified the minimum number of elements in the array (1 in this case) but we could leave it unbounded, or add a maximum number of elements depending on the situation we find ourselves in.

Now, if the elements inside the “items” block reflect the elements in the root object, it stands to reason that a nested object definition would look similar to the root object definition. In fact, it does. Let’s take a look.
```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "name": {
      "type": "string"
    },
    "date_created": {
      "type": "string",
      "format": "date-time"
    },
    "url": {
      "type": "string",
          "format": "uri"
    },
    "permissions": {
      "type": "array",
      "items": {
        "type": "string",
        "enum": ["CAN_READ", "CAN_WRITE", "IS_ADMIN"]
      },
      "minItems": 1
    },
    "profile": {
      "type": "object",
      "properties": {
        "display_name": {
          "type": "string"
        },
        "favorite_food": {
          "type": "string"
        }
      },
      "required": ["display_name", "favorite_food"],
      "additionalProperties": false
    }
  },
  "required": ["name"],
  "additionalProperties": false
}
```
And there we have it: we’ve added the type of “object”, the properties of “display_name” and “favorite_food” (both strings and both required), and prevented any additional fields from being allowed. Notice how similar this block is to the root block. This is how JSON-Schema definitions tend to look in the wild, similar blocks composed of similar blocks. All in all, it can look a little intimidating right off the bat, but in reality it’s just simple things being combined together to make more complex things.
