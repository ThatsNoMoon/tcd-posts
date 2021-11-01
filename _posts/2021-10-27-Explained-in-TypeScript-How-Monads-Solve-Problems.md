---
title: 'Explained in TypeScript: How Monads Solve Problems'
author: ThatsNoMoon
tags: [typescript, functional-programming]
categories: [Write-ups]
---

# Introduction: Callbacks

Imagine, for a moment, that it's 2013, you're involved in designing JavaScript, and you have a problem: callbacks are a mess. Because of the way the web was designed, JavaScript has to use asynchronous programming for I/O like making HTTP requests, but, so far, the only way to handle such async operations is to pass functions into other functions which pass other functions into more functions, on and on ad nauseum:

```js
function callbackHell(params, errCallback, idCallback) {
    makeRequest(params, (err, result) => {
        if (err) {
            return errCallback(err);
        }
        processResults(results, (err, data) => {
            if (err) {
                return errCallback(err);
            }
            sendResults(data, (err, response) => {
                if (err) {
                    return cb(err);
                }
                idCallback(response.id);
            });
        });
    });
}
```

Disgusting! Especially when you consider that if this was synchronous, it would look like:

```js
function synchronyHeaven(params) {
    const results = makeRequest(params);
    const data = processResults(results);
    return sendResults(data).id;
}
```

*(Yes, I'm using ES2015 features, but only because the point of this post isn't to discuss old JavaScript.)*

There are a few things going wrong in our sideways pyramid of doom that callbacks build up:
- Each callback needs to check if the operation failed manually
- Every chained operation creates another indentation level

Where do you go from here?

# How we'll discover monads

Whether you've heard of monads or not, my goal with this post is to show how you might discover monads, how monads solve problems, and what makes a monad a monad. This exploration will work best if you're familiar with TypeScript and callback-based asynchronous programming like the above.

# A dissection of callbacks

So, what's so bad about callbacks? From my earlier example, `callbackHell`, we can see the syntax disadvantages: identation floats to the right the more steps we take, and every step has a check for failure. Where do these problems come from? Let's look at an individual step:

```js
asyncStep(args, (err, result) => {
    if (err) {
        // handle err
    } else {
        // use result
    }
});
```

From a certain perspective, we can see how the next step is made *part* of this step; the callback is, visually, inside the call to `asyncStep`. This means any usage of the results of `asyncStep` have to go inside this step using `asyncStep`. We can try to disconnect these steps a bit, like so:

```js
asyncStep(args, nextStep);

function nextStep(err, result) {
    if (err) {
        // handle err
    } else {
        // use result
    }
}
```

Now the next step isn't inside the first step, but the first step still has to know about the second step, so this still isn't ideal.

Let's look at the other problem: every step that follows a fallible step has to check for failure manually. This is because errors are passed next to the result, so we can try splitting the callback up. I'll use the first callback to handle errors, and the second callback to handle successful results:

```js
asyncStep(args, nextStepErr, nextStepSuccess);

function nextStepSuccess(result) {
    // use result
}

function nextStepErr(err) {
    // handle err
}
```

This looks like it would require twice as many functions, but in many cases, a whole chain of operation uses the same error handling, which reduces the number of functions needed.

Let's try rewriting `callbackHell` with these techniques:

```js
function callbackHell(params, errCallback, idCallback) {
    makeRequest(params, errCallback, processStep);

    function processStep(result) {
        processResults(results, errCallback, sendStep);
    }

    function sendStep(data) {
        sendResults(data, errCallback, finish);
    }

    function finish(response) {
        idCallback(response.id);
    }
}
```

Well, I think this looks neater. Whether you agree or not, you probably agree that it's still worse than the synchronous alternative; there's quite a bit of repetition, between creating and using each step and using the provided `errCallback` in every step.

# Promises

How can we improve on our last version of `callbackHell`? Considering our goal is to make something as nice as our synchronous version, `synchronyHeaven`, a reasonable first step would be to try to make `const result = makeRequest(params);` meaningful, so let's make `makeRequest` return an object. What should this object do?

Since we're replacing a callback argument, how about we give this object methods to add callbacks:

```js
const result = makeRequest(params);

result.onSuccess(processStep);
result.onFailure(errCallback);
```

This is certainly interesting, even if it hasn't made our code shorter. I'll call this object a promise, since it represents a guarantee that something will happen later, either a success or a failure.

Let's look at our `finish` function, and the step before it, translated into our promise style:

```js
function sendStep(data) {
    const response = sendResults(data);
    response.onFailure(errCallback);
    response.onSuccess(finish);
}

function finish(response) {
    idCallback(response.id);
}
```

`finish` is a unique step in our function, as it doesn't do anything asynchronous, so we haven't changed it much yet. I'll examine a possible improvement to `sendStep` by adding a method to our promise type to do a synchronous step first:

```js
function sendStep(data) {
    const response = sendResults(data);
    response.onFailure(errCallback);

    response.map(response => response.id);

    response.onSuccess(idCallback);
}
```

And do a little API manipulation:

```js
function sendStep(data) {
    sendResults(data)
        .onFailure(errCallback);
        .map(response => response.id);
        .onSuccess(idCallback);
}
```

That's pretty neat, right? How about we try adding something similar to `map`, but instead executes asynchronous operations:

```js
// note that we've gone back one step in
// the chain to illustrate this method
function processStep(results) {
    processResults(results)

        .andThen(data => sendResults(data))

        .onFailure(errCallback)
        .map(response => response.id)
        .onSuccess(idCallback);
}
```

Well, that's even better, let's try applying it to the whole function:

```js
function promisePurgatory(params) {
    return makeRequest(params)
        .andThen(processResults)
        .andThen(sendResults)
        .map(response => response.id);
}
```

Nice! Notice that we're able to replace the callbacks in this function as well, and return a promise instead. The ability to chain async operations together means we can even remove `onSuccess`, at least from the public API.

# Arrays

Let's look at another problem: transforming arrays. Say you're making a social media app, and you have an array of a user's friends. You might want to get an array of their usernames so you can easily display them on your website. It's 2013, so you write out a loop to make a new array:

```js
const usernames = [];

for (const user of friends) {
    usernames.push(user.name);
}
```

Or maybe you're displaying a summary of things the user's friends did:

```js
const summary = [];

for (const event of events) {
    summary.push(event.type);
}
```

We could implement a function that handles any array transformations like this fairly easily:

```js
function mapArray(array, func) {
    const transformed = [];

    for (const x of array) {
        result.push(func(x));
    }

    return transformed;
}
```

Now our loops from above can be replaced:

```js
const usernames = mapArray(friends, user => user.name);

const summary = mapArray(events, event => event.type);
```

And since you're helping design JavaScript, you can just make that a convenient method:

```js
const usernames = friends.map(user => user.name);

// ...
```

However, `map` doesn't solve all of our array transformation needs. In our previous example, how would you get an array of things the user's friends did in the first place?

```js
const events = [];

for (const user of friends) {
    for (const event of user.recentEvents) {
        events.push(event);
    }
}
```

This might come up often enough to deserve a method too, for example, you might want an array of all of the people tagged in the user's recent photos, so let's add a `flatMap` method that does the work of the nested loop above:

```js
const events = friends.flatMap(user => user.recentEvents);
```

The name `flatMap` refers to how, if you used just `map`, you would end up with a nested array, but `flatMap` returns a flat array.


# The Patterns

If we compare the solutions we came up with for the above problems, obviously we came up with a `map` method for both `Promise` and `Array`. Upon closer inspection, these `map` methods even have the same signature, besides using `Promise` vs `Array`:

```ts
let promiseMap: <T, U>(this: Promise<T>, func: (t: T) => U) => Promise<U>;

let arrayMap:   <T, U>(this: Array<T>,   func: (t: T) => U) => Array<U>;
```

We've also created two other methods with the same signature, but different names:

```ts
let promiseAndThen:
    <T, U>(this: Promise<T>, func: (t: T) => Promise<U>) => Promise<U>;

let arrayFlatMap:
    <T, U>(this: Array<T>,   func: (t: T) => Array<U>)   => Array<U>;
```

# Functor

One abstraction we've discovered is that of functors: types that can be mapped. A functor is any type `M` that could have a method like:

```ts
<T, U>(this: M<T>, f: (t: T) => U) => M<U>
```

This includes `Array` and `Promise`, as we've seen, as well as `Set`, `[T, T, T]`, `T | undefined` (or a wrapper around it), and many other types.

One consequence of this definition is that all functors have exactly one type argument. This means that `string` can't be a functor, even though you could write a function that iterates over its contents and transforms them.

Formally, a functor also has to satisfy this rule, assuming its method is called `map`:

```ts
m.map(x => x) = m
```

Meaning, `map` should only change `m` in the way the mapping function changes the elements. (And, for clarity, `=` here just means equivalence, not necessarily any specific notion of equality in JavaScript.)

We've already seen the definition of `map` for one functor, `Array`, but I'll show another functor's definition here, that of `T | undefined`, or the wrapper I'll make called `Optional`:

```ts
class Optional<T> {
    #value: T | undefined;

    constructor(value: T | undefined) {
        this.#value = value;
    }

    map<U>(func: (t: T) => U): Optional<U> {
        if (this.#value) {
            return new Optional<U>(func(this.#value));
        } else {
            return new Optional<U>(undefined);
        }
    }

    // other methods...
}

new Optional(4);
// Optional { #value: 4 }

new Optional("hello").map(s => s + ".");
// Optional { #value: "hello." }

new Optional<number>(undefined).map(x => x + 1);
// Optional { #value: undefined }

new Optional({ name: "epsilon" }).map(obj => obj.name);
// Optional { #value: "epsilon" }
```

In summary:
- Functors are types containing values that can be mapped, each to another value
- Functors can have a method `map`: `<T, U>(this: M<T>, f: (t: T) => U) => M<U>`

# Monad

The other pattern we saw is (part of) the monad abstraction! A monad is a type `M` that can have a method like:

```ts
<T, U>(this: M<T>, f: (t: T) => M<U>) => M<U>
```

In our examples, this method sequenced asynchronous actions (`Promise`'s `andThen`), and mapped and flattened nested structure (`Array`'s `flatMap`). The way I think about this method, which I'll call `bind`, is that it sequences actions. For `Promise` this isn't too unclear I hope, `andThen` makes a promise that runs one `Promise`, and then runs another `Promise` by using the result of the first one. For `Array`, this conceptualization still works, but it is somewhat less obvious. Consider a function `T => Array<U>` as an action that can produce multiple values. With that context, `Array`'s `flatMap` sequences actions that create mulitple values, always creating another action that creates multiple values with the results of the last action.

`Optional` from the last section is also a monad, so I'll show its definition as well:

```ts
class Optional<T> {
    #value: T | undefined;

    constructor(value: T | undefined) {
        this.#value = value;
    }

    // Optional's `bind` method
    flatMap<U>(
        func: (t: T) => Optional<U>,
    ): Optional<U> {
        if (this.#value) {
            return func(this.#value);
        } else {
            return new Optional<U>(undefined);
        }
    }

    // other methods...
}

function safeDiv(
    x: number,
    y: number,
): Optional<number> {
    if (y == 0) {
        return new Optional<number>(undefined);
    } else {
        return new Optional(x / y);
    }
}

new Optional("hello"
    .flatMap(s => safeDiv(10, s.length));
// Optional { #value: 2 }

new Optional<number>(undefined)
    .flatMap(x => safeDiv(1024, x));
// Optional { #value: undefined }
```

`bind` for the `Optional` monad can be thought of as sequencing actions that might not produce a value.

I said at the top that the pattern we saw is only part of what monads are, that's just because there's one extra function a monad needs, `unit`.

```ts
<T>(t: T) => M<T>
```

This can be thought of as creating a container with only on element, or creating an action that does nothing but return the provided value.

```ts
const arrayUnit: <T>(t: T) => Array<T>
    = (t) => [t];

const optionalUnit: <T>(t: T) => Optional<T>
    = (t) => new Optional(t);
```

Because we have a `unit` operation, this means that every monad is also a functor:

```ts
const map: <T, U>(m: M<T>, f: (t: T) => U) => M<U>
    = (m, f) => m.bind(x => M.unit(f(x)));
```

Formally, monads also have three rules:
- Right Identity: `unit(x).bind(f) = f(x)`
- Left Identity: `m.bind(unit) = m`
- Associativity: `m.bind(x => f(x).bind(g)) = m.bind(f).bind(g)`

To skip ahead a little, JavaScript's builtin `Promise` type is a functor and a monad in the same way the `Promise` we came up with is, but the names are a little different:
- `map` is called `then`
- `bind` is also called `then`
- `unit` is `Promise.resolve`

In summary:
- Monads are actions that can be sequenced, executed one after the other
- They have a method `bind`: `<T, U>(this: M<T>, f: (t: T) => M<U>) => M<U>`
- They have a function `unit`: `<T>(t: T) => M<T>`

# Building around monads

If we go back to the example we started with, we have async and sync versions of a certain function:

```js
function synchronyHeaven(params) {
    const results = makeRequest(params);
    const data = processResults(results);
    return sendResults(data).id;
}

function promisePurgatory(params) {
    return makeRequest(params)
        .andThen(processResults)
        .andThen(sendResults)
        .map(response => response.id);
}
```

While `promisePurgatory` is certainly nicer than `callbackHell`, it's not quite as nice as `synchronyHeaven`. Since we have an abstraction over sequencing actions, why not make a language feature to make that nicer? We'll create a syntax sugar for `andThen` chains (or `bind` chains in general) that cleans this up quite a bit:

```js
function promiseHeaven(params) {
    return sequenced {
        const results = exec makeRequest(params);
        const data = exec processResults(results);
        const reponse = exec sendResults(data);
        return reponse.id;
    }
}
```

This would act the same as `promisePurgatory` above, but look at how natural it is! Instead of writing long method chains or deep pyramids of callbacks, we can write asynchronous code as if it were synchronous. Let's look at another example:

```ts
function motherInLawPicture(user: User): Optional<Image> {
    return sequenced {
        const spouse: User = exec user.getSpouse();
        const mother: User = exec spouse.getMother();
        return exec mother.getProfilePicture();
    }
}
```

In this example, we're sequencing actions that might not produce a value; the user may not have a spouse listed on the platform, the spouse may not have a mother listed on the platform, and the mother in law may not have set a profile picture. Despite that, we can seamlessly write this without explicitly checking for a missing value once.

# Async/Await

Many readers will probably be aware that JavaScript does have a feature to sequence async operations together: `async`/`await`. It even looks very similar to the hypothetical `sequenced`/`exec` feature I described here:

```js
async function promiseHeaven(params) {
    const results = await makeRequest(params);
    const data = await processResults(results);
    const response = await sendResults(data);
    return response.id;
}
```

With this post I hope you can see how a more general form of `async`/`await` is possible using monads, allowing us to seamlessly write code using all sorts of effects and actions. While it's probably too late for JavaScript to include such a feature, I think it's worth considering how abstractions from functional programming can solve problems effectively.

# Bonus: Monoids

You might still be unsatisfied with my conceptualization of `Array`'s `flatMap` as a way to sequence actions; it is less intuitive than the more literal understanding of how it flattens nested structure. Let's look at an abstraction that fits this concept better, starting with a look at `string`.

If I write `array.flatMap(functionReturningArray)`, I'll get a flat array back. I might also want to be able to write `array.flatMap(functionReturningString)` and get a single string back, but the signature of `flatMap` doesn't allow for this. Let's add a method `concatMap` that can handle this case:

```js
const contents = objects.concatMap(obj => obj.text);
```

Nice, this will combine all of the `text` into one string. Since we've been looking for abstractions, though, what other things could we combine? We've already seen we can combine arrays with `flatMap`, but how about numbers?

```js
const total = cart.sum(item => item.price);
```

This works, but I might want the product of a list of numbers:

```js
const totalHealth
    = baseHealth
    * equipment.product(item => item.multiplier);
```

In all three of these cases, we have an element type that we can combine somehow, and, though I didn't mention it, we also have a starting element, `""` for concatenating strings, `0` for adding numbers, and `1` for multiplying numbers. This is what forms the monoid abstraction:
- Some values (in our case, a type)
- An (associative) operation that combines two values, which I'll use the operator `<>` for
- An identity for this operation, i.e. a value `i` such that `x <> i = x` and `i <> x = x`

Notice that numbers actually have two monoids, one that uses addition for `<>`, and one that uses multiplication for `<>`.

Let's make a method for folding up a list of monoids into one value using `<>`:

```js
const contents = objects.foldMap(obj => obj.text);

const totalPrice = cart.foldMap(item => Sum(item.price)).value;

const totalHealth
    = baseHealth
    * equipment.foldMap(item => Product(item.multiplier)).value;
```

Our general `foldMap` can work on strings, numbers, booleans, lists, `Promises` that make other monoids, functions, and many more!

# Summary

Functors:
- Represent something you can map values inside of
- Have a method `map`: `<T, U>(this: M<T>, f: (t: T) => U) => M<U>`
- Must follow the identity rule, `m.map(x => x) = m`

Monads:
- Represent actions that can be sequenced together
- Have a method `bind`: `<T, U>(this: M<T>, f: (t: T) => M<U>) => M<U>`
- Have a function `unit`: `<T>(t: T) => M<T>`
- Must follow three rules:
    - Right Identity: `unit(x).bind(f) = f(x)`
    - Left Identity: `m.bind(unit) = m`
    - Associativity: `m.bind(x => f(x).bind(g)) = m.bind(f).bind(g)`

Monoids:
- Represent values that can be combined, making another value of the same type
- Have an operator `<>`: `(l: M, r: M) => M`
- Have an identity value `i`: `i <> x = x`, `x <> i = x`
- Must follow the associativity rule: `(x <> y) <> z = x <> (y <> z)`

Thanks for coming along, and until next time o7
