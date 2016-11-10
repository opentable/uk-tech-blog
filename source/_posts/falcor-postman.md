---
layout: post
title: falcor-postman
date: 2016-11-09 16:40:22
author: mrichetto
tags: [Falcor, JavaScript, OSS]
---

At OpenTable, we have an engineering culture that empowers us to research, experiment and learn. 


In an effort to foster innovation and to try new ideas, [Chris Cartlidge](/blog/authors/ccartlidge.html), [Nick Balestra](https://twitter.com/nickbalestra), [Tom Martin](https://twitter.com/tpgmartin) and [myself](/blog/authors/mrichetto.html) started to work on a side project nicknamed big-mars. Our project is a mobile-first, responsive web application that uses Falcor by Netflix.


Falcor, a JavaScript library for efficient data fetching, is an implementation of the Backend for Frontend (BFF) or the API Gateway pattern.


One powerful concept that Falcor has is its query model, in which you access your data as if it was a single JSON model in memory.  You can navigate your data structure the way you would navigate a JSON object; either with a “dot notation” (e.g. restaurant.address.city) or with an “array notation” (e.g. restaurant[‘address’][‘city’]). 


Really easy and intuitive.


However, while experimenting and learning how to properly use its query model, we noticed that it was quite hard to “visualise” the expected result from the API calls without using, for instance, our beloved in-browser Developer Tools console.

## falcor-postman
The Falcor project released many additional packages in order to facilitate developers who are willing to use this technology (e.g. falcor-express) but in this ecosystem we noticed a lack of tools with a GUI on top.


We also noticed that the GraphQL, another project that shares with Falcor the same core concepts, has tools with visual interfaces (e.g. express-graphql and GraphiQL).


So as a spin-off of our big-mars project, Nick and I decided to build a tool with a nice and intuitive GUI responsible for exercising the Falcor endpoint in order to help us to validate our queries, and we named this tool [falcor-postman](https://github.com/opentable/falcor-postman).


So we decided to build falcor-postman as an Express.js middleware, making it easy to plug into an existing Falcor project that uses Express.js as a web application framework. We are releasing it as an Open Source Software project so that others can take advantage of it and hopefully speed up their own development lifecycle.

## The features
When plugged into an existing Express.js application it will be possible to access a specific configurable route, showing a web page in which you can write Falcor queries and send them to your Falcor endpoint. Then the result of the query will be presented.


It will be also possible to resend previous queries and modify them, as they will be saved for you into your browser's local storage.

{% img center /images/posts/falcor-postman.png %}

## Under the hood
Falcor-postman is composed by an Express.js middleware responsible for serving the UI which is a React.js application. Webpack is our module bundler. We take advantage of hot reloading in development mode while for the production release we create a physical file containing the bundle.


In addition we are using [Pure.css](http://purecss.io) and [Codemirror](https://codemirror.net) that are helping us with the UI, and we are also using eslint-config-opentable (another OpenTable open source project) as our .eslintrc config.

## What’s next?
falcor-postman v2.0.0 has just been released on npm and we’re awaiting feedback. Also, we have some issues in our GitHub repository which we will use to build our own product roadmap.


In the short term we would like to fix some outstanding minor issues that we are aware of (polishing the UI, enhancing the query history) but in the long term we would also like to enable data update; at the moment, our middleware only enables fetches but we’d like to better reflect the full functionality of Falcor, which allows for updates as well.

## One final comment
At OpenTable we love OSS and we truly believe that sharing knowledge and experiences is one of the best ways for learning and growing as engineers. We highly encourage you to reach out to us if you would like to discuss your experience with us or simply would like to understand better what we do.


## Links and resources
* [falcor-postman GitHub repository](https://github.com/opentable/falcor-postman)
* [Falcor website](https://netflix.github.io/falcor/)


