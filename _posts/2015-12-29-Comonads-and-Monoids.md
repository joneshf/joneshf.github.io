---
author: Hardy Jones
categories: programming
date:   2015-12-29 03:22:03
layout: post
title: Comonads and Monoids
---

## Table of Contents

1. [Introduction](#introduction)
1. [Tree Solution](#tree-solution)
1. [Functor](#functor)
1. [Comonad](#comonad)
1. [Foldable](#foldable)
1. [Monoid](#monoid)
1. [More Refactoring](#more-refactoring)
1. [Conclusion](#conclusion)

## Introduction

[@lucasfeliciano][@lucasfeliciano] asked a [question][lucas-question] on the [ramda][ramda] gitter recently:

> I'm struggling trying to do something here:
> I have a tri dimensional array and I want to change a atribute of a object that is inside the deepest array.
> But by doing that I need to change the reference to their parents
> To respect this scenario
>
> ```
> initial state
> A
> |\
> B C
> | |\
> D E F
> D changes, all parents are replaced
> A*
> |\
> B* C
> |  |\
> D* E F
> ```
>
> What is the best approach to do that using ramda?

The motivation for this problem is explained:

> ...if the reference of B change, A needs to change as well
>
> ```
> we know something has changed because A changed
> we know something in B changed because b changed
> we know everything below C hasnt changed because C hasnt changed
> ```
>
> see?
> That is a react concept
> How it compares the tree state
> to calculate what need to be re rendered

It is like a difference algorithm, something that might be used in react, virtual-dom or (more generally) **any** tree library.

## Tree Solution

After playing around with a couple of solutions,
I think there is an elegant solution using some of the less familiar concepts of comonads and monoids.

The first thing that will make this whole solution much easier is to use a more descriptive data type.
A 3-d array is not ideal for this problem.
I think the visual representation that [@lucasfeliciano][@lucasfeliciano] gave is a good model.
So we should start with a tree.
We will keep it simple and have a binary tree.
Since we want to know what nodes are affected by their children, we will hold on to some state in each node.
We call this state, the `annotation`.
The leaves hold the values of the tree and an annotation.
The branches hold more trees and an annotation.
The motivation for adding an annotation to each node is:
when a node is changed, its annotation can reflect this change without affecting the actual value of the node.

We can create some constructors that encapsulate this logic.

```js
// Leaf : val -> ann -> Tree val ann
function Leaf(val, ann) {
  return {
    ann: ann,
    val: val,
    toString: () => `Leaf(${val}, ${ann})`,
  };
}
// Branch : (Tree val ann) -> (Tree val ann) -> ann -> Tree val ann
function Branch(left, right, ann) {
  return {
    ann: ann,
    left: left,
    right: right,
    toString: () => `Branch(${left}, ${right}, ${ann})`,
  };
}
```

Note that we are just returning a plain old javascript object (POJO).
There's no need for `new` or a `class` or anything like that.

## Functor

Given that we have this `Tree val ann` type with at least one type variable,
we can make a `Functor` from `Tree val`.
We choose to make the `Functor` over the `ann` variable,
since this is the one we care about in the solution to the problem.

The intuition for wanting a `Functor` in the solution comes from recognizing that we are probably going to want to apply some function uniformly across all nodes in the tree.
For instance, we might want to adjust every annotation in the tree all at once.
Or if we have a subtree, we might want to adjust all those nodes at once.
Recognizing that we can apply a function to all the values in the tree uniformly is the main take away here.

```js
// Leaf : val -> ann -> Tree val ann
function Leaf(val, ann) {
  return {
    ann: ann,
    val: val,
    toString: () => `Leaf(${val}, ${ann})`,
    map: f => Leaf(val, f(ann)),
  };
}
// Branch : (Tree val ann) -> (Tree val ann) -> ann -> Tree val ann
function Branch(left, right, ann) {
  return {
    ann: ann,
    left: left,
    right: right,
    toString: () => `Branch(${left}, ${right}, ${ann})`,
    map: f => Branch(left.map(f), right.map(f), f(ann)),
  };
}
```

Let us see this in action:

```js
> const l1 = Leaf(1, 'wat');
> const l2 = Leaf(2, 'yup');
> const b1 = Branch(l1, l2, 'nope');
> l1.toString();
'Leaf(1, wat)'
> l2.toString();
'Leaf(2, yup)'
> b1.toString();
'Branch(Leaf(1, wat), Leaf(2, yup), nope)'
> b1.map(str => str.length).toString();
'Branch(Leaf(1, 3), Leaf(2, 3), 4)'
> b1.map(_ => false).toString();
'Branch(Leaf(1, false), Leaf(2, false), false)'
```

Notice that we can only change the annotation.
The value stays the same throughout the tree.

This can be used to "reset" the tree annotations.
For instance, after we have found the nodes that need to change,
we can map everything to `false` so any new changes that are made can be tracked.

## Comonad

We need a way to propagate annotations through the branches of a tree.

Say we make our annotation a boolean where `false` implies unchanged, and `true` implies changed.
In the real world, we would want to use a more descriptive data type than boolean,
but for the sake of brevity, we stick with boolean.
When a child has been annotated `true`, the parent should also be annotated `true`.
Or thinking about it another way, to determine a parent's annotation, we must know the annotation of the children.

If we modify some leaf or branch in a tree and want to propagate the change upwards, what can we do?
We could write some specific algorithm that works only for booleans.
While this would solve the problem, it would have to be rewritten somehow if we change the annotation.

A slightly more interesting solution is to look towards comonads.
In this solution we can stop before we get the full power of a comonad.
We only need the power that `extend` provides.
The intuition for wanting a comonadic interface comes from realizing that in order to update the annotations of a tree,
the rest of the values in the tree are needed.
Put another way, the value of a single annotation depends on the value of the annotations around it.
Put yet another way, the context in which an annotation is in determines the value of that annotation.

We can implement `extend` fairly simply:

```js
// Leaf : val -> ann -> Tree val ann
function Leaf(val, ann) {
  return {
    ann: ann,
    val: val,
    toString: () => `Leaf(${val}, ${ann})`,
    map: f => Leaf(val, f(ann)),
    extend: f => Leaf(val, f(Leaf(val, ann))),
  };
}
// Branch : (Tree val ann) -> (Tree val ann) -> ann -> Tree val ann
function Branch(left, right, ann) {
  return {
    ann: ann,
    left: left,
    right: right,
    toString: () => `Branch(${left}, ${right}, ${ann})`,
    map: f => Branch(left.map(f), right.map(f), f(ann)),
    extend: f =>
      Branch(left.extend(f), right.extend(f), f(Branch(left, right, ann))),
  };
}
```

I am willing to bet that it is not immediately clear how this is helpful.
Bear with me for a bit.

Well, what if we had some function that took a `Tree` and returned a boolean that said whether or not any of the nodes had been updated?
If we had this function, we could pass it to `extend`, and it would change all the annotations to reflect that.

So let us think about that function.

How do you know if a `Leaf` is changed?
The `ann` tells you directly.

How do you know if a `Branch` is changed?
Either the `ann` or the `left` side has been changed or the `right` side has been changed.
If either side is a `Leaf`, we know directly.
If either side is a `Branch`, we have to continue recursing.

We can implement this function fairly simply as well.

```js
// Leaf : val -> ann -> Tree val ann
function Leaf(val, ann) {
  return {
    ann: ann,
    val: val,
    toString: () => `Leaf(${val}, ${ann})`,
    map: f => Leaf(val, f(ann)),
    extend: f => Leaf(val, f(Leaf(val, ann))),
    changed: ann,
  };
}
// Branch : (Tree val ann) -> (Tree val ann) -> ann -> Tree val ann
function Branch(left, right, ann) {
  return {
    ann: ann,
    left: left,
    right: right,
    toString: () => `Branch(${left}, ${right}, ${ann})`,
    map: f => Branch(left.map(f), right.map(f), f(ann)),
    extend: f =>
      Branch(left.extend(f), right.extend(f), f(Branch(left, right, ann))),
    changed: ann || left.changed || right.changed,
  };
}
```

Before we see if this works,
we should recognize there is a big problem with `changed`.
The types of `Leaf` and `Branch` are said to take any annotation,
but here we are assuming that the given annotation is a boolean.
While it is fine to have a boolean specific version,
it breaks the `Functor` and `Extend` implementations we just wrote if we force the annotation to be a boolean.
A `Functor` cannot work over a specific data type,
it must work with every data type.

## Foldable

All is not lost though.
Look long enough at `changed` and you might recognize that it is taking our tree structure and breaking it down to a single value.
In this case, we are breaking the tree down to a single boolean value.
This idea has an abstraction: `Foldable`.
So we should be able to implement `reduce` and then rewrite `changed` in terms of `reduce`.

```js
// Leaf : val -> ann -> Tree val ann
function Leaf(val, ann) {
  return {
    ann: ann,
    val: val,
    toString: () => `Leaf(${val}, ${ann})`,
    map: f => Leaf(val, f(ann)),
    extend: f => Leaf(val, f(Leaf(val, ann))),
    reduce: (f, acc) => f(acc, ann),
  };
}
// Branch : (Tree val ann) -> (Tree val ann) -> ann -> Tree val ann
function Branch(left, right, ann) {
  return {
    ann: ann,
    left: left,
    right: right,
    toString: () => `Branch(${left}, ${right}, ${ann})`,
    map: f => Branch(left.map(f), right.map(f), f(ann)),
    extend: f =>
      Branch(left.extend(f), right.extend(f), f(Branch(left, right, ann))),
    reduce: (f, acc) => right.reduce(f, left.reduce(f, f(acc, ann))),
  };
}
// changed : Tree val Bool -> Bool
const changed = tree => tree.reduce((acc, x) => acc || x, false);
```

Notice that we have moved `changed` to be a plain function rather than being a method on the `Tree`.
This makes things a bit easier to work with.

Now, let us see if it works:

```js
> const l1 = Leaf(1, false);
> const l2 = Leaf(2, false);
> const l3 = Leaf(3, true);
> const b1 = Branch(l1, l2, false);
> const b2 = Branch(b1, l3, false);
> changed(l1);
false
> changed(l2);
false
> changed(l3);
true
> changed(b1);
false
> changed(b2);
true
```

Seems like it works.
`l3` is changed, and it affects only `l3` and its ancestors (`b2` in this case).

Now we can use this in our implementation.

Looking back at the implementation of `extend`,
notice we always pass an entire `Tree val ann` to the given function.
It takes the `Tree val ann` and produces a single value.
This is exactly what `reduce` does.
So we can pass our `changed` function to `extend` and it will annotate the tree with its own changes.

```js
> const l1 = Leaf(1, false);
> const l2 = Leaf(2, false);
> const l3 = Leaf(3, true);
> const b1 = Branch(l1, l2, false);
> const b2 = Branch(b1, l3, false);
> b2.extend(changed).toString();
'Branch(Branch(Leaf(1, false), Leaf(2, false), false), Leaf(3, true), true)'
```

Excellent! We have arrived at a solution!

## Monoid

Now that we have an actual solution, we should think about ways to improve it.

There is one thing that stands out immediately to me.
The changed function is implemented directly in terms of `reduce`.
Lately, I have been noticing that whenever there is a `reduce` around, it is indicative of an abstraction somewhere.
Many people call this a [code smell][code smell].

What is the underlying abstraction we are missing?
We are reducing with `||` and the initial value of `false`.
This binary function and value form a `Monoid`.

How would we actually abstract it away?
Make a `Monoid` implementation for booleans.
But since we do not want to [monkey patch][monkey patch] a built-in,
we will create a new data type specifically for this.

```js
// Any : Bool -> Any
function Any(bool) {
  return {
    bool: bool,
    concat: a => Any(bool || a.bool),
  };
}
Any.empty = () => Any(false);
```

So if we had all of the annotations in the tree as `Any`s,
we could reduce over the tree using the `Monoid` interface.
This function has a name, [`fold`][fold] in Haskell.
Let us implement this function on a base object that both `Leaf` and `Branch` extend.

```js
// Tree : Tree val ann
const Tree = {
  fold: function(Monoid) {
    return this.reduce((acc, x) => acc.concat(x), Monoid.empty());
  },
};
// Leaf : val -> ann -> Tree val ann
function Leaf(val, ann) {
  return Object.assign(Tree, {
    ann: ann,
    val: val,
    toString: () => `Leaf(${val}, ${ann})`,
    map: f => Leaf(val, f(ann)),
    extend: f => Leaf(val, f(Leaf(val, ann))),
    reduce: (f, acc) => f(acc, ann),
  });
}
// Branch : (Tree val ann) -> (Tree val ann) -> ann -> Tree val ann
function Branch(left, right, ann) {
  return Object.assign(Tree, {
    ann: ann,
    left: left,
    right: right,
    toString: () => `Branch(${left}, ${right}, ${ann})`,
    map: f => Branch(left.map(f), right.map(f), f(ann)),
    extend: f =>
      Branch(left.extend(f), right.extend(f), f(Branch(left, right, ann))),
    reduce: (f, acc) => right.reduce(f, left.reduce(f, f(acc, ann))),
  });
}
```

Note that since there is no compiler for js,
we have to explicitly state which `Monoid` we want to use.

We can put these together to make our changed function clearer.

```js
// Tree : Tree val ann
const Tree = {
  fold: function(Monoid) {
    return this.reduce((acc, x) => acc.concat(x), Monoid.empty());
  },
};
// Leaf : val -> ann -> Tree val ann
function Leaf(val, ann) {
  return Object.assign({
    ann: ann,
    val: val,
    toString: () => `Leaf(${val}, ${ann})`,
    map: f => Leaf(val, f(ann)),
    extend: f => Leaf(val, f(Leaf(val, ann))),
    reduce: (f, acc) => f(acc, ann),
  }, Tree);
}
// Branch : (Tree val ann) -> (Tree val ann) -> ann -> Tree val ann
function Branch(left, right, ann) {
  return Object.assign({
    ann: ann,
    left: left,
    right: right,
    toString: () => `Branch(${left}, ${right}, ${ann})`,
    map: f => Branch(left.map(f), right.map(f), f(ann)),
    extend: f =>
      Branch(left.extend(f), right.extend(f), f(Branch(left, right, ann))),
    reduce: (f, acc) => right.reduce(f, left.reduce(f, f(acc, ann))),
  }, Tree);
}
// Any : Bool -> Any
function Any(bool) {
  return {
    bool: bool,
    concat: a => Any(bool || a.bool),
  };
}
Any.empty = () => Any(false);
// changed : Tree val Bool -> Bool
const changed = tree => tree.map(Any).fold(Any).bool;
```

And we can see it in action:

```js
> const l1 = Leaf(1, false);
> const l2 = Leaf(2, false);
> const l3 = Leaf(3, true);
> const b1 = Branch(l1, l2, false);
> const b2 = Branch(b1, l3, false);
> b2.extend(changed).toString();
'Branch(Branch(Leaf(1, false), Leaf(2, false), false), Leaf(3, true), true)'
```

Great! The refactoring worked!

## More Refactoring

Notice now however, we have gotten less efficient.
First we `map` over the entire tree, then we `fold` it.
We are iterating the tree twice, when before we iterated it once.

We can make this have the same efficiency as before if we look for more help from Haskell.
There is a function [`foldMap`][foldMap] that does exactly these two iterations in one go.

Let us add this to our `Tree` object.

```js
const R = require('ramda');
// Tree : Tree val ann
const Tree = {
  fold: function(Monoid) {
    return this.reduce((acc, x) => acc.concat(x), Monoid.empty());
  },
  foldMap: R.curry(function(Monoid, f) {
    return this.reduce((acc, x) => acc.concat(f(x)), Monoid.empty());
  }),
};
// Leaf : val -> ann -> Tree val ann
function Leaf(val, ann) {
  return Object.assign({
    ann: ann,
    val: val,
    toString: () => `Leaf(${val}, ${ann})`,
    map: f => Leaf(val, f(ann)),
    extend: f => Leaf(val, f(Leaf(val, ann))),
    reduce: (f, acc) => f(acc, ann),
  }, Tree);
}
// Branch : (Tree val ann) -> (Tree val ann) -> ann -> Tree val ann
function Branch(left, right, ann) {
  return Object.assign({
    ann: ann,
    left: left,
    right: right,
    toString: () => `Branch(${left}, ${right}, ${ann})`,
    map: f => Branch(left.map(f), right.map(f), f(ann)),
    extend: f =>
      Branch(left.extend(f), right.extend(f), f(Branch(left, right, ann))),
    reduce: (f, acc) => right.reduce(f, left.reduce(f, f(acc, ann))),
  }, Tree);
}
// Any : Bool -> Any
function Any(bool) {
  return {
    bool: bool,
    concat: a => Any(bool || a.bool),
  };
}
Any.empty = () => Any(false);
// changed : Tree val Bool -> Bool
const changed = tree => tree.foldMap(Any, Any).bool;
```

The fact that we pass `Any` twice is an unfortunate side effect of
not having a compiler to take care of this kind of boiler plate for us.

Once again, we can see it in action:

```js
> const l1 = Leaf(1, false);
> const l2 = Leaf(2, false);
> const l3 = Leaf(3, true);
> const b1 = Branch(l1, l2, false);
> const b2 = Branch(b1, l3, false);
> b2.extend(changed).toString();
'Branch(Branch(Leaf(1, false), Leaf(2, false), false), Leaf(3, true), true)'
```

And once again, we see that the refactoring worked!

One final thing to note is

## Conclusion

Was it worth it implement the `Any` data type?
From the perspective of having to pass around what should be implicit arguments,
I would say no.
However, I can give a very real example of when this comes in handy.

While writing this post, I was testing the `reduce` version,
and not getting the correct result.
After fumbling about for a while,
I realized that I had given the wrong initial value to `reduce`: `true`.
This kind of mistake cannot happen when using `Any`,
because the initial value is encoded in the data type itself.
Of course, someone has to write that implementation first,
so it is still prone to failure at some point.
But if `Any` were in a library somewhere,
it would be much less likely to have an error since that library could guarantee a correct implementation.
Rather then each person writing their own implementation and
forgetting if the initial value should be `true` or whatever.

More importantly, was it beneficial to implement the `extend` function?
I would say wholeheartedly, yes!
Using this abstraction, we made the rest of the code simpler.
We could concentrate on each aspect of the problem in isolation,
rather than conflating concerns.
Since it required a `Functor` implementation,
we also got a way to modify all values in the tree at once.
This led to us being able to implement `foldMap` to regain efficiency after we lost a bit with abstraction.
Plus, we got to see the beginnings of a `Comonad` in action.
The only thing missing is the `extract` function.
That seems like a nice function to let people figure out on their own though...

[code smell]: http://martinfowler.com/bliki/CodeSmell.html
[fold]: http://hackage.haskell.org/package/base-4.8.1.0/docs/Data-Foldable.html#v:fold
[foldMap]: http://hackage.haskell.org/package/base-4.8.1.0/docs/Data-Foldable.html#v:foldMap
[@lucasfeliciano]: https://github.com/lucasfeliciano
[lucas-question]: https://gitter.im/ramda/ramda?at=567c02983acb611716ffac24
[monkey patch]: https://en.wikipedia.org/wiki/Monkey_patch
[ramda]: http://ramdajs.com
