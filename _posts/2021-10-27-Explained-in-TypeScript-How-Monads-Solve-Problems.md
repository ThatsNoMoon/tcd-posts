---
title: 'Explained in TypeScript: How Monads Solve Problems'
author: ThatsNoMoon
tags: [typescript, functional-programming]
categories: [Write-ups]
---

# Introduction: Callbacks

Imagine, for a moment, that it's 2013, you're involved in the language design of JavaScript, and you have a problem: callbacks are a mess. Because of the way the web was designed, JavaScript has to use asynchronous programming for I/O like making HTTP requests, but the only way to handle such async operations is to pass functions into other functions which pass other functions into more functions, on and on ad nauseum:

```ts
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

```ts
function synchronyHeaven(params) {
	const results = makeRequest(params);
	const data = processReslts(results);
	return sendResults(data);
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

```ts
asyncStep(args, (err, result) => {
	if (err) {
		// handle err
	} else {
		// use result
	}
});
```

From a certain perspective, we can see how the next step is made *part* of this step; the callback is, visually, inside the call to `asyncStep`. This means any usage of the results of `asyncStep` have to go inside this step using `asyncStep`. We can try to disconnect these steps a bit, like so:

```ts
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

```ts
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

```ts
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

Well, I think this looks neater. Whether you agree or not, you probably agree that it's still worse than the synchronous alternative; there's quite a bit of repetition, between creating and using each step and using the provided `errCb` in every step.

# Promises

How can we improve on our last version of `callbackHell`? Considering our goal is to make something as nice as our synchronous version, `synchronyHeaven`, a reasonable first step would be to try to make `const result = makeRequest(params);` meaningful, so let's make `makeRequest` return an object. What should this object do?

Since we're replacing a callback argument, how about we give this object methods to add callbacks:

```ts
const result = makeRequest(params);
result.onSuccess(processStep);
result.onFailure(errCallback);
```

This is certainly interesting, even if it hasn't made our code shorter. I'll call this object a promise, since it represents a guarantee that something will happen later, either a success or a failure.

Let's look at our `finish` function, and the step before it, translated into our promise style:

```ts
function sendStep(data) {
	const response = sendResults(data);
	response.onFailure(errCallback);
	response.onSuccess(finish);
}

function finish(response) {
	idCallback(response.id);
}
```

`finish` is a unique step in our function, as it doesn't do anything asynchronous, so we haven't changed it much yet. I'll examine a possible improvement to `sendStep` by adding a method to our callback object to do a synchronous step first:

```ts
function sendStep(data) {
	const response = sendResults(data);
	response.onFailure(errCallback);

	response.map(response => response.id);

	response.onSuccess(idCallback);
}
```

And via a little API manipulation:

```ts
function sendStep(data) {
	sendResults(data)
		.onFailure(errCallback);
		.map(response => response.id);
		.onSuccess(idCallback);
}
```

That's pretty neat, right? How about we try adding something similar to `map`, but instead executes asynchronous operations:

```ts
function processStep(results) {
	processResults(results)

		.andThen(data => sendResults(data))

		.onFailure(errCallback)
		.map(response => response.id)
		.onSuccess(idCallback);
}
```

Well, that's even better, let's try applying it to the whole function:

```ts
function promisePurgatory(params) {
	return makeRequest(params)
		.andThen(processResults)
		.andThen(sendResults)
		.map(response => response.id);
}
```

Nice! Notice that we're able to replace the callbacks in this function as well, and return a promise instead.

# Arrays

Let's look at another problem: transforming arrays. Say you're making a social media network, and you have an array of a user's friends. You might want to get an array of their usernames so you can easily display them on your website. It's 2013, so you write out a loop to make a new array:

```ts
const usernames = [];

for (const user of friends) {
	usernames.push(user.name);
}
```

# Monoids

Monoids, from group theory, are relatively easy to understand in terms of
familiar programming concepts. A monoid is a set of elements, a way to combine
them, an identity, and a few rules.

When using monoids in programming, the set of elements is usually a type, e.g.
in TypeScript, `string`. Types can be thought of as sets of values, so `string`
is the set of all possible strings. The (most obvious) way to combine strings is
by concatenation, so this will be our monoidal operation. The last thing to
define is the identity. An identity is an element which, when combined with any
other element, doesn't change that element. For string concatenation, the
identity is the empty string, `""`.

```ts
const a = "hello";
const b = " ";
const c = "world";

const d = a + b + c;
// d == "hello world"

const e = d + "";
// e == "hello world" == d
```

The rules a monoid need to follow are:
- Associativity: `(a + b) + c = a + (b + c)`
- Identity: `a + z = a` where `z` is the identity element

In summary:
- A monoid is a set of elements and an associative binary operation between them
  with an identity
- `string` is a monoid
- Its set is the set of all possible strings
- Its monoidal operation is concatenation
- Its identity element is `""`

# Functors

A functor, simply speaking, is something you can `map`. In TypeScript, a functor
is something that has an operation that looks like `(F<T>, ((T) => U)) => F<U>`,
i.e. you can take a container of `T`s, a function that turns a `T` into a `U`,
and get a container of `U`s[^functor-uses]. The example I'll use is `Array`,
which conveniently calls its mapping operation `map`:

```ts
const numbers = [1, 2, 3, 4];
const incrementedNumbers = numbers.map(x => x + 1);
// incrementedNumbers == [2, 3, 4, 5]

const strings = ["hi", "bye", "hello", "goodbye"];
const lengths = strings.map(s => s.length);
// lengths == [2, 3, 5, 7];
```
As we can see, `map` takes an `Array<T>` and transforms each `T` in the input
array to produce an output array. It can either transform the value into another
`T`, or it can transform it into a `U`, which changes the type of the array. In
the first case, `map` takes the form `(Array<number>, ((number) => number)) =>
Array<number>`, while in the second it takes the form `(Array<string>, ((string)
=> number)) => Array<number>`.

Functors also have an identity rule[^functor-composition]: `m.map(x => x) = m`

Some other examples of functors in TypeScript include `Set` `Promise`, `[T, T,
T]`, `Map` (if you pick a fixed key type) and `T | undefined`.

In summary:
- `Array` is a functor
- Its map operation is `array.map(f)`


# Monads
Monads are somewhat harder to wrap your mind around, which is why so many guides
have been devoted to the topic alone, but I hope to offer an explanation that
makes sense and shows how monads could be a useful abstraction, but doesn't get
too advanced.

Monad is like a sub-interface of functor, in that every monad is also a functor.
This means that every monad has a `map` operation. However, a monad also has a
`flatMap` operation that looks like `(M<T>, ((T) => M<U>)) => M<U>`, i.e. you can
take a container of `T`s, a function that makes a container of `U`s from a `T`,
and get a container of `U`s[^functor-uses]. `Array` is again the example I'll
use for a monad, and its monadic operation is called `flatMap`:

```ts
const numbers = [2, 4, 6];
const numberFactors = numbers.flatMap(x => factors(x));
// Assume `factors` is some function that returns an array
// of the prime factors of a number.
// numberFactors == [1, 2, 1, 4, 2, 1, 6, 2, 3]

const strings = ["hi", "bye", "hello", "goodbye"];
const edges = strings.flatMap(s => [s.slice(0, 1), s.slice(-1)]);
// edges = ["h", "i", "b", "e", "h", "o", "g", "e"];
```

I'll be referring to this monadic operation as `bind` later on.

Monads have one other function, `(T) => M<T>`, i.e. you can take a `T` and make
a container of `T`s (usually just containing that one `T`). I'll call this
`unit`, and for `Array` define it as:

```ts
const unit: <T>(t: T) => Array<T> = (t) => [t];
```

Monads have three rules:
- Left Identity: `unit(x).flatMap(f) = f(x)`
- Right Identity: `m.flatMap(unit) = m`
- Associativity: `m.flatMap(x => f(x).flatMap(g)) = m.flatMap(f).flatMap(g)`

Since it will come in handy later, there's an equivalent way to define monads in
terms of `join`, which is a function `(M<M<T>>) => M<T>`. `Array`'s `join`
operation is named `flat`:

```ts
const manyNumbers = [[1, 2], [3, 4], [5]];
const numbers = manyNumbers.flat();
// numbers = [1, 2, 3, 4, 5]
```

`bind` can be made from `join` by using `map`:

```ts
array.flatMap(f) = array.map(f).flat()
```

Likewise, `join` can be made from `bind`:

```ts
array.flat() = array.flatMap(x => x)
```

Thus these are equivalent ways to talk about monads.

Other examples of monads in TypeScript include most of the functors I listed
earlier, `Set` `Promise`, `Map` (if you pick a fixed key type) and `T |
undefined`.

In summary:
- A monad has an operation `unit`: `(T) => M<T>`
- A monad has an operation `bind`: `(M<T>, ((T) => M<U>)) => M<U>`
- A monad has an operation `join`: `(M<M<T>>) => M<T>`
- `Array` is a monad
- Its `unit` is `x => [x]`
- Its `bind` is `array.flatMap(f)`
- Its `join` is `array.flat()`

## Tangent: Why should I care?

*This section isn't necessary to understand for the rest of the article, but I
think it's a common enough question, and related enough, to warrant an answer*

The utility of monads is less obvious inside an imperative language, but the
gist of why monads are useful in functional programming is that the interface
allows you to sequence actions together. For example, I mentioned that `Promise`
is a monad, and this is used to great effect in TypeScript in the same way
monads are used in pure functional languages.

In a pure functional language, functions cannot have side effects, and have to
return the same thing for the same arguments. This means, in TypeScript lingo,
that `fetch` cannot return a `string`. But, as you may be aware already, `fetch`
can't even return a `string` in TypeScript, which is an impure language.

While pure functional languages use monads to contain side-effects, TypeScript
uses monads (`Promise` in particular) to contain asynchronous operations. The
signature of `bind`, `(M<T>, ((T) => M<T>)) => M<T>`, means that you are never
provided with a `T` directly, only through a callback. This means that in a pure
functional language, `fetch` can always return the same `Promise` representing
an incomplete HTTP request, and in TypeScript, `fetch` can return a `Promise`
without blocking.

Other monads, like `T | undefined`, represent other kinds of actions. `T |
undefined` is one monad that represents possible failure, where failure is the
absence of a value. `bind` can be used to sequence these actions together, using
the result of one to set up another.

# Categories

A category is some objects and some arrows between them. When thinking about
categories in programming, it's often useful to consider the category of types,
which I'll notate `Types`. In this category, all the possible types are objects,
and functions are arrows between them. For example:
- `string` is an object in the category `Types`
- `number` is an object
- `parseInt` is an arrow from `string` to `number`[^type-coercion]
- `isNaN` is an arrow from `number` to `boolean`
- `toUpperCase` is an arrow from `string` to `string`

Keep in mind that a category is both the objects and the arrows between them,
not just the objects. Don't confuse objects in a category with objects in a
programming language, the objects in the category we're interested in are
actually types.

# Endofunctors

First, functors. We've already discussed programming functors, and they are
related to category theory functors, but perhaps not in an obvious way.

A functor is a mapping between two categories in such a way that all of the
objects *and* all of the arrows in the first category are mapped to objects and
arrows in the second category.

An endofunctor, then, is a way to map from a category onto itself. In our
example category, the category of types, an endofunctor is a (certain) generic
type:
- `Array` takes any type `T`, and produces another type `Array<T>`
- `Array`'s `map` can take any function and produces another function that
	operates on arrays of that type: `f => arr => arr.map(f)`

From this comparison, we can see that my programming functor definition from
earlier is equivalent to an endofunctor on the category `Types`.

In summary:
- An endofunctor is a mapping from a category to itself
- `Array` is an endofunctor for the category `Types`
- `Array` takes a type and produces another type
- `Array`'s `map` maps functions on elements to functions on arrays

# Monoids and Categories

I started with a discussion of monoids, but now we need to connect monoids and
categories. A monoidal category is similar to a group theory monoid (from
above), but instead of being comprised of a set with a binary operation and
identity, it's comprised of a category with a binary operation and identity:
- A binary operation, which I'll notate ⊗. This operation takes two elements in
	the category and produces another element in the category.
- An identity to ⊗, which I'll notate `I`: `x ⊗ I = x`, `I ⊗ x = x`

A monoidal category is not the same as a monoid, however. A monoid in category
theory is an object `M` in a monoidal category, plus two operations:
- Multiplication, which is an arrow from `M ⊗ M` to `M`
- Unit, which is an arrow from `I` to `M`

Note that for the monoidal category, the operation was from any two objects in
the category to another object in the category, while for the monoid, the
operation is an arrow inside the category, and it goes from the result of `M ⊗ M`
to `M`. The monoidal category's `I` is an object that is an identity of ⊗, while
the monoid's unit is an arrow from `I` to `M`.

# Monoids in the Category of Endofunctors

We're in the home stretch!

First, I'll introduce the category of endofunctors. In the same way we've been
using a category of types, our category of endofunctors is, for our purposes,
the category of generic types (or type constructors) like `Array`.

The category of endofunctors is a monoidal category, because we can choose as
our binary operation composition: `Array ⊗ Array = <T> => Array<Array<T>>`. This
is a bit of made up syntax, but it essentially means that composing `Array` with
`Array` creates a new generic type that represents a nested `Array`. It means
that our `Array ⊗ Array` creates a generic type equivalent to this typedef
`DoubleArray`:

```ts
type DoubleArray<T> = Array<Array<T>>;
```

The identity element for this monoidal category is the identity type
constructor:

```ts
type Identity<T> = T;
```

We can see that `Array<Identity<T>>` is equivalent to `Array<T>`, and
`Identity<Array<T>>` is equivalent to `Array<T>`, thus `Identity ⊗ Array =
Array`, and `Array ⊗ Identity = Array`, satisfying our identity requirements.

Next, a monoid in the category of endofunctors is an object in our monoidal
category, e.g. `Array`, with two operations:
- Multiplication: `M ⊗ M => M`
- Unit: `I => M`

Considering `M ⊗ M` is `M` composed, or "nested", with itself, it's like saying
`<T> => M<M<T>>` with my imagined TypeScript syntax. Likewise, `M` and `I` are
themselves type functions `<T> => M<T>` and `<T> => T`.

Finally, compare the signatures for the `join` and `unit` operations on monads I
introduced previously to these monoidal multiplcation and unit operations:
```
join           :         M<M<T>>  =>         M<T>
multiplication : (<T> => M<M<T>>) => (<T> => M<T>)

unit (monad)  :         T  =>         M<T>
unit (monoid) : (<T> => T) => (<T> => M<T>)
```

And now, at long last, we can see these two formulations of monad are describing
the same thing! Our programming monad is defined in terms of functions on
values, and the category theory definition is defined in terms of operations on
types, but they describe the same things about monadic types.

As a final note, because monads are monoids in the category of endofunctors,
they are necessarily also endofunctors, and this means they're also functors
from my programming definition, which provides `map`. This means you can define
`bind` or `flatMap` in terms of this `join` or multiplication operation from the
mathematical monad definition.

Thanks for coming on this ride with me! If you're not from TCD, thanks
especially for joining me, but if you are, feel free to ask me any questions you
have about this material, I love talking about it!

Until next time o7

#### Footnotes

[^functor-uses]: Functors have more applications than simple containers of
	values, but I'll mostly stick with this conceptualization, as it's usually
	easier to understand.

[^functor-composition]: There's another law, composition, that states `f.map(x
	=> f(g(x))) = f.map(x => g(x)).map(x => f(x))`, but for our purposes, [it's
	enough to use only the first
	law](https://hackage.haskell.org/package/base-4.15.0.0/docs/Data-Functor.html#t:Functor)

[^type-coercion]: I'm ignoring type coercion, because it makes signatures harder
	to write for little benefit.
