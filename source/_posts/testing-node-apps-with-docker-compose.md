---
layout: post
title: testing-node-apps-with-docker-compose (and some Soul)
date: 2016-11-18 16:40:22
author: fmaffei
tags: [Docker, Node-js, TDD, Compose]
---

## Contents of this post

* [Purpose](#purpose): the reason for this blog post.
* [Scenario](#scenario): what this example of using docker-compose can be useful for.
* [Prerequisites](#prerequisites): basic setup to be able to run the code contained in this post.
* [Code example](#example): an actual step-by-step guide on how you can setup your test environment to run with docker-compose.
* [Improvements](#improvements): a couple of ideas on how to expand this technique.

## <a name="purpose"></a>Purpose

As I am sure the audience of this post knows to some extent, [Docker](https://www.docker.com/what-docker) is a technology that has grown to become popular over the last few years, allowing developers to deploy pieces of software by packaging them into standardized containers, in a number of various ecosystems (Apache Mesos, Amazon Web Services and many more).

So we can use Docker for our deployment needs, awesome. But let’s pay attention to a key word I used above. Docker grants _isolation_. And what do we like to perform on our application in isolation? Yeah, you guessed right &ndash; testing!

Specifically, with this post, I aim to dig deeper into how to use [docker-compose](https://docs.docker.com/compose/) (a specific Docker-based tool that enables creation of multi-container Docker applications) to build and run a Node.js application connected to MongoDB, to test their interaction and the interaction of the app with the external world, all inside containers running on your machine. All isolated and testable thanks to the usage of containers that we can spin up, hit with tests, and clean up with little effort. 

Interested? Let’s go!

## <a name="scenario"></a>Scenario

In this scenario we will use Docker and one of its functionalities, docker-compose, to build a container and spin up our app.  Then we build another container with a copy of the database where we can freely create and manipulate data, and finally we perform all the integration testing we want against those self-contained entities, which we can clean up after the tests ran. Total isolation and, very importantly, no need to pollute our development or pre-production environment with superfluous test data.

Let’s imagine an app that we can build and test, for example a directory of soul music artists.

Then let’s scope what our app needs to do, and how to test it. Our purpose is to:

* Test that when we hit the `/` path we get a 200 response and a basic home page.
* Test that we can post a payload against the `/artist` path, to create one entry in our database (let's say the great Marvin Gaye).
* Test that when we hit the `/artist/marvingaye` we get the artist page with its name

## <a name="prerequisites"></a>Prerequisites

Before diving into the prototype we should make sure everything is setup correctly. Requirements:

* Node.js v4 and npm
* [Docker v1.12](https://docs.docker.com/docker-for-mac/) (assuming you are on OSX you can use Docker For Mac)
* [Docker Compose](https://docs.docker.com/compose/install/)
* The [Mocha](https://mochajs.org/#installation) testing framework

## <a name="example"></a>Code example

### Step 1: Scaffolding ###

Let’s build as little as we can without testing.

This first step is not the most crucial one for this blog post’s sake, so I will not delve into it too much. [I have pushed everything into a Github repo](https://github.com/federicomaffei/soul-compose), so you will be able to see how the code should look like at each step.

We will use the [hapi.js](http://hapijs.com/) framework to create a basic server application and [here is the package.json](https://github.com/federicomaffei/soul-compose/blob/78400f40606c032fb01542d35e579dc851d82fb3/package.json) file of the Soul Compose app with all the dependencies you need to get started. Copy it in your home folder and run:

```
npm install
```

Now let’s go ahead and create an [index.js](https://github.com/federicomaffei/soul-compose/blob/78400f40606c032fb01542d35e579dc851d82fb3/index.js) file which will host our server. This is the only thing I will not test, it just comes out of the box with Hapi.js.

```javascript
const Hapi = require('hapi');

const server = new Hapi.Server();

server.connection({
    port: 3000
});

server.start((err) => {
    if (err) {
        throw err;
    }
    console.log('Server running at:', server.info.uri);
});
```

##### [Code example at this point.](https://github.com/federicomaffei/soul-compose/tree/78400f40606c032fb01542d35e579dc851d82fb3)

### Step 2: Test the / path ###

Now we can add something interesting: a failing test where we try to hit the / path of the app and expect to get a status code 200 and some text. Let’s write it using Mocha syntax:

```javascript
beforeEach((done) => {
    request.get(`http://localhost:3000`, (error, res, b) => {
        response = res;
        body = b;
        done();
    });
});

it('returns a 200 and a message', () => {
    expect(response.statusCode).to.equal(200);
    expect(body).to.equal('Funky soul singers');
});
```

It obviously fails, because we have no handler for that route and the app is not running. Let’s add a little code to fix this:

``` javascript
server.route({
    method: 'GET',
    path: '/',
    handler: (req, reply) => {
        reply('Funky soul singers')
    }
});
```

And then start the app, before running tests again:

```
npm start
```

Tests green!

##### [Code example at this point.](https://github.com/federicomaffei/soul-compose/tree/70f945e8e79e7fb62f4dda37cecbd2611f6630ea)

### Step 3: Dockerise the app ###

There is already something we could improve here. We are hitting the ‘real’ app with our request, but this is not what we call isolation, right?

Now is a good time to pull Docker in. We can use docker-compose to build a container with our app that runs on port 3001, nice and separated from our development one. Let’s do it.

All we need to do is create a Dockerfile in the root folder:

```
FROM node:4-onbuild
EXPOSE 3000
ENTRYPOINT ["/usr/local/bin/node", "index.js"]
```

Let’s take a look at what this set of instructions mean. We are building a container with our app, taking the base image from the Node.js official image (the environment where the app will run), exposing the port that the app will use for serving requests, and running the server at last. This, by the way, is a pretty standard way of using a Node.js within Docker.

Now, to spin up the app and hit it with tests, let’s use docker-compose. This will come in useful later, when we will add another container (the database) linked with our app. For now it will run a single container. To do it, all we need to do is create a [docker-compose.yml](https://github.com/federicomaffei/soul-compose/blob/faca99d02178407f3ffa9bfd932bcb8932dfd2e0/docker-compose.yml) file in our root folder:

``` yml
version: '2'
services:
  soul-compose:
    build:
      context: .
    ports:
      - "3001:3000"
```

This is a standard way of adding a service to a docker-compose configuration file. We are declaring that the context of the container is the top folder (where the Dockerfile that will be used to build it lives), and mapping port 3000 of the container to port 3001 of our local environment. This will allow the container to run in parallel with the app running locally, without the risk of having port allocation issues.
Now, let’s build and run our one-container composition:

```
docker-compose up -d
```

The interesting detail here is the -d option: it basically allows the container to run detached from the command line (in background mode, if you wish). This means we can stay on the same terminal and just run the tests again.

And (making sure we changed our tests to make requests to port 3001) our single test should pass! Now on to bigger things.

##### [Code example at this point.](https://github.com/federicomaffei/soul-compose/tree/faca99d02178407f3ffa9bfd932bcb8932dfd2e0)

### Step 4: New artist (throw Mongo into the mix)! ###

So the first acceptance test passes. Now, let’s test a route that allows us to create an entry for a new artist, and test that if we hit an endpoint called `/artist` with a payload, we get a 200 code from the route. It won’t actually create it for now, but it will give us a path to do the actual creation later.

``` javascript
beforeEach((done) => {

    const options = {
        method: 'POST',
        uri: 'http://localhost:3001/artist',
        body: { name: 'Marvin Gaye', id: 'marvingaye' },
        json: true
    }


    request(options, (error, res, b) => {
        response = res;
        body = b;
        done();
    });
});

it('returns a 200 and a message', () => {
    expect(response.statusCode).to.equal(200);
    expect(body).to.equal('Created a soul singer named Marvin Gaye');
});
```

Let’s go ahead and make it green, but without actually creating the artist.

``` javascript
server.route({
    method: 'POST',
    path: '/artist',
    handler: (req, reply) => {
        reply(`Created a soul singer named ${req.payload.name}`).code(200);
    }
});
```

In the next test we can dive into the thick of it: let’s say that we now want to get a nice, shiny page for the singer we just created, reading it from a data persistence system (we'll use MongoDB):

``` javascript
beforeEach((done) => {
    request.get(`http://localhost:3001/artist/marvingaye`, (error, res, b) => {
        response = res;
        body = b;
        done();
    });
});

it('returns a 200 and an artist page', () => {
    expect(response.statusCode).to.equal(200);
    expect(body).to.equal('Marvin Gaye');
});
```

This acceptance test will give me the chance to show how we can hook up another Docker image ([MongoDB](https://hub.docker.com/_/mongo/)) to docker-compose, and run the test doing a write (the POST) and a read (a GET) from that DB instance.

I am going to add some boilerplate code that serves the purpose of having a connection to the db and perform reads and writes ([find it here](https://github.com/federicomaffei/soul-compose/blob/bfe5d89c907cc2dc55acfbd4a763fdc56f9763b8/index.js)). The important detail is the fact that now it is actually used in the endpoints we are testing. We are ready to re-run our tests.

##### [Code example at this point.](https://github.com/federicomaffei/soul-compose/tree/bfe5d89c907cc2dc55acfbd4a763fdc56f9763b8)

The tests will fail, the reason being that our app is unable to connect to the database in its Docker environment at this point. So let’s take advantage of the features of docker-compose, and add a fully-featured MongoDB instance ready to be used, adding it to our [docker-compose.yml](https://github.com/federicomaffei/soul-compose/blob/2962d28e33366a92801f0ef2c18ef3dc7dd9f9db/docker-compose.yml) file.

Let’s go through the code. First and foremost, we added an entry for the MongoDB image to be built and run:

``` yml
mongodb:
  image: mongo:3.0.11
  ports:
    - "27018:27017"
  command: --smallfiles
```

To make sure that the app and MongoDB load up in the right order (mongo, then app) we can also specify a *depends_on* property, meaning the app will wait for Mongo to start, and then will be able to access it through the hostname *mongodb*.

``` yml
soul-compose:
  build:
    context: .
  ports:
    - "3001:3000"
  environment:
    - NODE_ENV=test
  depends_on:
    - mongodb
```

This means we will also have to change the code in the app to take that into account. One way to do it is by exporting an environment variable in the application container using docker-compose (as you can see above), and set the MongoDB hostname depending on it [in our index.js file](https://github.com/federicomaffei/soul-compose/blob/2962d28e33366a92801f0ef2c18ef3dc7dd9f9db/index.js). A small but important change.

``` javascript
const mongoHost = process.env.NODE_ENV === 'test' ? 'mongodb' : 'localhost';
```

And this covers what we had in scope. You should be able to re-run the tests to watch them going green. Pat yourself on the back!

##### [Code example at this point.](https://github.com/federicomaffei/soul-compose/tree/2962d28e33366a92801f0ef2c18ef3dc7dd9f9db)

## <a name="improvements"></a>Improvements

Let's wrap up with some improvements we could make to the current state of the application.

An useful exercise would be creating a [script file](https://github.com/federicomaffei/soul-compose/blob/fcb8b653988672f16670c0c65db599b0c8fb2580/scripts/run-tests.sh) to perform all the commands to build the docker-compose images, run the tests, and perform a [cleanup of the test images and containers](https://github.com/federicomaffei/soul-compose/blob/master/scripts/run-tests.sh#L10).

And then slightly modify the *npm test* command in package.json to run the script:

``` json
"test": "./scripts/run-tests.sh"
```

##### [Code example at this point.](https://github.com/federicomaffei/soul-compose/tree/fcb8b653988672f16670c0c65db599b0c8fb2580)

Other ideas for improvement:

* Modify the Node.js Dockerfile to use suggested [best practices](https://github.com/nodejs/docker-node/blob/master/docs/BestPractices.md).

* Pull in more dependencies your app might have into the docker-compose.yml, like [Redis](https://hub.docker.com/_/redis/).

* Change from using the *onbuild* Node.js image to a more customized Dockerfile that does not run npm install at every build but [instead caches modules](http://bitjudo.com/blog/2014/03/13/building-efficient-dockerfiles-node-dot-js/) if the *package.json* file has not changed, dramatically reducing execution time of the tests.

Thanks for reading!

-----
Thanks to [Stefano Ricciardi](https://twitter.com/stefanoric) for the proof-reading and feedback.
