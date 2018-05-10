---
layout: post
title: "Monads explained by a imperative programmer - with examples"
date: 2018-05-10 16:00:00 +0100
author: dmistry
tags: [Monads, Functional programming, Design patterns]
---

A monad is any type construct that follows a specific pattern; it works in the same way as other design patterns. The .NET `ApplicationContext` is a `singleton` in the same way that `Array` is a `Monad`. Something being a monad does not define its purpose; in the same way that a `class` can implement many interfaces, a monadic type can also conform to many patterns to increase its usefulness.

Monads are hard to get our heads around because they are very generic. They are defined by having two methods: the `bind` method (also commonly called `fmap`), and the `unit` method (also commonly known as `pure` or `from`).

## A `bind`/`fmap` method.
The fmap method follows this signature:
```typescript
// Given a monadic type TMonad with value type V1,
// and a method that takes a value of type V1 and 
// returns the same monadic type TMonad but with 
// a value type of V2, produce a monadic type 
// TMonad with value V2.
function (TMonad<V1> m, Func<V1, TMonad<V2>> f) => TMonad<V2>
```

A quick breakdown of the method signature in plain english: _given a monadic type `TMonad` with value type `V1`, followed by a method that takes a value of type `V1` and returns the **same** monadic type `TMonad` but with a value type of `V2`, produce a monadic type `TMonad` with value `V2`._

From the signature, it appears that we should give the value V1 from the first monad to the function `f` and simply return the TMonad from that call. The ability to control the usage of that function and the ability to intercept the return value is where it gets interesting.

The resulting monad will usually be a composition of the original monad, and the new monad produced by `f`. That means the "state" held by the two monads are `merged` and the specifics of how is up to the developer.

_The monadic type `TMonad` cannot change after a call to `f`; a `Maybe` monad cannot change to a `State` monad, but this also means that a `Maybe` monad can never read the value of a `State` monad by being bound to it. This is something I understood later when I started to question how to use different monads together._

_The value type `V1` is free to change as a result of calling `f`. This is why we denote the final result as `TMonad<V2>`; this doesn't prevent the value staying the same, in the same way that in a program, `x` can equal `1` and `y` can also equal `1`, `V1` could represent a `boolean` and `V2` could also represent a `boolean`._

## A `unit`/`pure`/`from` method
The from method follows this signature:
```typescript
// Given a value of type T, return a monadic value TMonad
// that holds the provided value.
function (T value) => TMonad<T>
```

This should be quite simple. You take a single value and store it in a way that makes sense for your monad.

## The monadic laws
Yes laws; it turns out that function signatures aren't enough to describe this pattern and there needs to be some laws that are satisfied in order for a monadic type to be called a `Monad`. These laws ensure that all monads compose the same way and in some functional languages, they enable some pretty cool functions and operators that just work for all monadic types. FREE FUNCTIONALITY!!

### Left Identity
`from x >>= f ≡ f x`

Given a monad of `x` created with the `from` method, when bound to the function `f`, the resulting monad should be identical to simply providing `x` to the function `f`.

Keep in mind that `f` will return a Monad, the monad that `f` produces should be no different than the monad that is a composition of what `from` produced, and what `f` produces.
```ts 
from(x).fmap(fn);
// is the same as:
fn(x);
// given that fn returns a monadic value of the same return type of from.

[10].fmap(x => posAndNeg(x)); //=[10, -10];
posAndNeg(10); //=[10, -10];
```

### Right Identity
`m >>= from ≡ m`

Given a monad `m`, when bound to the function `from`, the resulting monad should be identical to `m`. This is the same law written in reverse and is designed to ensure that the merge logic works bi-directionally.
```ts 
monad.fmap(from);
// is the same as:
monad;

// x => [x] is basically the from function for arrays.
[1,2,3,4].fmap(x => [x]); // [1,2,3,4]
```

### Associativity
`(m >>= f) >>= g ≡ m >>= (\x -> f x >>= g)`

This one is a little harder to stomach, but it enforces the same rules we saw above, but this time from the point of view that you are working with monads that are more complex than the `from` monad.

```ts 
const AF = A.fmap(v => {const F = fnF(v); return F;});
const AFG = AF.fmap(v => {const G = fnG(v); return G;});
// -----
const AFG = A.fmap(x => {
    const F = fnF(x);
    const FG = F.fmap(fnG);
    return FG;
})
```
The two code examples above create `AFG` but the merging of the monads happen in opposite orders. The first example creates merges `A` to `F` (the monad produced from `fnF`) and then `AF` is merged with `G` to give `AFG`. The second example takes the value from `A`, creates `F`, merges it with `G` (implicitly by fmapping it to `fnG`) to produce `FG` and then finally merges `A` with `FG`.

The point is that it shouldn't matter in what order these monads are created or merged to each other, `AFG` in both cases should be identical.

_Caveat: if we were working with immutable types, the two monads would be identical, however if not, then the only difference between the two monads should be where they are in memory. All of the internal values should be identical._

```ts 
[1,2].fmap(posAndNeg).fmap(selfAndDouble) // =[1, 2 ,-1, -2, 2, 4, -2, -4]
[1,2].fmap(x => posAndNeg(x).fmap(selfAndDouble)) // =[1, 2, -1, -2, 2, 4, -2, -4]
```

## Oh, by the way
If you were wondering about `map`, you can always create it using `fmap` and `from`, watch this:
```ts 
function map(monad, fn) {
    return monad.fmap(x => from(fn(x)));
}
```

If you have read about functors, this is significant because this is proof that any monad, is also a functor. If you haven't read about functors, rest assured that it is not important to understand for this article.

## Examples!
### List
The list monad is quite interesting because to understand it, you have to limit your view of it to a subset of its functionality. Remember you can do a lot with lists:
 - You can count the items inside
 - You can `slice`, `reduce`, etc.

From a monadic point of view, a `List` represents `indeterminism`; a set of possible values of type T. The simplest of its type is a list of one value. Let's make that value `D4`. It represents a position on a chess board.

Now we have a function that can calculate, given a single position, all the possible positions that a knight could end up in after one move. That is an array of up to eight possible values (less if we started close to the edge of the board).

Now we want to know all the positions after two moves. We can take each of the single positions from the first calculation and call the same function again. Now we have eight lists!

This is where the merge comes to play. The merge happened in the first instance too, but it's more obvious here. The basic list monad will concatenate the lists. Each operation can actually change the type T to say T2, but the resulting list will then have all resulting values with the same type T2. At least in a typed language. `¯\_(ツ)_/¯`

To see an example of what a List monad does internally, and how code would look without it, have a look at [listComposition.js](https://github.com/dipunm/monads/blob/master/listComposition.js)

_One thing to keep note of, is that the value stored within a monad should not be restricted. That is, we should not expect any special type of value. In Haskell, `Set` cannot be a monad because a `Set` requires that its values are orderable and therefore must implement the interface `Ord`. That being said, I have read about `restricted monads` which is a concept that exists in order to work around this problem. This is beyond the scope of my knowledge, but I wonder if it applies in Javascript where everything can be compared?_

### Maybe
By now we have determined that one of the things that a `Monad` can do, is control how many `values` of type `T` it wants to hold. Well, that value can also become 0. More on this soon...

The `Maybe` monad has two factory methods: `Just(x)` which takes a parameter `x`, and `Nothing()` which takes no parameters. It can hold either one value, or zero values. Even though `Nothing` takes no values, it still represents a `Monad`; it's a monad of 0 values of type `T`. You may need to let that sink in a bit.

So how does the `Maybe` monad merge? Just like the array, we take each value we have and send it to the `f` function. For each monad we get back, we merge them. Fortunately, we can only ever get back a monad with one or less values, so we never have to worry about merging a maybe with multiple values. The maybe monad does this:

- If the new monad is `Just x` we just return it.
- If the new monad is `Nothing` we still just return it.

The moment we have `Nothing`, any function `f` provided to the `fmap` function becomes uncallable; we have no value to give to it. At this point, we just return a new `Nothing` that returns a nothing of type `T2` where `TMonad<T2>` represents the type that `f` would have returned (assuming we are in a typed language).

That is the mechanics of a `Maybe`; but what is it useful for? Due to its nature, it is very good at taking a sequence of computations, each that could fail to produce a value and `fail-fast` or terminate early as soon as it gets a `Nothing`. None of the following `f` in `fmap(f)` are ever executed which can save computation time.

Here is an example of a typical use of the `Maybe` monad:

```ts 
const x = "hello micheal";
const specialLetter = Maybe.from(x)
    .fmap(removeHelloOrReturnNothing) // gives us Just("micheal").
    .fmap(get10thChar) // there is no 10th character, return Nothing().

// note: the use of Maybe.from here allows us to take a non monadic value 
// and convert it into a monadic value to be merged by the fmap method.
// this helps us to avoid re-writing simple methods into Maybe specific
// methods unnecessarily.
    .fmap(x => Maybe.from(uppercase(x))); // nothing to uppercase.


if(specialLetter.hasValue) {
    console.log('success!', specialLetter.Value);
} else {
    console.log('oops, something went wrong.');
}
```

Notice that at the end, we use two properties: `hasValue` and `Value`. These are not part of its monadic properties in the same way that `length` is not a part of the monadic properties of `List`. They are, however very useful.

In Haskell, there is a convenience syntax called the `do` notation. This allows us to write our code in a somewhat procedural looking way. The closest thing to this in Javascript is the generator function syntax. I won't go into detail about what we need to do to get this code working in a real application, but I will show what monadic code could look like.

```ts 
function* getSpecialLetter(greetingMessage) {
    const name = yield removeHelloOrReturnNothing(greetingMessage);
    const char = yield get10thChar(name);
    const specialValue = uppercase(char);
    return Maybe.from(specialValue);
}

const letter = _do(getSpecialLetter,['hello micheal']);
if (letter.hasValue) {
  ...
  ...
}
```

Pretty simple, standard looking code. Hopefully, it should by now be clear that the `Maybe` monad is allowing us to write our sequence of operations without worrying about the edge case of Nothing being returned. It will `fails-fast™`. Once the sequence is complete, you usually need to take care of the `Nothing` by executing a backup plan, and by possibly logging an error. In the case it didn't fail, usually the `Maybe` type has done its job and the resulting internal value is what you're really after.

### Either a
The `Either` monad is interesting. Monads must only represent one value type. `Either` is known to represent two value types.

Either as a type has two useful properties called: `Left` and `Right` and a `isLeft()` and/or `isRight()` method for convenience. The properties are named generically because it generally is used for holding **one value** of any two types; the property that the value exists in depends on the type. 

When used as a monadic type, it is usually used for error detection. The type of `Left` becomes the error type, and the type of `Right` becomes the value type. Technically, `Either` is not monadic, `Either<T1>` is: 

```ts 
// Even though we don't provide a Left value, we still 
// need to specify its type.
const monad1 = Either.Right<Error, String>("hello");

// Likewise for the Right value. 
// Here we have a monad that stores 0 strings.
const monad1 = Either.Left<Error, String>(new Error("Mayday! Mayday!"));
```

As a monadic type, `Either` is very similar to `Maybe`. It can hold either one or zero values, but in this case, instead of only knowing that there was an error, we have the ability to provide some supporting data that can help us make more effective error handling decisions.

```ts
function divide(x, y) {
    if (y === 0) {
        return Either.Left<string, int>("cannot divide ${x} by zero");
    }
    if (x % y !== 0) {
        return Either.Left<string, int>("${y} does not divide into ${x} equally");
    }
    return Either.Right<string, int>(x/y)
}

const total = divide(20, 2).fmap(n => divide(n, 2));

if(total.isLeft()) {
    doSomethingAwesome(total.Left);
} else {
    console.log(total.Right);
    doSomethingElse();
}
```

### Future
First of all, this is not a `Promise`, even though it looks and feels like one; the main difference is that this type is lazy and `Promises` are eager.

This one is quite interesting too! Okay, they're all vastly different, which is why it can be hard to identify the pattern and give `Monads` a good description. But what if I told you that this monad never holds a value, and yet always produces a new monadic value using the function `f`?

This monad never holds a value, this is true, but just like the `Either` monad, it has an internal state; it holds a function that takes no value, and returns a value of type `A`. (We can call this a `Thunk`.. because mathematicians.)

_I won't talk about how the types compose too much here, because it gets pretty confusing as it is and types actually make it harder to wrap your head around it. :(_

The simplest `thunk` we can make is something like: `() => "hello"`; you call it, and you get a value. The return value of the thunk is the type `A` that was mentioned just above. The point of a future is that it is lazy; it won't try to execute any thunk until you explicitly tell it to execute. Just like `Either` has an `isLeft()` method, a `Future` would expose a `run()` method and calling this, will cause the thunk to execute and the final value to be revealed. This operation would be a thread blocking operation if called directly in a thread handling language; in `Haskell`, the closest monad of its type is the `IO` monad and the compiler will handle the execution for you provided that your `main` function returns an `IO` monad.

A more complicated thunk could be something like this:
```ts
makeThunk(filename) {
    return () => {
        // pseudo OS level code here:
        const handle = OS.readFile(filename);
        const data = OS.waitFor(handle); 
        // let's pretend that the OS knows that this 
        // routine should pause, hybernate and continue
        // when the file has been read. (instead of thread blocking)

        return data.contents;
    }
}
```

This is an example of what an asynchronous thunk may look like. It works with the underlying OS to do something that takes time without blocking a CPU thread. This is typically internal code that an application developer wouldn't write, we would instead write something like `Future.readFile('readme.txt')` and just like magic, we end up with the above thunk.

I intentionally embedded this thunk inside a factory method to show how a thunk could depend on a value even though it takes no parameters.

```ts
const monad = Future.from('readme.txt')
    .fmap(Future.readFile)
    .fmap(contents => Future.from(getFirst20Chars(contents)));
```

Hopefully by now we understand the snippet above and what it does. `monad` will not contain the final value, but it will have a `Run` method that can eventually provide that value. What happens inside the fmap function is one of the things that blew my mind. This monad will contain a thunk that looks a bit like this:
```ts 
() => {
    const contents = (
        () => {
            const filename = (
                () => {
                    return "readme.txt";
                }
            )();
            return readFile(filename);
        }
    )();
    return getFirst20Chars(contents);
}
```

Each time we fmap, we don't execute the provided function, we build a thunk around it. The last action to be added is the last action to be executed and its return value will match the final monads value type.

When executed, the program will recursively call the old thunks until it gets its first returned value,and the final value comes by traversing back up the call stack.

## Let's summarise
 - A monad is any type that has a `from` method and a `fmap` method.
 - The two methods can be given any name, but for us to take full advantage and build agnostic, reusable methods, we should keep them consistent.
 - In order to satisfy these methods, a monad should have some concept of a boxed value.
 - A monad always generalises over a value type. (like `List<T>`)
 - The two methods `from` and `fmap` must follow a set of laws.
 - The `fmap` function should merge `monadA` and `monadB` (the result of calling the provided function) to get a new monad which generalises over the same value type as `monadB`.
 - A `monad` can hold 0 or more of its generalised value type.
 - A `monad` can build an internal state. (building state is not the same as mutating state)
 - Knowing what a `monad` is, does not help us know what a `monad` does.
 - The purpose for generalising `monads` is to enable generic helper functions to be created.
 - In functional languages, they can encapsulate calls to external libraries.
 - A `monad` can have expose or encapsulate as much of their internal state as they want.

# A monad of my own!
In order to demonstrate the purpose of a monad, I decided to recreate a function that I made in a previous company, that I was unable to improve at the time, and see if I could create or use a monad to improve it.

**NOTE:** Before we continue, I would like to clarify that I made a large monad for the sake of solving a specific problem. This is probably considered a bad monad in the same way an OOP developer could create a [god object](https://en.wikipedia.org/wiki/God_object). That being said, it follows the monadic signature and laws, and can probably still help to showcase how monads allow better separation of concerns.

Here is a recreation of the original method in javascript:
```ts 
async function syncDataToSolr(lastUpdated) {
    var query = new Query(lastUpdated);
    const cursor = mongo.dispatch(query);
    const timer = new Timer();
    timer.start();
    let count = 0;
    let completed, error, dateOfLastUpdated;
    try {
        while (await cursor.hasNext()) {
            const data = await cursor.readNext(100);
            const success = await solrClient.sendData(data);
            if (!success) {
                completed = false;
                break;
            } else {
                count += data.length;
                dateOfLastUpdated = data[data.length - 1].updatedDate;
            }
        }
    } catch (err) {
        completed = false;
        error = err;
    } finally {
        completed = completed !== false;
        timer.end();
    }

    return Status({
        count,
        dateOfLastUpdated,
        completed,
        error,
        timeElapsed: timer.elapsed
    });
}

syncDataToSolr(new Date(2018, 5, 21))
    .then(console.log)
    .catch(console.log);
```

This method was built to synchronise data from a MongoDb store to a Solr lucene index. This job was very intensive and could fail for a number of reasons. We needed to be able to track its performance to ensure that there were no problems.
 - We needed to track the duration of each run so that we could adjust its execution frequency.
 - We needed to track the updatedDate of the last product we successfully synced in order to influence the next run.
 - We needed to track the number of products synced so that we could judge its performance.
 - We needed to know about any errors or exceptions that occurred along the way in order to debug.
 - We needed to know if the run completed its job or if it exited early for an unknown reason.

All of this information gathering clutters the original intent of the method:
 - Query mongoDb and get a cursor that can read incrementally.
 - Fetch batches of data to be sent to Solr
 - Post the data to Solr
 - Keep fetching and sending until the cursor reaches the end of the mongo resultset.

**_Spoiler Alert_** This is the same code using a Monad:
```ts 
async function* syncDataToSolr(lastUpdated) {
    var query = new Query(lastUpdated);
    const cursor = mongo.dispatch(query);
    yield Tracker.startTimer();
    while (await cursor.hasNext()) {
        const data = yield await cursor.readNext(100);
        yield await solrClient.sendData(data);
    }
    return Tracker.completeTimer();
}

runMonadic(syncDataToSolr, new Date(2018, 5, 21))
    .then(monad => console.log(monad.getStatus()))
    .catch(err => console.log(err));
```

In order to achieve this, as well as creating a monad that encapsulates the tracking behaviour, I had to extend the `solrClient.sendData` and the `cursor.readNext` methods:

```ts 

export class MonadicCursor extends Cursor {
    constructor(cursor) {
        super(cursor);
    }

    readNext(n) {
        return super.readNext(n)
            .then(data => Tracker.of(data))
            .catch(err => Tracker.Errored(err));
    }
}

class MonadicSolrClient extends SolrClient {
    async sendData(data) {
        const success = await super.sendData();
        if(success) {
            return Tracker.RecordSuccess({
                count: data.length,
                dateOfLastUpdated: data[data.length - 1].updatedDate
            })
            // change the monad value without affecting internal state.
            .fmap(() => Tracker.of(true));
        }else {
            return Tracker.Errored()
                // change the monad value without affecting internal state.
                .fmap(() => Tracker.of(false));
        }
    }
}
```

The `cursor` change is pretty simple, we return the value wrapped in my custom `Tracker` monad, or if there was an error, then we use the `Errored` function to return a new monad in its error state; this state has no value and will fast track the execution like the `Either` monad would.

The `solrClient` change looks more complicated, and that is because by design, the solrClient was built to return a boolean that represented if it was successful instead of throwing an error. Instead of relying on `catch` to split the logic, we use an `if` statement. If successful, we track information about what happened, if not, we indicate Errored without any context (we have none ourselves because any internal exceptions would have been handled).

The final step for both cases is to use the `from` method (here called `of`) to ensure our function returns the correct value inside the `Tracker` monad.

# Promises and Monads in Javascript
**Promises are not `monads`;** you can quote me on that. They have the required methods (namely `resolve` and `then`) and they follow the rules to a certain extent, but they were never created with the intention of being `monadic`. They were created with the intention of easing imperative style asynchronous code. [[source]](https://github.com/promises-aplus/promises-spec/issues/94#issuecomment-366157872)

**Note:** Thanks to [Jon Schapiro](https://github.com/JonSchapiro) who pointed out that since Promises are eager, they may be breaking the associativity law. A good example of this, would be to create two promises, one that writes "hello" onto the page, and the other writes "world". If you wanted to compose it in reverse: `worldPromise.then(() => helloPromise);`, the program will not swap the order of execution because it already executed the promises.

Even so, promises are useful. If we need to or want to create agnostic, reusable code, we can choose to use another library, or to monkey patch promises to make them conform better. That being said, we will also have to monkey patch other non-conforming would-be-monads, and it is unlikely that Javascript will build convenient keywords that could help legibility any time soon.

There are a lot of articles that discuss Promises and their impurity, but one article highlights a few pretty interesting side effects due to having eager execution. [[source]](https://staltz.com/promises-are-not-neutral-enough.html)

**Promises are like `monads`.** Notice how with the `Maybe` monad, where we used the `generator function` syntax, we never do any error handling inside? When using `async/await`, if we are trying to write like for like code, we should be grouping a sequence of asynchronous operations together without any error handling as well. Until we exit the async function and get back a `Promise`, we can't really access the properties and methods of `Promise` directly and so we can't use the `catch` method.

The point is to group a sequence of async operations without thinking about edge cases, and then handle the errors at the end. Consider this:
```ts 
fetch('http://www.my-url.com/endpoint')
    .then(data => fetch('http://internal.com/get/' + data.id))
    .then(person => Promise.delay(20).then(person))
    .catch(err => handle(err))
    .then(....);
```

Notice how there are a bunch of thens before the catch? Using the `async/await` syntax, we would end up with:

```ts 
async function doStuff() {
    const data = await fetch('http://www.my-url.com/endpoint');
    const person = await fetch('http://internal.com/get/' + data.id);
    await Promise.delay(20);
    return person;
}
async function doStuff2(data) {
    //.....
}

const stage1 = doStuff().catch(err => handle(err));
const stage2 = stage1.then(doStuff2);
```

I intentionally did not use `try/catches` because that would be a change in logic, but now we end up separating logic into multiple areas. This type of separation can start to become nonsensical; your error handling moves away which can be a problem if you are used to executing logic and handling errors in one place.

There is another way though:
```ts 
async function doStuff() {
    const person = await (async function() {
        const data = await fetch('http://www.my-url.com/endpoint');
        const person = await fetch('http://internal.com/get/' + data.id);
        await Promise.delay(20);
        return person;
    })().catch(err => {
        return handle(err)
    });
    //... continue operations here using the person variable.
}
doStuff();
```

Embedding functions inside one another is quite normal in Javascript. There are concerns amongst my fellow peer developers that using `try/catch` to handle errors generally causes performance issues. Exceptions should be for exceptional cases; yet many promises reject for valid reasons without exceptions and using the catch method for promises feels less controversial than using a `try/catch` pair. Happily, the level of nesting remains consistent and so do variable scopes (_maybe not for vars_) so this style of code can feel familiar, be easier to reason about, and should be easier to debug than the declaritive style of programming that promise chaining provides.

The biggest downside is that `return` keyword doesn't terminate the function anymore so formatting it to look like a try/catch may cause misunderstandings down the line; only experimentation and time can tell. There could be better ways to avoid using try/catch unnecessarily whilst keeping the style more procedural looking. There are many ways to skin a cat.
