# Chapter 8: Tupperware

## The Mighty Container


We've seen how to write programs which pipe data through a series of pure functions. They are declarative specifications of behaviour. But what about control flow, error handling, asynchronous actions, state and, dare I say, effects?! In this chapter, we will discover the foundation upon which all of these helpful abstractions are built.

First we will create a container. This container must hold any type of value; a ziplock that holds only tapioca pudding is rarely useful. It will be an object, but we will not give it properties and methods in the OO sense. No, we will treat it like a treasure chest - a special box that cradles our valuable data.

```js
var Container = function(x) {
  this.__value = x;
}

Container.of = function(x) { return new Container(x); };
```

Here is our first container. We've thoughtfully named it `Container`. We will use `Container.of` as a constructor which saves us from having to write that awful `new` keyword all over the place. There's more to the `of` function than meets the eye, but for now, think of it as the proper way to place values into our container.

Let's examine our brand new box...

```js
Container.of(3);
//=> Container(3)


Container.of('hotdogs');
//=> Container("hotdogs")


Container.of(Container.of({
  name: 'yoda',
}));
//=> Container(Container({name: "yoda" }))
```

If you are using node, you will see `{__value: x}` even though we've got ourselves a `Container(x)`. Chrome will output the type properly, but no matter; as long as we understand what a `Container` looks like, we'll be fine. In some environments you can overwrite the `inspect` method if you'd like, but we will not be so thorough. For this book, we will write the conceptual output as if we'd overwritten `inspect` as it's much more instructive than `{__value: x}` for pedagogical as well as aesthetic reasons.

Let's make a few things clear before we move on:

* `Container` is an object with one property. Lots of containers just hold one thing, though they aren't limited to one. We've arbitrarily named its property `__value`.

* The `__value` cannot be one specific type or our `Container` would hardly live up to the name.

* Once data goes into the `Container` it stays there. We *could* get it out by using `.__value`, but that would defeat the purpose.

The reasons we're doing this will become clear as a mason jar, but for now, bear with me.

## My First Functor

Once our value, whatever it may be, is in the container, we'll need a way to run functions on it.

```js
// (a -> b) -> Container a -> Container b
Container.prototype.map = function(f) {
  return Container.of(f(this.__value));
}
```

Why, it's just like Array's famous `map`, except we have `Container a` instead of `[a]`. And it works essentially the same way:

```js
Container.of(2).map(function(two) {
  return two + 2;
});
//=> Container(4)


Container.of("flamethrowers").map(function(s) {
  return s.toUpperCase();
});
//=> Container("FLAMETHROWERS")


Container.of("bombs").map(_.concat(' away')).map(_.prop('length'));
//=> Container(10)
```

We can work with our value without ever having to leave the `Container`. This is a remarkable thing. Our value in the `Container` is handed to the `map` function so we can fuss with it and afterward, returned to its `Container` for safe keeping. As a result of never leaving the `Container`, we can continue to `map` away, running functions as we please. We can even change the type as we go along as demonstrated in the latter of the three examples.

Wait a minute, if we keep calling `map`, it appears to be some sort of composition! What mathematical magic is at work here? Well chaps, we've just discovered *Functors*.

> A Functor is a type that implements `map` and obeys some laws

Yes, *Functor* is simply an interface with a contract. We could have just as easily named it *Mappable*, but now, where's the *fun* in that? Functors come from category theory and we'll look at the maths in detail toward the end of the chapter, but for now, let's work on intuition and practical uses for this bizarrely named interface.

What reason could we possibly have for bottling up a value and using `map` to get at it? The answer reveals itself if we choose a better question: What do we gain from asking our container to apply functions for us? Well, abstraction of function application. When we `map` a function, we ask the container type to run it for us. This is a very powerful concept, indeed.

## Schrödinger's Maybe

`Container` is fairly boring. In fact, it is usually called `Identity` and has about the same impact as our `id` function(again there is a mathematical connection we'll look at when the time is right). However, there are other functors, that is, container-like types that have a proper `map` function, which can provide useful behaviour whilst mapping. Let's define one now.

```js
var Maybe = function(x) {
  this.__value = x;
};

Maybe.of = function(x) {
  return new Maybe(x);
};

Maybe.prototype.isNothing = function() {
  return (this.__value === null || this.__value === undefined);
};

Maybe.prototype.map = function(f) {
  return this.isNothing() ? Maybe.of(null) : Maybe.of(f(this.__value));
};
```

Now, `Maybe` looks a lot like `Container` with one minor change: it will first check to see if it has a value before calling the supplied function. This has the effect of side stepping those pesky nulls as we `map`(Note that this implementation is simplified for teaching).

```js
Maybe.of('Malkovich Malkovich').map(match(/a/ig));
//=> Maybe(['a', 'a'])

Maybe.of(null).map(match(/a/ig));
//=> Maybe(null)

Maybe.of({
  name: 'Boris',
}).map(_.prop('age')).map(add(10));
//=> Maybe(null)

Maybe.of({
  name: 'Dinah',
  age: 14,
}).map(_.prop('age')).map(add(10));
//=> Maybe(24)
```

Notice our app doesn't explode with errors as we map functions over our null values. This is because `Maybe` will take care to check for a value each and every time it applies a function.

This dot syntax is perfectly fine and functional, but for reasons mentioned in Part 1, we'd like to maintain our pointfree style. As it happens, `map` is fully equipped to delegate to whatever functor it receives:

```js
//  map :: Functor f => (a -> b) -> f a -> f b
var map = curry(function(f, any_functor_at_all) {
  return any_functor_at_all.map(f);
});
```

This is delightful as we can carry on with composition per usual and `map` will work as expected. This is the case with ramda's `map` as well. We'll use dot notation when it's instructive and the pointfree version when it's convenient. Did you notice that? I've sneakily introduced extra notation into our type signature. The `Functor f =>` tells us that `f` must be a Functor. Not that difficult, but I felt I should mention it.

## Use cases

In the wild, we'll typically see `Maybe` used in functions which might fail to return a result.

```js
//  safeHead :: [a] -> Maybe(a)
var safeHead = function(xs) {
  return Maybe.of(xs[0]);
};

var streetName = compose(map(_.prop('street')), safeHead, _.prop('addresses'));

streetName({
  addresses: [],
});
// Maybe(null)

streetName({
  addresses: [{
    street: 'Shady Ln.',
    number: 4201,
  }],
});
// Maybe("Shady Ln.")
```

`safeHead` is like our normal `_.head`, but with added type safety. A curious thing happens when `Maybe` is introduced into our code; we are forced to deal with those sneaky `null` values. The `safeHead` function is honest and up front about its possible failure - there's really nothing to be ashamed of - and so it returns a `Maybe` to inform us of this matter. We are more than merely *informed*, however, because we are forced to `map` to get at the value we want since it is tucked away inside the `Maybe` object. Essentially, this is a `null` check enforced by the `safeHead` function itself. We can now sleep better at night knowing a `null` value won't rear its ugly, decapitated head when we least expect it. APIs like this will upgrade a flimsy application from paper and tacks to wood and nails. They will guarantee safer software.


Sometimes a function might return a `Maybe(null)` explicitly to signal failure. For instance:

```js
//  withdraw :: Number -> Account -> Maybe(Account)
var withdraw = curry(function(amount, account) {
  return account.balance >= amount ?
    Maybe.of({
      balance: account.balance - amount,
    }) :
    Maybe.of(null);
});

//  finishTransaction :: Account -> String
var finishTransaction = compose(remainingBalance, updateLedger); // <- these composed functions are hypothetical, not implemented here...

//  getTwenty :: Account -> Maybe(String)
var getTwenty = compose(map(finishTransaction), withdraw(20));


getTwenty({
  balance: 200.00,
});
// Maybe("Your balance is $180.00")

getTwenty({
  balance: 10.00,
});
// Maybe(null)
```

`withdraw` will tip its nose at us and return `Maybe(null)` if we're short on cash. This function also communicates its fickleness and leaves us no choice, but to `map` everything afterwards. The difference is that the `null` was intentional here. Instead of a `Maybe(String)`, we get the `Maybe(null)` back to signal failure and our application effectively halts in its tracks. This is important to note: if the `withdraw` fails, then `map` will sever the rest of our computation since it doesn't ever run the mapped functions, namely `finishTransaction`. This is precisely the intended behaviour as we'd prefer not to update our ledger or show a new balance if we hadn't successfully withdrawn funds.

