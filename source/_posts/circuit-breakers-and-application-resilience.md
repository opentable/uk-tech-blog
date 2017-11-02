---
layout: post
title: "Circuit Breakers and Application Resilience"
date: 2017-11-02 12:00:00 +0000
comments: true
author: dgiddins
tags: [Polly, Application Resilience, Microservices, Error Handling]
---

For most people circuit breakers are a concept that belongs in the world of electricity. They are usually a box of fuses under the stairs that prevents the tangle of extension cords from turning into an unexpected open fireplace behind the TV. But the concept of a circuit breaker is something that we can apply to software and software services. 

Martin Fowler described a software circuit breaker as follows:

> "... a protected function call in a circuit breaker object, which monitors for failures. Once the failures reach a certain threshold, the circuit breaker trips, and all further calls to the circuit breaker return with an error, without the protected call being made at all" 

## Background
OpenTable runs on many microservices that depend on one another to deliver the OpenTable site. These services can have dependencies on data stores such as MongoDB and Elastic Search as well as services such as RabbitMQ and Redis. Sometimes these dependencies can have performance or availability issues; such as when somebody accidentally drops an index on a collection in Mongo. 

>This actually happened, your author accidentally dropped a critical index while working with a DB synchronization tool which resulted in downtime. Hence this article.

When this happens calls to dependencies may fail or take a long time to complete which then causes the dependent service to behave in the same way. Other upstream services can also suffer the same problem. This can be described as a cascading failure.

Ideally what we would like to happen at this point is for our service to recognize what is happening, fail gracefully, and stop calling the service that already can't keep up with the load placed upon it. (As well as alerting us to the problem). This is where a software circuit breaker can help us.

<img src="/images/posts/polly-logo.png" style="float:right;margin:0 0 2rem 2rem;width:20rem;"/>

## Polly to the rescue 

Polly (http://www.thepollyproject.org/) - is a transient fault handling library which includes an implementation of a software circuit breaker (amongst other things which we will come to later). The [documentation](https://github.com/App-vNext/Polly#resilience-policies) for Polly is excellent so I won't try to replicate it here. 

### How we used Polly
As you would expect we wrapped calls to external services in Polly circuit breakers (using the Advanced Circuit Breaker policy) and configured it with an arbitrary failure threshold which we thought would protect downstream services that are overwhelmed. But then we went one step further...

A circuit breaker has state, we wanted to be able to see what that state was as it provides insight into the state of our service. To achieve this, we named every circuit breaker instance according to what it protected (RabbitMQ, external Services or Mongo collection names). We then made that state accessible on an endpoint of our API. This allows us to use our monitoring tools to alert us when one of the breakers Opens, providing us with early warning of failures and making it easier to pin-point the failure.

We created a CircuitBreakerRegistry, initialized as a singleton instance within our DI container, which lets us store a handle to a CircuitBreakerPolicy with its unique name, or retrieve all stored policies and retrieve by name. 

```csharp
public class CircuitBreakerRegistry
    {
        public void StoreCircuitBreaker(CircuitBreakerPolicy policy , string breakerId)
        {
            ...
        }
        
        public ConcurrentDictionary<string, CircuitBreakerPolicy> GetAllRegisteredBreakers()
        {
            ...
        }

        public CircuitBreakerPolicy GetCircuitNamedBreaker(string breakerId)
        {
            ...
        }
    }
```

We then created an API controller that allows us to retrieve a list of all registered Circuit Breakers along with their current state. We configured our monitoring software to check this endpoint and alert when an Open breaker is detected. 

```csharp
public class CircuitBreakerController : ApiController
    {
        [HttpGet]
        public IHttpActionResult ListAllCircuitBreakers()
        {
            List<CircuitBreakerViewModel> breakers = _circuitBreakerRegistry
                .GetAllRegisteredBreakers()
                .Select(breaker => new CircuitBreakerViewModel() {BreakerId = breaker.Key, BreakerState = breaker.Value.CircuitState.ToString()})
                .ToList();
            var viewModel = new CircuitBreakersViewModel() {Breakers = breakers};

            return Content(HttpStatusCode.OK,viewModel, new JsonMediaTypeFormatter());
        }   

        [HttpPut]
        public IHttpActionResult ToggleBreaker([FromUri]string breakerId, [FromBody]string value)
        {
            if (value != "Closed" || value != "Isolated")
                return BadRequest("Specify either 'Isolated' or 'Closed'");

            var breaker = _circuitBreakerRegistry.GetCircuitNamedBreaker(breakerId);
            if (breaker == null)
                NotFound();
                
            if (value == "Isolated")
                breaker.Isolate();
            else
                breaker.Reset();

            return ListAllCircuitBreakers();
        }
    }
  ```

The API also allows us to toggle the state of the breaker between Closed (think automatic) and Isolated (think manual fail-over). This could be useful for several reasons including:
* to test our fall-back strategy
* if we knew a dependency was going to fail or in a failing state (but not triggering the breaker)
* if we knew a dependency had a bad deploy that might take a while to rollback/fix. 

> You might think that Closed is bad and Open is good. It’s the other way around in circuit breakers as a switch that is open in a circuit breaks the circuit. When it is closed the circuit is complete and electricity/data can flow.

## Motivation
I've already mentioned that the reason for my interest in Circuit Breakers and this post was through direct experience of downtime I caused. The truth is I have always been interested in Circuit Breaker technology and application resilience in general, but this incident focused time and attention on the problem. 

Clearly this is a case of "Locking the stable door after the horse has bolted" but that's often the way in engineering. However, something like this might happen again and I would rather put measures in place to deal with it ahead of time. You might argue that given an Open circuit breaker can only fail fast or return an empty result, it makes no difference to the user experience as the data they want to view isn't available either way. This is true to a degree, but a Circuit Breaker isolates only broken aspects of a service allowing the rest to function normally. Failing fast also means that the user experience of other aspects of the site are unaffected which is better than potentially impacting the whole site.

### Insurance... 
...is something everybody buys and hopes to never use. Protecting your application with Circuit Breakers and other application resilience measures is one of those things that you should really do, but hope to never need. There is of course a cost involved in adding such measures and it is up to you to decide whether you think it is worth it. 

## Application Resilience
The Circuit Breaker pattern is only one of many Application Resilience Patterns. If you have already looked at the Polly Project Web site you may have seen that it lists several different Application Resilience policies that it offers. Briefly these are:
* Retry (which is where the Polly name comes from)
* Circuit Breakers (already covered)
* Timeout
* Bulkhead Isolation (think of it as a predictive circuit breaker)
* Cache
* Fall-back 

These are by no means a definitive list of Application Resilience measures but are a good starting point. 

Again, the documentation on the Polly website describe these in detail so I won't repeat it here other than to say these Application Resilience methods are something that you might want to consider using alongside Circuit Breaker policies. We use Retry inside of a Circuit Breaker (using the Policy Wrap) to overcome transient faults. We apply a Timeout policy when calling some 3rd party client libraries that have no built-in timeout mechanism. For instance, a version of RabbitMQ for .NET will block our application start-up if the configured RabbitMQ instance is not running on the target host. A forced timeout ensures that our application fails and informs us of why.

## Other Libraries
Polly is a great library for .NET developers and there are similar libraries are available in other languages. Hysterix (https://github.com/Netflix/Hystrix) is one such tool that you may have already heard of as it was developed by Netflix and is written for Java applications. There is a version of Polly in Javascript called PollyJs (https://github.com/mauricedb/polly-js) that provides retry and there are of course packages that implement Circuit Breakers in Javascript as well.

## Final thoughts
If you decide to use any of these existing frameworks, roll your own or simply implement these patterns directly into your application, the key thing to consider is how you want your application to behave during failure. You should consider how it will affect its upstream clients and how they will react to your application in different states as this is not always as obvious as you might expect. Most importantly, consider the business impact of your failure modes on the customer. You should aim to minimize the impact on the user experience and think about how much you want to tell them when you are offering a reduced service. 
