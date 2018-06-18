---
layout: post
title:  JSON - Create Arrays with Streams
tags: javaee rest
---

This post shows how to create a cool *JavaEE 7* JsonArray using *Java 8* streams.

In the "old" days we had to iterate through a collection of objects with primitive instructions such as `for` and `while` (imperative style).
*Java 8* allows you to build JsonArrays with Stream reduce terminal operation - `collect`.

***

#### Cool JsonArray Builder

```java
    @GET
    @Path("people")
    public Response listPeople() {

        // get people list data
        List<JsonObject> peopleJson = sampleData();

        // using java 8 streams and method references
        JsonArray jsonArray = peopleJson.stream()
                .collect(Json::createArrayBuilder, JsonArrayBuilder::add, JsonArrayBuilder::add)
                .build();

        return Response.ok(jsonArray).build();
    }
```


****


