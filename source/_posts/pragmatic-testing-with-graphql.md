---
layout: post
title: "Pragmatic Testing with GraphQL"
date: 2017-06-16 15:41:16
author: rluong
comments: true
tags: [Testing, Engineering, Acceptance tests, API, GraphQL]
---

We’ve been using GraphQL at OpenTable for a little over half a year now.  I won’t go into detail as to why we started using it, but suffice to say, we really enjoyed creating our very first GraphQL endpoint.  It eased a lot of the inconsistencies that we were experiencing with some of our REST-ful services.

This post assumes you have some experience building a GraphQL endpoint.  For those of who you aren’t familiar with it, it feels a lot like having a querying language via an HTTP endpoint.  If you want to try it for yourself, I recommend [GitHub’s endpoint](https://developer.github.com/v4/explorer/).  To get started with GraphQL, [the official documentation](http://graphql.org/learn/) is a perfect place.

## The Problem

Now, let’s get back to writing tests for an endpoint.  When we were creating our first endpoint, we started facing regression testing problems while expanding our schema.  It seemed our existing testing methods were ill-equipped to handle it.

Finding a good solution to this is important because I’ve had such a love--hate affair with testing.  All tests are not created equal and a naive developer would say, “write a test in case something changes”.  This certainly isn’t a good measure.  I’ve also found that just attempting to write tests immediately without forethought can be a costly distraction if you grow your codebase.

## The Approach

So we took a step back and looked at the code we were writing.  Most of the connectors we wrote were fairly anemic.  We didn’t have much logic for our API nor did we want any.  We could have written unit tests for each parts of the schema but mocking portions of the system seemed more work than it was worth.

So where did we start?  I personally enjoy the outside-in approach and started writing a few acceptance tests. So, we wrote a test for each query in the schema that we had.  It would fire an HTTP request and expect a 200 status code and have no errors in the resulting JSON body.  Thanks to GraphQL most errors are handily reported using this mechanism so we would be able to make a decent impact using little work.

{% img left /images/posts/graphql-base-tests.gif %}

As you can see, we found that this would be good enough at the time but eventually found that it would not scale very well.  If you forget to update one of these queries, you could leave out a major section out accidentally.  Being a big fan of automation, we wondered how we could scale this out.

## The Solution

We knew GraphiQL (the web interface for testing GraphQL queries) was using introspection to give us intellisense.  What a great idea: we could leverage introspection in a similar way to discover all the possible queries for a given endpoint.  After doing so, we repeated our previous tests, which was to fire an HTTP request.

So how would you generate queries with parameters?  For example numDice is required for this query:

```graphql
rollDice(numDice: 4, numSides: 6) {
	name
}
```

So, how would we be able to provide a valid parameter to a query that we’ve discovered in this endpoint?  We believe, this is a great opportunity to kill two birds with one stone!  Write an example in your documentation/comments for your query and we’ll re-use it as a test!

```graphql
 type Query {
   # RollDice has four examples
   #
   # Examples:
   # rollDice(numDice: 4, numSides: 2)
   # rollDice( numDice : 40 , numSides:2)
   # rollDice ( numDice: 2, numSides: 299 )
   # rollDice (
   #   numDice:4,
   #   numSides: 2342
   # )
   rollDice(numDice: Int!, numSides: Int): RandomDie
 }
```

From this, all we did is write examples for every single one of our queries and re-run our methodology.

We thought that we wouldn’t be the only ones running into this problem so we decided to open source this tool: [graphql-query-generator](https://github.com/opentable/graphql-query-generator).  It can be used as a CLI or a library (if you need more tweaking).  Feel free to give constructive feedback.  Here is a quick demonstration.

![Demonstration of GraphQL Testing Tool](/images/posts/graphql-tool-test.gif)

If you’re not seeing the above commands and want to run it for yourself.  Just follow these three steps:


```
1. npm i -g graphql-query-generator
2. gql-test http://www.example.com/graphql
3. Have a cup of tea :)
```

… And that’s it!  You have a full suite of acceptance tests running against your own GraphQL endpoint!

Please check out: https://www.npmjs.com/package/graphql-query-generator for more information and updates!

We are always looking for contributors and feedback here: https://github.com/opentable/graphql-query-generator/issues

Finally, if you find it valuable in anyway, perhaps you could throw a star :)

##### Disclaimer: we do have performance and unit tests on top of this :)