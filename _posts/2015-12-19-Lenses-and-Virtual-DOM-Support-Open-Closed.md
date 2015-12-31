---
author: Hardy Jones
categories: programming
date:   2015-12-19 06:50:42
layout: post
title: Lenses and Virtual DOM Support Open Closed
---

N.B. Some knowledge of van Laarhoven lenses, [ramda][ramda], and [virtual-dom][virtual-dom] is assumed.
This post will not go over the specifics of those.

## Table of contents

1. [Introduction](#introduction)
1. [Goal](#goal)
1. [Simple Radio](#simple-radio)
1. [Lenses](#lenses)
1. [Easier alternative](#easier-alternative)
1. [Complex Radio](#complex-radio)
1. [Compare Complex Solutions](#compare-complex-solutions)
1. [Wrap Up](#wrap-up)

## Introduction

Here is a *hypothetical* scenario:

```
It is an hour before the end of the day on Friday.
You are working on a large JavaScript project with no tests.
You have a function that takes some values and returns a stylized radio button.
You are tasked with making the radio button disabled for some customer requirement.
The function does not currently take into account the `disabled` attribute.
This function is used in multiple files.
```

Ignoring the insanity of working with a large js project that has no tests,
and the ridiculous business requirement to change a feature at the end of the day on Friday,
how do you handle this?
Well, there are a number of ways.

* You could modify the function to take into account the `disabled` field,
  and update all call sites.
  One problem with this is that it violates [The Open/Closed Principle][open-closed].
  Another problem is that since you are using JavaScript and you do not have tests,
  you are at the mercy of your find and replace skills, and whoever else might review your work.
  Yet another problem is that this assumes you own the source and can modify it.
* You could copy the implementation of the function
  to a new one that takes into account this `disabled` field.
  This is probably the simplest and most straight forward thing to do.
  You do not have to modify the original source,
  and it is a [hopefully] small amount of duplication.
  To quote [Sandi Metz][sandi-metz],
  ["duplication is far cheaper than using the wrong abstraction."][duplication]
  This solution is actually not so bad in the grand scheme of things,
  but it only really works if you/your team knows when to stop duplicating,
  and start abstracting.
* You could rely on the craziness of JavaScript and add an optional argument to the function.
  This is similar to the first suggestion, except you do not have to update every call site.
  But this is probably the last thing you want in a large js code base with no tests.

## Goal

There are many other things you could also do.
I would like to explore one specific option: using lenses to follow The Open/Closed Principle.
In particular, [ramda][ramda]'s implementation of van Laarhoven lenses
(though the idea is the same with other kinds of lenses).
If you are unfamiliar with lenses,
the simplified version is that they are first-class getters and setters.

Where you might write:

```js
foo.bar;
```

You could instead use a lens to write:

```js
view(bar, foo);
```

Similarly for setting:

```js
foo.bar = 3;
```

You could instead use a lens to write:

```js
set(bar, 3, foo);
```

And they compose:

```js
foo.bar.baz.quux;
```

You could instead use a lens to write:

```js
view(compose(bar, baz, quux), foo);
```

Similarly for setting:

```js
foo.bar.baz.quux = 3;
```

You could instead use a lens to write:

```js
set(compose(bar, baz, quux), 3, foo);
```

It is more verbose for sure,
but simple examples rarely show the benefits of an abstraction,
so stick with me on this one.

## Simple Radio

How does this help solve our original problem?
First, let's take a look at a simplified version.

```js
import {h} from 'virtual-dom';

function radio(name, value, description, actualValue) {
  return h('input', {
    checked: value === actualValue,
    type: 'radio',
    value
  }, description);
}
```

So the function takes some values, and creates a [`VNode`][vnode].

We can see the `VNode`:

```js
> radio('foo', 'bar', 'Bar', 'bar');
VirtualNode {
  tagName: 'input',
  properties:
   { checked: true,
     type: 'radio',
     value: SoftSetHook { value: 'bar' } },
  children: [ VirtualText { text: 'Bar' } ],
  key: undefined,
  namespace: null,
  count: 1,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false }
> radio('foo', 'baz', 'Baz', 'bar');
VirtualNode {
  tagName: 'input',
  properties:
   { checked: false,
     type: 'radio',
     value: SoftSetHook { value: 'baz' } },
  children: [ VirtualText { text: 'Baz' } ],
  key: undefined,
  namespace: null,
  count: 1,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false }
```

The important thing here is that a `VNode` is just an object.
It does not mutate the DOM when you create one.
Since it is just an object, we can manipulate it like we do any other js value.

Look closely at the result of `radio`.
The `VNode` has a field `properties`.
This field is an object that will correspond to the attributes of the actual DOM element.
Notice we have the properties: `checked`, `type`, and `value`.
We need some way to add another property: `disabled`.

## Lenses

This is where lenses come in.
Ramda provides a couple of simple lenses for two common tasks:
getting and setting fields on an object ([`lensProp`][lensProp]),
and getting and settings indices on an array ([`lensIndex`][lensIndex]).

We can use `lensProp` to set the `disabled` field of a `radio`:

```js
import {compose, lensProp, set} from 'ramda';

const properties = lensProp('properties');
const disabled = lensProp('disabled');
const propsDisabled = compose(properties, disabled);
const disable = set(propsDisabled, true);
```

Here we create two lenses, and compose them.
Note that the composition of lenses seems to read in the opposite order from the composition of functions.
But this is entirely correct.
Transducers do a similar thing
(which they should, since they're a simplified version of van Laarhoven lenses).

We can see it in action:

```js
> const bar = radio('foo', 'bar', 'Bar', 'bar');
> bar
VirtualNode {
  tagName: 'input',
  properties:
   { checked: true,
     type: 'radio',
     value: SoftSetHook { value: 'bar' } },
  children: [ VirtualText { text: 'Bar' } ],
  key: undefined,
  namespace: null,
  count: 1,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false }
> disable(bar)
{ tagName: 'input',
  properties:
   { checked: true,
     type: 'radio',
     value: SoftSetHook { value: 'bar' },
     disabled: true },
  children: [ VirtualText { text: 'Bar' } ],
  key: undefined,
  namespace: null,
  count: 1,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false,
  version: '1',
  type: 'VirtualNode' }
```

Notice that aside from the slightly different console representation,
the only thing that has changed is we have a new property `disabled` with a value `true`.
This is great!

Also note, we have not actually modified the original `VNode`:

```js
> bar
VirtualNode {
  tagName: 'input',
  properties:
   { checked: true,
     type: 'radio',
     value: SoftSetHook { value: 'bar' } },
  children: [ VirtualText { text: 'Bar' } ],
  key: undefined,
  namespace: null,
  count: 1,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false }
```

Great! We do not have to worry about accidentally breaking something else through mutation.
And, we did not have to modify the `radio` function to make the change!
We could take this `radio` function and squirrel it away in a library somewhere,
now that we know we can change properties fairly easily.

So, we have a software entity (`radio`) that is open for extension,
but closed for modification.
If that does not epitomize The Open/Closed Principle, I am not sure what does.

## Easier alternative

But wait, did we really need to bring lenses in for this, or are we just being esoteric?
I will admit for this simple example, the full power of lenses is a bit unwarranted.
After all, we could have written `disable` like this:

```js
function disableMutation(vnode) {
  vnode.properties.disabled = true;
  return vnode;
}
```

This is arguably easier (only because most of us have a frame of reference which is mutation first),
we do not introduce any additional concepts over plain js,
and it is very expressive.
It would have worked similarly:

```js
> disableMutation(bar)
VirtualNode {
  tagName: 'input',
  properties:
   { checked: true,
     type: 'radio',
     value: SoftSetHook { value: 'bar' },
     disabled: true },
  children: [ VirtualText { text: 'Bar' } ],
  key: undefined,
  namespace: null,
  count: 1,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false }
```

But notice, that we have mutated our original radio.

```js
> bar
VirtualNode {
  tagName: 'input',
  properties:
   { checked: true,
     type: 'radio',
     value: SoftSetHook { value: 'bar' },
     disabled: true },
  children: [ VirtualText { text: 'Bar' } ],
  key: undefined,
  namespace: null,
  count: 1,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false }
```

And mutation for the sake of easiness is not a trade-off we should be willing to make.

## Complex Radio

The real power of lenses shows itself when the object is quite a bit more nested.
So let's make a slightly more complex example.

```js
function complexRadio(name, value, description, actualValue) {
  return h('div.some-formatting-container', [
    h('div.some-other-formatting-container', [
      h('span', 'Some text about the radio'),
      radio(name, value, description, actualValue),
    ]),
  ]);
}
```

Using similar values, we get:

```js
> const bar = complexRadio('foo', 'bar', 'Bar', 'bar')
> bar
VirtualNode {
  tagName: 'div',
  properties: { className: 'some-formatting-container' },
  children:
   [ VirtualNode {
       tagName: 'div',
       properties: [Object],
       children: [Object],
       key: undefined,
       namespace: null,
       count: 4,
       hasWidgets: false,
       hasThunks: false,
       hooks: undefined,
       descendantHooks: false } ],
  key: undefined,
  namespace: null,
  count: 5,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false }
> bar.children[0].children[1]
VirtualNode {
  tagName: 'input',
  properties:
   { checked: true,
     type: 'radio',
     value: SoftSetHook { value: 'bar' } },
  children: [ VirtualText { text: 'Bar' } ],
  key: undefined,
  namespace: null,
  count: 1,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false }
```

Let's first take a crack at the mutation version.

```js
function complexDisableMutation(vnode) {
  vnode.children[0].children[1].properties.disabled = true;
  return vnode;
}
```

Easy, and it works too.

```js
> complexDisableMutation(bar)
VirtualNode {
  tagName: 'div',
  properties: { className: 'some-formatting-container' },
  children:
   [ VirtualNode {
       tagName: 'div',
       properties: [Object],
       children: [Object],
       key: undefined,
       namespace: null,
       count: 4,
       hasWidgets: false,
       hasThunks: false,
       hooks: undefined,
       descendantHooks: false } ],
  key: undefined,
  namespace: null,
  count: 5,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false }
> bar.children[0].children[1]
VirtualNode {
  tagName: 'input',
  properties:
   { checked: true,
     type: 'radio',
     value: SoftSetHook { value: 'bar' },
     disabled: true },
  children: [ VirtualText { text: 'Bar' } ],
  key: undefined,
  namespace: null,
  count: 1,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false }
```

But again, it is riddled with mutation.

Now, let us take a look at the lens version.

```js
import {compose, lensIndex, lensProp, set} from 'ramda';

// Using the `propsDisabled` lens from before.

// We add a few more.
const children = lensProp('children');
const _0 = lensIndex(0);
const _1 = lensIndex(1);
const complexDisabled = compose(children, _0, children, _1, propsDisabled);
const complexDisable = set(complexDisabled, true);
```

And it works similarly (minus the mutation):

```js
> const bar = complexRadio('foo', 'bar', 'Bar', 'bar')
> bar
VirtualNode {
  tagName: 'div',
  properties: { className: 'some-formatting-container' },
  children:
   [ VirtualNode {
       tagName: 'div',
       properties: [Object],
       children: [Object],
       key: undefined,
       namespace: null,
       count: 4,
       hasWidgets: false,
       hasThunks: false,
       hooks: undefined,
       descendantHooks: false } ],
  key: undefined,
  namespace: null,
  count: 5,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false }
> complexDisable(bar).children[0].children[1]
{ tagName: 'input',
  properties:
   { checked: true,
     type: 'radio',
     value: SoftSetHook { value: 'bar' },
     disabled: true },
  children: [ VirtualText { text: 'Bar' } ],
  key: undefined,
  namespace: null,
  count: 1,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false,
  version: '1',
  type: 'VirtualNode' }
> bar
VirtualNode {
  tagName: 'div',
  properties: { className: 'some-formatting-container' },
  children:
   [ VirtualNode {
       tagName: 'div',
       properties: [Object],
       children: [Object],
       key: undefined,
       namespace: null,
       count: 4,
       hasWidgets: false,
       hasThunks: false,
       hooks: undefined,
       descendantHooks: false } ],
  key: undefined,
  namespace: null,
  count: 5,
  hasWidgets: false,
  hasThunks: false,
  hooks: undefined,
  descendantHooks: false }
```

## Compare Complex Solutions

You might object to the fact that we used previously defined lenses.
We could do similar with the mutation version:

```js
function complexDisableMutation(vnode) {
  vnode.children[0].children[1] = disableMutation(vnode.children[0].children[1]);
  return vnode;
}
```

That is not looking too good.
We have to repeat the exact same path on both sides.
And we cannot really abstract that repetition away,
since that line is a statement not an expression.

Also notice, `complexDisable` used the lenses directly,
rather than the `disable` function itself.
This exposes the fact that we're using lenses in `disable`,
but, we can rewrite `complexDisable` to use `disable` rather than the lenses directly:

```js
import {compose, lensIndex, lensProp, over} from 'ramda';

// Using the `disable` function from before.

// We add a few new lenses.
const children = lensProp('children');
const _0 = lensIndex(0);
const _1 = lensIndex(1);
const complexDisabled = compose(children, _0, children, _1);
const complexDisable = over(complexDisabled, disable);
```

This works just like before,
and we are none the wiser about how `disable` is implemented,
nor should we care.

If you are unfamiliar with `over`,
it works similar to `set`,
but allows you to apply a function to the focus of the lens.

```js
set(lensProp('foo'), 13, {foo: 3}); //=> {foo: 13}
over(lensProp('foo'), add(10), {foo: 3}); //=> {foo: 13}
```

## Wrap Up

Well, both solutions satisfied The Open/Closed Principle so we met that goal,
but was it worthwhile to bring lenses in?

For the simple example,
it is arguable whether lenses provide enough benefit for the added cognitive load.

For anything you might actually be faced with in the real world,
I would say resoundingly yes!

For reference, here are the two options presented:

```js
import {h} from 'virtual-dom';
import {compose, lensIndex, lensProp, over, set} from 'ramda';

function radio(name, value, description, actualValue) {
  return h('input', {
    checked: value === actualValue,
    type: 'radio',
    value
  }, description);
}

function complexRadio(name, value, description, actualValue) {
  return h('div.some-formatting-container', [
    h('div.some-other-formatting-container', [
      h('span', 'Some text about the radio'),
      radio(name, value, description, actualValue),
    ]),
  ]);
}

const _0 = lensIndex(0);
const _1 = lensIndex(1);
const children = lensProp('children');
const disabled = lensProp('disabled');
const properties = lensProp('properties');

const complexDisabled = compose(children, _0, children, _1);
const propsDisabled = compose(properties, disabled);

const disable = set(propsDisabled, true);
const complexDisable = over(complexDisabled, disable);

function disableMutation(vnode) {
  vnode.properties.disabled = true;
  return vnode;
}

function complexDisableMutation(vnode) {
  vnode.children[0].children[1] = disableMutation(vnode.children[0].children[1]);
  return vnode;
}
```

I will not deny that the lens solution is less familiar.
But it is worlds simpler to reason about.

I wonder if you can guess which solution got implemented.
Now for those tests...

[duplication]: https://youtu.be/8bZh5LMaSmE?t=14m52s
[open-closed]: https://en.wikipedia.org/wiki/Open/closed_principle
[ramda]: http://ramdajs.com
[sandi-metz]: http://www.sandimetz.com/
[vnode]: https://github.com/Matt-Esch/virtual-dom/blob/903d884a8e4f05f303ec6f2b920a3b5237cf8b92/docs/vnode.md
[lensIndex]: http://ramdajs.com/docs/#lensIndex
[lensProp]: http://ramdajs.com/docs/#lensProp
[virtual-dom]: https://github.com/Matt-Esch/virtual-dom
