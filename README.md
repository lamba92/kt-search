[![](https://jitpack.io/v/jillesvangurp/es-kotlin-wrapper-client.svg)](https://jitpack.io/#jillesvangurp/es-kotlin-wrapper-client)
[![Actions Status](https://github.com/jillesvangurp/es-kotlin-wrapper-client/workflows/CI-gradle-build/badge.svg)](https://github.com/jillesvangurp/es-kotlin-wrapper-client/actions)


# Introduction

ES Kotlin Wrapper client for the Elasticsearch Highlevel REST client is a client library written in Kotlin that adapts the official [Highlevel Elasticsearch HTTP client for Java](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high.html) (introduced with Elasticsearch 6.x) with some Kotlin specific goodness. 

The highlevel elasticsearch client is written in Java and provides access to essentially everything in the REST API that Elasticsearch exposes. The kotlin wrapper takes away none of that and adds some power and convenience to it.

## Why?

The Java client is designed for Java users and comes with a lot of things that are a bit awkward in Kotlin. This client cuts down on the boiler plate and uses Kotlin's DSL features, extension functions, etc. to layer a friendly API over the underlying client functionality. 

Kotlin also has support for co-routines and we use this to make using the asynchronous methods in the Java client a lot nicer to use. Basics for this are in place but Kotlin's co-routine support is still evolving and some of the things we use are still labeled experimental.

## Code generation

As of 1.0-M1, this library makes use of code generated using my [code generation gradle plugin](https://github.com/jillesvangurp/es-kotlin-codegen-plugin) for gradle. This uses reflection to generate extension functions for a lot of the Java SDK. E.g. all asynchronous functions gain a co-routine friendly variant this way. Future versions of this plugin will add more Kotlin specific stuff. One obvious thing I'm considering is generating kotlin DSL extension functions for the builders in the java sdk. Builders are a Java thing and this would be a nice thing to have. 

Ideas welcome ...

## Platform support

This client requires Java 8 or higher (same JVM requirements as Elasticsearch). Some of the Kotlin functionality is also usable by Java developers (with some restrictions). However, you will probably want to use this from Kotlin. Android is currently not supported as the minimum requirements for the highlevel client are Java 8. Besides, embedding a fat library like that on Android is probably a bad idea and you should probably not be talking to Elasticsearch directly from a mobile phone in any case.

# Documentation

- [demo project](https://github.com/jillesvangurp/es-kotlin-demo) - This demo project was used for a presentation and includes a some nice examples. I try to keep this in sync with the latest release. If you need a quick overview, start there.
- Check the code samples on this page. I try to keep these up to date but it's a manual thing currently and they are not part of any build process currently. This is something I'm investigating some options for.
- The tests test most of the important features and should be fairly readable and provide a good overview of how to use things. I like keeping the tests somewhat readable.
- [dokka api docs](https://htmlpreview.github.io/?https://github.com/jillesvangurp/es-kotlin-wrapper-client/blob/master/docs/es-kotlin-wrapper-client/index.html) - API documentation - this gets regenerated for each release and should be up to date.
- [Elasticsearch java client documentation](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high.html) - All of the functionality provided by the java client is supported. All this kotlin wrapper does is add stuff. Elasticsearch has awesome documentation for this.

# Get it

We are using jitpack for releases currently; the nice thing is all I need to do is tag the release in Git and they do the rest. They have nice instructions for setting up your gradle or pom file:

[![](https://jitpack.io/v/jillesvangurp/es-kotlin-wrapper-client.svg)](https://jitpack.io/#jillesvangurp/es-kotlin-wrapper-client)

See [release notes](https://github.com/jillesvangurp/es-kotlin-wrapper-client/releases) with each github release tag for an overview what changed.

Note. this client assumes you are using this with Elasticsearch 7.x. The versions listed in our build.gradle and docker-compose file are what we test with. Usually we update to the latest version within days after Elasticsearch releases.

For version 6.x, check the es-6.7.x branch.

# Examples/highlights

The examples below are not the whole story. Please look at the tests for more details on how to use this and for working examples. I try to add tests for all the features along with lots of inline documentation. Also keeping markdown in sync with code is a bit of a PITA, so beware minor differences.

## Initialization

```kotlin
// we need the official client but we can create it with a new extension function
val esClient = RestHighLevelClient()

```

Or specify some optional parameters

```kotlin
// great if you need to interact with ES Cloud ...
val esClient = RestHighLevelClient(host="domain.com",port=9999,https=true,user="thedude",password="lebowski")
```

Or pass in the builder and rest client as you would normally.

**Everything** you do normally with the Java client works exactly as it does normally. But, we've added lots of extra extension functions to make things easier. Mostly these are in kotlin files with packages that match the classes they are extending. So, using a class from the java sdk means it it's in your imports already, which in turn makes the extension functions available for you to use.

## DAOs and serialization/deserialization

A key feature in this library is using Data Access Objects or DAOs to abstract the business of using indices (and the soon to be removed types). Indices store JSON that conforms to your mapping. Presumably, you want to serialize and deserialize that JSON. DAOs take care of this and more for you.

You create a DAO per index:

```kotlin
// Lets use a simple Model class as an example
data class TestModel(var message: String)

// we use the refresh API in tests; you have to opt in to that being allowed explicitly 
// since you should not use that in production code.
val dao = esClient.crudDao<TestModel>("myindex", refreshAllowed = true,
modelReaderAndWriter = JacksonModelReaderAndWriter(
                TestModel::class,
                ObjectMapper().findAndRegisterModules()
))
```

The `modelReaderAndWriter` parameter takes care of serializing/deserializing. In this case we are using a default implementation that uses Jackson and we are using the kotlin extension for that, which registers itself via `findAndRegisterModules`. 

Typically, most applications that use jackson, would want to inject their own custom object mappers. It's of course very straightforward to use alternatives like GSon or whatever else you prefer. In the code below, we assume Jackson is used.

## CRUD with Entities

Managing documents in Elasticsearch basically means doing CRUD operations. The DAO supports this and makes it painless to manipulate documents.

```kotlin
// creates a document or fails if it already exists
// note you should probably apply a mapping to your 
// index before you index something ...
dao.index("xxx", TestModel("Hello World"))

// if you want to do an upsert, you can 
// do that with a index and setting create=false
dao.index("xxx", TestModel("Hello World"), create=false)
val testModel = dao.get("xxx")

// updates with conflict handling and optimistic locking
// you apply a lambda against an original 
// that is fetched from the index using get()
dao.update("xxx") { original -> 
  original.message = "Bye World"
}
// in case of a version conflict, update 
// retries. The default is 2 times 
// but you can override this.
// version conflicts happen when you
// have concurrent updates to the same document
dao.update("xxx", maxUpdateTries=10) { original -> 
  original.message = "Bye World"
}

// delete by id
dao.delete("xxx")
```
See [Crud Tests](https://github.com/jillesvangurp/es-kotlin-wrapper-client/blob/master/src/test/kotlin/io/inbot/eskotlinwrapper/IndexDAOTest.kt) for more.

## Bulk Indexing using the Bulk DSL

The Bulk indexing API in Elasticsearch is complicated and it requires a bit of boiler plate to use and more boiler plate to use responsibly. This is made trivially easy with a `BulkIndexingSession` that abstracts all the hard stuff and a DSL that drives that:

```kotlin
// creates a BulkIndexingSession and uses it 
dao.bulk {
  // lets fill our index
  for (i in 0..1000000) {
    index("doc_$i", TestModel("Hi again for the $i'th time"))
  }
}
// when the block exits, the last 
// bulkRequest is send. 
// The BulkIndexingSession is AutoClosable.
```
See [Bulk Indexing Tests](https://github.com/jillesvangurp/es-kotlin-wrapper-client/blob/master/src/test/kotlin/io/inbot/eskotlinwrapper/BulkIndexingSessionTest.kt) for more. 

There are many features covered there including per item callbacks to deal with success/failure per bulk item (rather than per page), optimistic locking for updates (using a callback), setting the bulk request page size, etc. You can tune this a lot but the defaults should be fine for simple usecases.

## Search

For search, the wrapper offers several useful features that make make constructing search queries and working with the response easier.

Given an index with some documents:

```kotlin
dao.bulk {
    index(randomId(), TestModel("the quick brown emu"))
    index(randomId(), TestModel("the quick brown fox"))
    index(randomId(), TestModel("the quick brown giraffe"))
    index(randomId(), TestModel("the quick brown horse"))
    index(randomId(), TestModel("lorem ipsum"))
    // throw in some more documents
    for(i in 0..100) {
        index(randomId(), TestModel("another document $i"))
    }
}
dao.refresh()
```

... lets do some searches. We want to find matching TestModel instances:

```kotlin
// our dao has a search short cut that does all the right things
val results = dao.search {
  // inside the block this is now the searchRequest, the index is already set correctly
  // you can use it as you would normally. Here we simply use the query DSL in the high level client.
  val query = SearchSourceBuilder.searchSource()
      .size(20)
      .query(BoolQueryBuilder().must(MatchQueryBuilder("message", "quick")))
  // set the query as the source on the search request
  source(query)
}

// SeachResults wraps the original SearchResponse and adds some features
// e.g. we put totalHits at the top level for convenience. 
assert(results.totalHits).isEqualTo(3L)

// results.searchHits is a Kotlin Sequence
results.searchHits.forEach { searchHit ->
    // the SearchHits from the elasticsearch client
}

// mappedHits maps the source using jackson, done lazily because we use a Kotlin sequence
results.mappedHits.forEach {
    // and we deserialized the results
    assert(it.message).contains("quick")
}
// or if you need both the original and mapped result, you can
results.hits.forEach {(searchHit,mapped) ->
    assert(mapped.message).contains("quick")
}
```

## Raw Json queries and multi line strings

The wrapper adds a few strategic extension methods. One of those takes care of the boilerplate you need to turn strings or streams into a valid query source.

So, the same search also works with multiline json strings in Kotlin and you don't have to jump through hoops to use raw json. Multiline strings are awesome in Kotlin and you can even inject variables and expressions.

```kotlin
val keyword="quick"
val results = dao.search {
    source("""
    {
      "size": 20,
      "query": {
        "match": {
          "message": "$keyword"
        }
      }
    }
    """)
}
```

In the same way you can also use `InputStream` and `Reader`. Together with some templating, this may be easier to deal with than programatically constructing complex queries via  builders. Up to you. I'm aware of some projects attempting a Kotlin query DSL and may end up adding support for that or something similar as well.

See [Search Tests](https://github.com/jillesvangurp/es-kotlin-wrapper-client/blob/master/src/test/kotlin/io/inbot/eskotlinwrapper/SearchTest.kt) for more.

## Scrolling searches made easy

Scrolling is kind of tedious in Elastisearch. You have to keep track of scroll ids and get pages of results one by one. This requires boilerplate. We use Kotlin sequences to solve this. Sequences are lazy, so this won't run out of memory and you can safely stream process huge indiceses. Very nice in combination with bulk indexing.  

The Kotlin API is exactly the same as a normal paged search (see above). But please note that things like ranking don't work with scrolling searches and there are other subtle differences on the Elasticsearch side. 

```kotlin
// We can scroll through everything, all you need to do is set scrolling to true
// you can optionally override scrollTtlInMinutes, default is 1 minute
val results = dao.search(scrolling=true) {
  val query = SearchSourceBuilder.searchSource()
      .size(7) // lets use weird page size
      .query(matchAllQuery())
  source(query)
}

// Note: using the sequence causes the client to request for pages. 
// If you want to use the sequence again, you have to do another search.
results.mappedHits.forEach {
    println(it.message)
}
```


The `ScrollingSearchResults` implementation that is returned takes care of fetching all the pages, clearing the scrollId at the end, and of course mapping the hits to TestModel. You can only do this once of course since we don't keep the whole result set in memory and the scrollids are invalidated as you use them.

See [Search Tests](https://github.com/jillesvangurp/es-kotlin-wrapper-client/blob/master/src/test/kotlin/io/inbot/eskotlinwrapper/SearchTest.kt) for more.

## Async search with co-routines

The high level client supports asynchronous IO with a lot of async methods that take an `ActionListener`. Probably the most common use case is searching and for that we provide a convenient short hand:

```kotlin
val keyword="quick"
// you may want some more appropriate scope than global scope ...
runBlocking {
  val results = dao.searchAsync {
    source("""
      {
        "size": 20,
        "query": {
          "match": {
            "message": "$keyword"
          }
        }
      }
    """)
  }
}
```

One potential usecase for this could be e.g. doing several queries at once in response to a user request to give the user results for several different indices, each with their own query.

## Async Bulk with co-routines

We also have an asynchronous bulk indexer that internally depends on the experimental `Flow` and `Channel` APIs in Kotlin. However, you don't need to know anything about how that works to use this. For the user it works the same way as the synchronous bulk index api above. To use this, you will need to add the `kotlin.Experimental` flag to your build and to your runtime.

```kotlin
val successes = mutableListOf<Any>()
runBlocking {
    val totalItems = 10000
    dao.bulkAsync(
        bulkSize = 200,
        refreshPolicy = WriteRequest.RefreshPolicy.NONE,
        bulkDispatcher = newFixedThreadPoolContext(10, "test-dispatcher"),
        itemCallback = { operation, response ->
            if (response.isFailed) {
                println(response.failureMessage)
            } else {
                // this only gets called if ES reports back with a success response
                successes.add(operation)
            }
        }
    ) {
        (0 until totalItems).forEach {
            index(randomId(), TestModel("object $it"))
        }
    }
    // ES has confirmed we have the exact number of items that we bulk indexed
    assertThat(successes).hasSize(totalItems)
}
```

The call to `bulkAsync` creates an asynchronous BulkIndexingSession. That creats a `Channel` based `Flow` that processes the bulk operations in a separate co-routine. Calls to `index`, `update`, and `delete`, push operations in the channel. The Flow groups the operations in chuncks of `bulkSize` and sends them off to elastic search with more co-routines . Just like with the synchrounous bulk index API we use a callback to process responses from elasticsearch and you group the bulk operations in a block.

The `Flow API` still has a few question marks around it regarding e.g. parallelism and we may change how this works internally. However, it is shaping up as a really nice alternative to manually orchestrating producer and consumer type logic.  Ideally we'd fire bulk requests on different threads in parallel and use back pressure to avoid overloading ES with requests. However, the current behavior seems to be strictly sequential. There seems to be a lot of discussion around this topic still in the [Kotlin issue tracker](https://github.com/Kotlin/kotlinx.coroutines/issues/1147). I will update the internals accordingly when this changes.

## A few notes on Co-routines

Note, co-routines are a **work in progress** and things may change as Kotlin evolves and as my understanding evolves. This is all relatively new in Kotlin and there is still a lot of stuff happening around e.g. the `Flow` and `Channel` concepts.

The `RestHighLevelClient` exposes asynchronous variants of all API calls as an alternative to the synchronous ones. The main difference is that these use the asynchronous http client instead of the synchronous one. Additionally they use callbacks to provide a response or an error. We provide a `SuspendingActionListener` that addapts this to Kotlin's co-routines.

For example, here is the async search [extension function](https://github.com/jillesvangurp/es-kotlin-wrapper-client/blob/master/src/main/kotlin/org/elasticsearch/client/KotlinExtensions.kt) we add to the client:
```
suspend fun RestHighLevelClient.searchAsync(
    requestOptions: RequestOptions = RequestOptions.DEFAULT,
    block: SearchRequest.() -> Unit
): SearchResponse {
    val searchRequest = SearchRequest()
    block.invoke(searchRequest)
    return suspending {
        this.searchAsync(searchRequest, requestOptions, it)
    }
}
```

The `suspending` call here creates a [`SuspendingActionListener`](https://github.com/jillesvangurp/es-kotlin-wrapper-client/blob/master/src/main/kotlin/io/inbot/eskotlinwrapper/SuspendingActionListener.kt) and passes it as `it` to the lambda. Inside the lambda we simply pass that into the searchAsync call. There are more than a hundred async methods in the RestHighLevel client and we currently don't cover most of them but you can easily adapt them yourself by doing something similar.

# Building

You need java >= 8 and docker + docker compose (to run elasticsearch and the tests).

Simply use the gradle wrapper to build things:

```
./gradlew build
```
Look inside the build file for comments and documentation.

If you create a pull request, please also regenerate the documentation with gradle dokka. We host the [API documentation](docs) inside the repo as github flavored markdown.

Gradle will spin up Elasticsearch using the docker compose plugin for gradle and then run the tests against that. If you want to run the tests from your IDE, just use `docker-compose up -d` to start ES. The tests expect to find that on a non standard port of `9999`. This is to avoid accidentally running tests against a real cluster and making a mess there (I learned that lesson a long time ago).

# Development status

This library should be perfectly fine for general use at this point. Also note, that you can always access the underlying Java client, which is stable. However, until we release 1.0, refactoring can still happen ocassionally.

Currently the main blockers for a 1.0 are:

- ES 7.5.x should include my pull request to enable suspendCancellableCoRoutine
- We recently added reflection based code generation that scans the sdk and adds some useful extension functions. More feature work here is coming.
- We depend on a few things related to flow and co-routines that still require annotations like `@InternalCoroutinesApi`. Ideally we get rid of needing that anywhere before 1.0.
- Currently asynchronous is optional and I want to make this the default way of using the library as this can be so nice with co-routines and should be the default in a modern web server. 

If you want to contribute, please file tickets, create PRs, etc. For bigger work, please communicate before hand before committing a lot of your time. I'm just inventing this as I go. Let me know what you think.

## Compatibility

The general goal is to keep this client in sync with the current stable version of Elasticsearch. We rely on the most recent 7.x version. From experience, this mostly works fine against any 6.x node with the exception of some changes in APIs or query DSL; and possibly some older versions. Likewise, forward compatibility is generally not a big deal barring major changes such as the removal of types in v7.

For version 6.x, you can use the es-6.7.x branch or use one of the older releases (<= 0.9.11). Obviously this lacks a lot of the recent feature work.


# License

This project is licensed under the [MIT license](LICENSE). This maximizes everybody's freedom to do what needs doing. Please exercise your rights under this license in any way you feel is appropriate. Forking is allowed and encouraged. I do appreciate attribution and pull requests ...
