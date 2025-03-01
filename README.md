# "Call-This" Operator (`fn@()` == `fn.call()`) for Javascript

* Stage: 0
* Champions: @tabatkins, @js-choi, others?

---

* [Motivation](#motivation)
* [Proposal](#proposal)
* [Comparison With Other Proposals](#comparison-with-other-proposals)
	* [bind-this operator](#bind-this)
	* [Extension operator](#extensions)
	* [Partial Function Application](#partial-function-application)
	* ["Uncurrying"](#uncurrying)

Motivation
==========

There are good reasons to want to "extract" a method from a class, and then later call that method, either on instances of the extracted class, or on unrelated objects that mimic the original class enough to be useful context objects.

For example, `Object.prototype.toString()` is a useful method for doing cheap, easy "brand"-like identification (not *reliable*, but useful), but won't be callable on any object whose class provides its own `toString()` method, or which has a null prototype.

For another example, most of the Array methods are intentionally defined to be generic and usable on any "array-like" with a `.length` property and numeric properties storing data. Reusing them, rather than writing your own equivalents by hand, is a reasonable thing to do.

Finally, some "defensive" coding styles want to guard against prototype mutation in general, and thus want to pull significant methods off of objects early in a script's lifetime, and then use them later rather than relying on the prototype still having the correct methods on it.

Currently, all of these use-cases can be handled by calling the method with `.call()`, like:

```js
const {slice, map} = Array.prototype;
const skipFirst = slice.call(arrayLike, 1);
```

This usage is in fact *very common*, particularly among "utility" code, as shown [by studies of large bodies of extant JS code](https://github.com/tc39/proposal-bind-this/issues/12). It's so common that it's probably worthwhile to make easier and more reliable.
Taking some text directly from the [bind-this proposal](https://github.com/tc39/proposal-bind-this/blob/main/README.md#why-a-bind-this-operator):

> In short:
> 
> 1. [`.bind`][bind] and [`.call`][call]
>    are very useful and very common in JavaScript codebases.
> 2. But `.bind` and `.call` are clunky and unergonomic.
> 
> [bind]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind
> [call]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call
> 
> The dynamic `this` binding is a fundamental part of JavaScript design and
> practice today. Because of this, developers frequently need to change the `this`
> binding. `.bind` and `.call` are arguably two of the most commonly used
> functions in all of JavaScript.
> 
> We can estimate `.bind` and `.call`’s prevalences using [Node Gzemnid][].
> Although [Gzemnid can be deceptive][], we are only seeking rough estimations.
> 
> [Node Gzemnid]: https://github.com/nodejs/Gzemnid
> [Gzemnid can be deceptive]: https://github.com/nodejs/Gzemnid/blob/main/README.md#deception
> 
> The following results are from the checked-in-Git source code of the top-1000
> downloaded NPM packages.
> 
> | Occurrences | Method      |
> | ----------: | ----------- |
> | 1,016,503   |`.map`       |
> | 315,922     |**`.call`**  |
> | 271,915     |`console.log`|
> | 182,292     |`.slice`     |
> | 170,248     |**`.bind`**  |
> | 168,872     |`.set`       |
> | 70,116      |`.push`      |
> 
> These results suggest that usage of `.bind` and `.call` are comparable to usage
> of other frequently used standard functions. In this dataset, their combined
> usage even exceeds that of `console.log`.
> 
> Obviously, [this methodology has many pitfalls][Gzemnid can be deceptive], but
> we are only looking for roughly estimated orders of magnitude relative to other
> baseline functions. Gzemnid counts each library’s codebase only once; it does
> not double-count dependencies.
> 
> In fact, this method definitely underestimates the prevalences of `.bind` and
> `.call` by excluding the large JavaScript codebases of Node and Deno. Node and
> Deno [copiously use bound functions for security][security-use-case] hundreds or
> thousands of times.
> 
> [security-use-case]: https://github.com/js-choi/proposal-bind-this/blob/main/security-use-case.md

Proposal
--------

We add a new calling syntax, the "call-this" operator `fn@(receiver, ...args)`, which invokes fn with the given receiver and arguments, identical to `fn.call(receiver, ...args)`.

Precedence is the same as normal function calling; the `@(` is a singular token treated identically to `(` when doing a normal function call. This is the same as the optional-call operator `.?()`, or partial function application `~()`.

Example usage:

```js
const {slice, map} = Array.prototype;
const skipFirst = slice@(arrayLike, 1);
const mapped = map@(arrayLike, x=>x+1);

const {toString} = Object.prototype;
const brand = toString@(randomObj);
```

Comparison With Other Proposals
-------------------------------

### Bind-This

The call-this operator is very similar to [the bind-this `::` operator](https://github.com/tc39/proposal-bind-this).
Rather than being based on `.call()`, bind-this is based on `.bind()`; `receiver::fn` is identical to `fn.bind(receiver)`. You can immediately invoke the bound function, as `receiver::fn(args)`, giving the same behavior as `fn@(receiver, args)` with call-this, but can do somewhat more as well.

So on the surface, the bind-this operator appears to completely subsume the call-this operator. However, I believe call-this is the better choice, for a few reasons.

1. Parsing/precedence is simpler. The bind-this operator operator has to put restrictions on the RHS of the operator in order to have reasonable and predictable parsing, and allow the immediate-call form: only dotted ident sequences are allowed, with parens required to do generic operations. For example, if you need to fetch the method out of a map, you can't write `newReceiver::fnmap.get("theMethod")`, because it's unclear whether the author means `fnmap.get("theMethod").bind(newReciever)` or `fnmap.get.bind(newReciever)("theMethod")` (aka a `.call()`). In fact, this sort of expression is still absolutely *valid*, it'll just always be interpreted as the second possibility, possibly giving a confusing runtime error.

	On the other hand, `fnmap.get("theMethod")@(newReciever)` is immediately clear and distinct from `fnmap.get@(newReceiver, "theMethod")`, with no chance of confusion. You know precisely what expression is getting invoked to provide the method, and what arguments are being passed to it.
	
2. Bind-this overlaps with [partial function application](https://github.com/tc39/proposal-partial-application). PFA makes it extremely easy to hard-bind a method to the object it's currently on - just write `obj.meth~(...)` and you're done, since the reciever is automatically captured by PFA when creating the temporary function. Bind-this can do this too, just more verbosely: `obj::obj.meth`, which is annoying to write and potentially problematic if `obj` is actually a non-trivial expression that now needs to be written and executed twice.

3. Bind-this overlaps with [the pipe operator `|>`](https://github.com/tc39/proposal-pipeline-operator). Using bind-this, one can write and invoke "methods" without having to add them to an object's prototype. For example:

	```js
	function zip(that, fn) {
		return this.map((e, i)=>{
			return fn(e, that[i]);
		});
	}
	[1,2,3]::zip([4,5,6], (a,b)=>a+b);
	
	// vs, with pipe
	function zip(arr1, arr2, fn) {
		return arr1.map((e, i)=>{
			return fn(e, arr2[i]);
		});
	}
	[1,2,3] |> zip(##, [4,5,6], (a,b)=>a+b);
	```
	
	Note that the `zip()` function had to be written to specifically expect and use its `this` binding, 
	making it inconvenient and awkward to invoke in any way *other than* with this operator;
	you *must* call it either as `a::zip(b, fn)` or, verbosely, as `zip.call(a, b, fn)`.
	
	On the other hand, the pipe version is just a normal function, which can be called with pipe if you want a "method-chaining"-esque experience,
	or just as `zip(a, b, fn)` if that's not necessary and calling it normally is fine and clear.
	
	This was already one of the concerns cited when discussing pipe operator semantics,
	as a strike against the F#-style pipe.
	Authors would be incentivized to write libraries *intended* for the F#-style pipe,
	which are less convenient to invoke any other way
	(`a |> f(b)` with pipe, or `f(b)(a)` without);
	this potential ecosystem forking was seen as a problem to avoid.
	The adoption of Hack-style pipe syntax avoided this entirely,
	but bind-this brings it back.
	
	Note that call-this does *not* introduce such problems;
	while a library author *could* still write free functions that rely on a `this` binding being provided,
	calling it would be done as `zip@(a, b, fn)`,
	which is identical but slightly less convenient to just writing the function as taking all its arguments normally
	so it's invokeable as `zip(a, b, fn)`.
	Thus there's simply no reason for an author to write their library that way,
	and the ecosystem-forking concerns are minimal.
	
### Extensions

The [Extensions operator](https://github.com/tc39/proposal-extensions) is essentially just bind-this's "methods without having to mutate prototypes" functionality
(aka the "bind, but immediately invoke" syntax),
with some more functionality.
It suffers from identical overlap and issues as bind-this.
	
### Partial Function Application

No direct overlap - PFA creates new functions by binding some of the arguments of an existing function, possibly including the receiver of a method.
It doesn't directly invoke functions.

The only conceptual overlap is that PFA makes it easy to hard-bind a method to the object it's already on,
like `const fn = obj.meth~(...); fn(a, b, c);`,
and call-this allows one to extract a method from a class
and then call it on objects of that class,
but that's only similar at a high level;
in practice the two use-cases are quite distinct.

On the plus side, the two proposals potentially work together quite well--
PFA does *not* allow one to hard-bind a method against *a different object*,
like `.bind()` allows,
but it can easily be defined to work together with the call-this operator to achieve this:
`meth~@(newReciever, ...)` is now equivalent to `meth.bind(newReceiver)`.

### "Uncurrying"

I don't think this has an official proposal, but it's a legit alternative: a `Function.uncurry(fn)` that is equivalent to `fn.call.bind(fn)`;
aka it converts a function that uses `this` to instead take its `this` argument as its first argument.

In other words:

```js
const slice = Function.uncurry(Array.prototype.slice);
const skipFirst = slice(arrayLike, 1);
```

Alternately, rather than a built-in function this might be an operator, like:

```js
const slice = ::Array.prototype.slice;
const skipFirst = slice(arrayLike, 1);
```

The benefit of this approach is that it doesn't introduce *any* new calling conventions or syntax. 
You end up with a perfectly ordinary function that can be called normally,
passed around to things that expect to pass data as arguments,
etc.
It works with PFA and optional-calling *out of the box*,
without having to do *anything*.

There are two downsides, both relatively minor.
The first is that this creates additional closures,
whereas call-this doesn't.
That's a small but real tax on engines if it becomes very common to do this "on-the-fly".
Luckily that should be pretty unlikely,
since the syntax is somewhat annoying--
you have to wrap the thing in parens,
like `(::Array.prototype.slice)(arrayLike, 1)`.
That said, this pattern might be recognizable,
such that engines dont' actually generate the intermediate closure.

The second is that you can't easily apply it to multiple methods extracted from a prototype.
In the above call-this examples,
I could use object destructuring to pull multiple methods off of Array,
and then immediately use them with call-this.
An uncurry op/func would require you to extract them one-by-one instead,
so you could feed each one to the operator/function.
