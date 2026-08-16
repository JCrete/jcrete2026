# Simple JSON and Immutable Trees

When I publish material for one online platform or another, I often need to process some JSON files that describe some deployment details. Make that a lot of JSON files. Often, the platform designers start out with a naïve scenario and don't think how that scales to book-length material. Then scripting becomes essential.

That's why I am looking forward to [JEP 540: Simple JSON API](https://openjdk.org/jeps/540).

Why have a JSON library in the core API at all? Perhaps the Java team can use it to replace internal ad-hoc JSON usage? Or perhaps they like the scripting use case? Jackson is too heavyweight for dragging it into the core API or simple scripts.

What makes Jackson heavyweight? Streaming and data binding. JEP 540 doesn't deal with either. It is just a simple API for reading and creating JSON trees.

Interestingly, the [API](https://cr.openjdk.org/~naoto/json/javadoc/api/jdk.incubator.json/jdk/incubator/json/package-summary.html) does not use a sealed family of records that would allow for pattern matching and deconstruction. There is simply a sealed interface JsonValue with subinterfaces JsonString, JsonNumber, JsonBoolean, JsonNull, JsonObject, and JsonArray. If you know the structure of your document, you can traverse a path like this:

```
JsonValue root = Json.parse("chapter4.json");
String title = root.get("courses").get(0).get("title").asString();
```

To analyze an arbitrary node, call

```
Map<String, JsonValue> children = root.asMap();
```

Creating a JSON document is a bit tedious since you need to first put values into lists and maps, and wrap primitives. For example:

```
JsonObject root = JsonObject.of(Map.of(
    "title", JsonString.of("Control Structures"),
    "description", JsonString.of(chapterDescription),
    "courses", JsonArray.of(List.of(
        JsonObject.of(Map.of(
            "course_id", JsonNumber.of(id1),
            "title", JsonString.of("Expressions, Statements, Semicolons"),
            "description", JsonString.of(section1Description))),
        ...
        ))
    ));
```

Note that you cannot build up a JSON object incrementally like you can in Jackson. In this API, JSON values are _immutable_.

That makes it difficult to transform JSON objects. I often need to read a JSON file and then produce one that is similar to the original. In Jackson, I just edit the objects. But with an immutable value, I need to copy the entire structure except for the parts that change.

I have a fair amount of experience doing that in Scala, rewriting immutable XML trees. The Scala XML library has a rudimentary [RuleTransformer](https://scala-lang.org/api/2.12.8/scala-xml/scala/xml/transform/RuleTransformer.html) that I found insufficient. I wrote a few helper methods that, if I adapted them to JSON, could look like this: 

```
JsonObject transform(BiPredicate<JsonValue, Path> condition, Function<JsonValue, JsonValue> mapper)
JsonObject remove(BiPredicate<JsonValue, Path> condition)
```

Here `Path` describes the path from the root to the value, with each path element being a map key or list index. (There is no such thing in JEP 540.)

We also noted that the [Java Class File API](https://openjdk.org/jeps/484) has transformers for transforming code sequences, methods, or classes.

We discussed whether there is a more generic way of describing these tree transforms, and someone mentioned [Haskell lenses](https://hackage.haskell.org/package/lens). Not sure that's the Java way, though. 

Now is the time to kick the tires on this API and let the folks at Oracle know what's not working for you!


