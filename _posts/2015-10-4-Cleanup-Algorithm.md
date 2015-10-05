---
author: Hardy Jones
categories: programming
date:   2015-10-04 15:40:50
layout: post
title: Cleanup Algorithm
---

This is an example of some code I ran into in the wild.
It was much more unwieldy than this in the actual program,
but the ideas are the same.
Be warned, this is a long post, so long that I think a table of contents might help.

## Table of Contents

1. [Introduction](#introduction)
1. [Issues](#issues)
1. [Initial refactoring](#initial-refactoring)
1. [Out of scope mutation](#out-of-scope-mutation)
1. [Reducing reduce duplication](#reducing-reduce-duplication)
1. [Quick recap](#quick-recap)
1. [Ramda refactoring](#ramda-refactoring)
1. [Final recap](#final-recap)



## Introduction

Let's say we have the following data, an array of objects:

```js
var data = [
  { foo: '2', bar: '12', baz: '6', quux: true },
  { foo: '10', bar: '4', baz: '17', quux: false },
  { foo: '5', bar: null, baz: null, quux: false },
  { foo: '8', bar: '2', baz: '12', quux: false },
  { foo: '-2', bar: '2', baz: '9', quux: false },
  { foo: '-8', bar: '17', baz: '8', quux: false },
  { foo: '9', bar: '18', baz: null, quux: true },
  { foo: '6', bar: '14', baz: null, quux: false },
  { foo: '6', bar: null, baz: null, quux: true },
  { foo: '-999', bar: '18', baz: '14', quux: true },
  { foo: '-999', bar: '18', baz: '9', quux: true },
  { foo: '-6', bar: '12', baz: '17', quux: true },
  { foo: '-9', bar: '12', baz: '12', quux: true },
  { foo: '-999', bar: '20', baz: '3', quux: true },
  { foo: '-6', bar: '5', baz: '6', quux: true },
  { foo: '5', bar: '15', baz: '1', quux: true },
  { foo: '-8', bar: '0', baz: '10', quux: false },
  { foo: '0', bar: '9', baz: '17', quux: false },
];
```

And we want to transform it so that we end up with an object of arrays.

```js
var output1 = {
  foos: [], bars: [], bazs: [],
};
```

Notice that some of the `bar` and `baz` fields are `null`.
Also notice that some of the `foo` fields are `-999`.
For each of these cases, we have a special way to handle them.
 `null` values should use the previous value in the array corresponding to the key.

For instance, the third object in the array is

```js
{ foo: '5', bar: null, baz: null, quux: false }
```
.
When transforming our output, we want to use the value that was previous to it:

```js
{ foo: '5', bar: '4', baz: '17', quux: false }
```
.
So we keep all the non-`null` values, but replace `null`s with the previous value.

 `-999` values should be replaced with `0`.

A straight forward imperative approach might look like the following:

```js
function algorithm1(output, data) {
  var lastBar = 0;
  var lastBaz = 0;

  for (var i = 0; i < data.length; i++) {
    var datum = data[i];
    for (var key in datum) {
      if (datum.hasOwnProperty(key)) {
        if (key === 'foo') {
          var val = datum[key];
          if (val === '-999') {
            val = 0;
          }
          output.foos.push(Math.abs(Number(val)));
        } else if (key === 'bar') {
          var val = datum[key];
          if (val != null) {
            lastBar = val;
          }
          output.bars.push(Number(lastBar));
        } else if (key === 'baz') {
          var val = datum[key];
          if (val != null) {
            lastBaz = val;
          }
          output.bazs.push(Number(lastBaz));
        }
      }
    }
  }
}
```

And we can see the result of this

```js
algorithm1(output1, data);
console.log(output1);
//=>
// {
//   foos: [2, 10, 5, 8, 2, 8, 9, 6, 6, 0, 0, 6, 9, 0, 6, 5, 8, 0],
//   bars: [12, 4, 4, 2, 2, 17, 18, 14, 14, 18, 18, 12, 12, 20, 5, 15, 0, 9],
//   bazs: [6, 17, 17, 12, 9, 8, 8, 8, 8, 14, 9, 17, 12, 3, 6, 1, 10, 17]
// }
```

## Issues

Unfortunately, this approach has many problems.
The first being that it mutates its input argument.
Another problem is that it violates the open/closed principle.
It was written with the idea that each new case you had to handle would go in the conditional chain.
Yet another problem is that it has multiple responsibilities.

Given that it's 2015, at the time of writing,
we should not be writing imperative loops by hand to manipulate data.

## Initial refactoring

Let's rework this algorithm using ramda to pretend we care about our own sanity and want to write maintainable code.

Since we want to get rid of mutation, let's break the api of this function.
Rather than taking an object to mutate,
let's return a new object with the same values.

```js
function algorithm2(data) {
  var lastBar = 0;
  var lastBaz = 0;
  var foos = [];
  var bars = [];
  var bazs = [];

  for (var i = 0; i < data.length; i++) {
    var datum = data[i];
    for (var key in datum) {
      if (datum.hasOwnProperty(key)) {
        if (key === 'foo') {
          var val = datum[key];
          if (val === '-999') {
            val = 0;
          }
          foos.push(Math.abs(Number(val)));
        } else if (key === 'bar') {
          var val = datum[key];
          if (val != null) {
            lastBar = val;
          }
          bars.push(Number(lastBar));
        } else if (key === 'baz') {
          var val = datum[key];
          if (val != null) {
            lastBaz = val;
          }
          bazs.push(Number(lastBaz));
        }
      }
    }
  }

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}

var output2 = algorithm2(data);
console.log(output2);
//=>
// {
//   foos: [2, 10, 5, 8, 2, 8, 9, 6, 6, 0, 0, 6, 9, 0, 6, 5, 8, 0],
//   bars: [12, 4, 4, 2, 2, 17, 18, 14, 14, 18, 18, 12, 12, 20, 5, 15, 0, 9],
//   bazs: [6, 17, 17, 12, 9, 8, 8, 8, 8, 14, 9, 17, 12, 3, 6, 1, 10, 17]
// }
```

So far, so good. Not much has changed,
and in fact it's a bit harder to understand,
but its going to get a bit worse before it gets better.

Next thing to think about is that each case of the nested loop is mutually exclusive.
So we could in theory loop multiple times, and just perform one check in each nested loop.
Bear in mind, this is demonstrably less efficient.
However, the point of this is not to gain speed, it's to write maintainable code.

```js
function algorithm3(data) {
  var lastBar = 0;
  var lastBaz = 0;
  var foos = [];
  var bars = [];
  var bazs = [];

  for (var i = 0; i < data.length; i++) {
    var datum = data[i];

    for (var key in datum) {
      if (datum.hasOwnProperty(key)) {
        if (key === 'foo') {
          var val = datum[key];
          if (val === '-999') {
            val = 0;
          }
          foos.push(Math.abs(Number(val)));
        }
      }
    }

    for (var key in datum) {
      if (datum.hasOwnProperty(key)) {
        if (key === 'bar') {
          var val = datum[key];
          if (val != null) {
            lastBar = val;
          }
          bars.push(Number(lastBar));
        }
      }
    }

    for (var key in datum) {
      if (datum.hasOwnProperty(key)) {
        if (key === 'baz') {
          var val = datum[key];
          if (val != null) {
            lastBaz = val;
          }
          bazs.push(Number(lastBaz));
        }
      }
    }
  }

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

But notice that it doesn't matter the order in which we `push` to our arrays.
It doesn't even matter if we interleave the order (as we're currently doing).
So rather than interleaving this `push`ing, let's push all the `foo`s first,
then all the `bar`s, then all the `baz`s.

```js
function algorithm4(data) {
  var lastBar = 0;
  var lastBaz = 0;
  var foos = [];
  var bars = [];
  var bazs = [];

  for (var i = 0; i < data.length; i++) {
    var datum = data[i];

    for (var key in datum) {
      if (datum.hasOwnProperty(key)) {
        if (key === 'foo') {
          var val = datum[key];
          if (val === '-999') {
            val = 0;
          }
          foos.push(Math.abs(Number(val)));
        }
      }
    }
  }

  for (var i = 0; i < data.length; i++) {
    var datum = data[i];

    for (var key in datum) {
      if (datum.hasOwnProperty(key)) {
        if (key === 'bar') {
          var val = datum[key];
          if (val != null) {
            lastBar = val;
          }
          bars.push(Number(lastBar));
        }
      }
    }
  }

  for (var i = 0; i < data.length; i++) {
    var datum = data[i];

    for (var key in datum) {
      if (datum.hasOwnProperty(key)) {
        if (key === 'baz') {
          var val = datum[key];
          if (val != null) {
            lastBaz = val;
          }
          bazs.push(Number(lastBaz));
        }
      }
    }
  }

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

Now that we've got most of the groundwork laid,
the real fun can begin.

Since writing loops by hand is error prone, archaic, and not extensible,
let's do away with them.

As the loops don't actually return anything, we use `forEach` to iterate.

```js
function algorithm5(data) {
  var lastBar = 0;
  var lastBaz = 0;
  var foos = [];
  var bars = [];
  var bazs = [];

  data.forEach(function(datum) {
    for (var key in datum) {
      if (datum.hasOwnProperty(key)) {
        if (key === 'foo') {
          var val = datum[key];
          if (val === '-999') {
            val = 0;
          }
          foos.push(Math.abs(Number(val)));
        }
      }
    }
  });

  data.forEach(function(datum) {
    for (var key in datum) {
      if (datum.hasOwnProperty(key)) {
        if (key === 'bar') {
          var val = datum[key];
          if (val != null) {
            lastBar = val;
          }
          bars.push(Number(lastBar));
        }
      }
    }
  });

  data.forEach(function(datum) {
    for (var key in datum) {
      if (datum.hasOwnProperty(key)) {
        if (key === 'baz') {
          var val = datum[key];
          if (val != null) {
            lastBaz = val;
          }
          bazs.push(Number(lastBaz));
        }
      }
    }
  });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

We got rid of quite a bit of boiler plate, but there's still more to go.
Look at the inner loops.
Checking of `hasOwnProperty` is only necessary because `for..in` syntax enumerates all properties on an object.
If we use `Object.keys`, then we don't need to check `hasOwnProperty`,
since `Object.keys` only returns the own properties for objects.

What this means is that we can clean up quite a bit more.

```js
function algorithm6(data) {
  var lastBar = 0;
  var lastBaz = 0;
  var foos = [];
  var bars = [];
  var bazs = [];

  data.forEach(function(datum) {
    Object.keys(datum).forEach(function(key) {
      if (key === 'foo') {
        var val = datum[key];
        if (val === '-999') {
          val = 0;
        }
        foos.push(Math.abs(Number(val)));
      }
    });
  });

  data.forEach(function(datum) {
    Object.keys(datum).forEach(function(key) {
      if (key === 'bar') {
        var val = datum[key];
        if (val != null) {
          lastBar = val;
        }
        bars.push(Number(lastBar));
      }
    });
  });

  data.forEach(function(datum) {
    Object.keys(datum).forEach(function(key) {
      if (key === 'baz') {
        var val = datum[key];
        if (val != null) {
          lastBaz = val;
        }
        bazs.push(Number(lastBaz));
      }
    });
  });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

Now, rather than throwing the conditional check into the function,
we can filter for a cleaner pipeline.

```js
function algorithm7(data) {
  var lastBar = 0;
  var lastBaz = 0;
  var foos = [];
  var bars = [];
  var bazs = [];

  data.forEach(function(datum) {
    Object.keys(datum).filter(function(key) {
      return key === 'foo';
    }).forEach(function(key) {
      var val = datum[key];
      if (val === '-999') {
        val = 0;
      }
      foos.push(Math.abs(Number(val)));
    });
  });

  data.forEach(function(datum) {
    Object.keys(datum).filter(function(key) {
      return key === 'bar';
    }).forEach(function(key) {
      var val = datum[key];
      if (val != null) {
        lastBar = val;
      }
      bars.push(Number(lastBar));
    });
  });

  data.forEach(function(datum) {
    Object.keys(datum).filter(function(key) {
      return key === 'baz';
    }).forEach(function(key) {
      var val = datum[key];
      if (val != null) {
        lastBaz = val;
      }
      bazs.push(Number(lastBaz));
    });
  });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

A stylistic choice can be made here which will help with future refactorings.
We create a variable `val` within each last function.
Now, we can map out the `val` earlier in the pipeline.

```js
function algorithm8(data) {
  var lastBar = 0;
  var lastBaz = 0;
  var foos = [];
  var bars = [];
  var bazs = [];

  data.forEach(function(datum) {
    Object.keys(datum).filter(function(key) {
      return key === 'foo';
    }).map(function(key) {
      return datum[key];
    }).forEach(function(val) {
      if (val === '-999') {
        val = 0;
      }
      foos.push(Math.abs(Number(val)));
    });
  });

  data.forEach(function(datum) {
    Object.keys(datum).filter(function(key) {
      return key === 'bar';
    }).map(function(key) {
      return datum[key];
    }).forEach(function(val) {
      if (val != null) {
        lastBar = val;
      }
      bars.push(Number(lastBar));
    });
  });

  data.forEach(function(datum) {
    Object.keys(datum).filter(function(key) {
      return key === 'baz';
    }).map(function(key) {
      return datum[key];
    }).forEach(function(val) {
      if (val != null) {
        lastBaz = val;
      }
      bazs.push(Number(lastBaz));
    });
  });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

Now that we've made this choice, look at each of these pieces.
They all have this part to them:

```js
data.forEach(function(datum) {
  Object.keys(datum).filter(function(key) {
    return key === 'foo';
  }).map(function(key) {
    return datum[key];
  })...
});
```

What is happening here is that we're going over each datum,
then over each key in the datum, checking if it is correct,
and then pulling the value out.

That's all well and good, but it's horribly inefficient and terribly convoluted.
Let's clean that up right now, by replacing it by what we actually want.

We want to traverse each datum and pull out the specific value associated with the key.
We already know the key up front, so we can pull the value out in each case.

We want something like this:

```js
data.map(function(datum) {
  return datum.foo;
})...
```

Notice this also gets rid of the inner nesting, an overall win for readability!

```js
function algorithm9(data) {
  var lastBar = 0;
  var lastBaz = 0;
  var foos = [];
  var bars = [];
  var bazs = [];

  data.map(function(datum) {
    return datum.foo;
  }).forEach(function(val) {
    if (val === '-999') {
      val = 0;
    }
    foos.push(Math.abs(Number(val)));
  });

  data.map(function(datum) {
    return datum.bar;
  }).forEach(function(val) {
    if (val != null) {
      lastBar = val;
    }
    bars.push(Number(lastBar));
  });

  data.map(function(datum) {
    return datum.baz;
  }).forEach(function(val) {
    if (val != null) {
      lastBaz = val;
    }
    bazs.push(Number(lastBaz));
  });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

If we look at each of the last functions, we see they are all very similar:
the value is checked for an exceptional condition, and acted upon accordingly.
In the `foo` pipeline,
the exceptional value is replaced with a static value.
We can move this behavior out to another function.

```js
function algorithm10(data) {
  var lastBar = 0;
  var lastBaz = 0;
  var foos = [];
  var bars = [];
  var bazs = [];

  data.map(function(datum) {
    return datum.foo;
  }).map(function(val) {
    return val === '-999' ? 0 : val;
  }).forEach(function(val) {
    foos.push(Math.abs(Number(val)));
  });

  data.map(function(datum) {
    return datum.bar;
  }).forEach(function(val) {
    if (val != null) {
      lastBar = val;
    }
    bars.push(Number(lastBar));
  });

  data.map(function(datum) {
    return datum.baz;
  }).forEach(function(val) {
    if (val != null) {
      lastBaz = val;
    }
    bazs.push(Number(lastBaz));
  });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

For the other two pipelines, we cannot make such a simple conversion.
The reason we cannot make this conversion is that
each `map`/`forEach`/etc runs to completion before the next one starts.
In both of the `forEach`s, we're mutating a variable in an outer scope.
If we were to move this mutation up to its own pipeline,
the values pushed into `bars` and `bazs` would all be the same--`lastBar` and `lastBaz` respectively.

What we have is a coupling between the exceptional check and
the `push`ing of the value.

These are two separate behaviors, so we should want them to stay that way.
We can separate these behaviors by moving the exceptional check and mutation to its own function,
and returning the value from that function as well.

```js
function algorithm11(data) {
  var lastBar = 0;
  var lastBaz = 0;
  var foos = [];
  var bars = [];
  var bazs = [];

  data.map(function(datum) {
    return datum.foo;
  }).map(function(val) {
    return val === '-999' ? 0 : val;
  }).forEach(function(val) {
    foos.push(Math.abs(Number(val)));
  });

  data.map(function(datum) {
    return datum.bar;
  }).map(function(val) {
    if (val != null) {
      lastBar = val;
    }
    return lastBar;
  }).forEach(function(val) {
    bars.push(Number(val));
  });

  data.map(function(datum) {
    return datum.baz;
  }).map(function(val) {
    if (val != null) {
      lastBaz = val;
    }
    return lastBaz;
  }).forEach(function(val) {
    bazs.push(Number(val));
  });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

We'll revisit that mutation again later, for now let's continue with the `push` removal.

In each of the `forEach`s,
we're doing some cleanup to the value before we `push` into the respective array.
Let's move this cleanup out into the pipeline.

```js
function algorithm12(data) {
  var lastBar = 0;
  var lastBaz = 0;
  var foos = [];
  var bars = [];
  var bazs = [];

  data.map(function(datum) {
    return datum.foo;
  }).map(function(val) {
    return val === '-999' ? 0 : val;
  }).map(function(val) {
    return Math.abs(Number(val));
  }).forEach(function(val) {
    foos.push(val);
  });

  data.map(function(datum) {
    return datum.bar;
  }).map(function(val) {
    if (val != null) {
      lastBar = val;
    }
    return lastBar;
  }).map(function(val) {
    return Number(val);
  }).forEach(function(val) {
    bars.push(val);
  });

  data.map(function(datum) {
    return datum.baz;
  }).map(function(val) {
    if (val != null) {
      lastBaz = val;
    }
    return lastBaz;
  }).map(function(val) {
    return Number(val);
  }).forEach(function(val) {
    bazs.push(val);
  });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

Finally, we're ready to remove that `push`.
Look at each pipeline, the `forEach` does nothing worthwhile.
If instead of mutating with a `push`, we instead return the value,
then we can `map` instead of `forEach`, and just return the value.

We would end up with something like:

```js
data.map(function(datum) {
  ...
}).map(function(val) {
  return val;
});
```

But if we remember, there is a law that states `xs.map(function(x) { return x; }) == xs`.

So we don't even need the `map` at the end.
Let's just drop it all together.

```js
function algorithm13(data) {
  var lastBar = 0;
  var lastBaz = 0;

  var foos = data.map(function(datum) {
    return datum.foo;
  }).map(function(val) {
    return val === '-999' ? 0 : val;
  }).map(function(val) {
    return Math.abs(Number(val));
  });

  var bars = data.map(function(datum) {
    return datum.bar;
  }).map(function(val) {
    if (val != null) {
      lastBar = val;
    }
    return lastBar;
  }).map(function(val) {
    return Number(val);
  });

  var bazs = data.map(function(datum) {
    return datum.baz;
  }).map(function(val) {
    if (val != null) {
      lastBaz = val;
    }
    return lastBaz;
  }).map(function(val) {
    return Number(val);
  });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

## Out of scope mutation

Let's go back to that last piece of mutation we had: the `lastBar` and `lastBaz` variables.
What are we trying to encode in this piece of the program?
We want to traverse the entire array,
and fill in the exceptional values with the last normal value we saw.
Another way to think about it is:
we want to pass some state through the traversal and update it as necessary.

There are many ways to solve this problem.
One simple way is to use `reduce`.
 `reduce` encodes passing state through a traversal very well.
What state do we have in our piece of the program?
We want to keep track of the last normal value,
and we want to keep track of the values we're building.
So we need state that can contain that.
For the sake of simplicity, we'll use an object as our state: `{acc: [Number], last: Number}`.
We start off with an initial state of: `{acc: [], last: 0}`.

```js
function algorithm14(data) {
  var lastBar = 0;
  var lastBaz = 0;

  var foos = data.map(function(datum) {
    return datum.foo;
  }).map(function(val) {
    return val === '-999' ? 0 : val;
  }).map(function(val) {
    return Math.abs(Number(val));
  });

  var bars = data.map(function(datum) {
    return datum.bar;
  }).reduce(function(state, val) {
    if (val != null) {
      return {acc: state.acc.concat(val), last: val};
    } else {
      return {acc: state.acc.concat(state.last), last: state.last};
    }
  }, {acc: [], last: lastBar}).acc.map(function(val) {
    return Number(val);
  });

  var bazs = data.map(function(datum) {
    return datum.baz;
  }).reduce(function(state, val) {
    if (val != null) {
      return {acc: state.acc.concat(val), last: val};
    } else {
      return {acc: state.acc.concat(state.last), last: state.last};
    }
  }, {acc: [], last: lastBaz}).acc.map(function(val) {
    return Number(val);
  });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

Let's take a second to analyze what is happening here.
The `reduce` is operating on an array of numbers.
So each `val` in the function is a number (or `null` as it were).
The `state` in the function is our state we are passing through.
We start off with the initial state of `{acc: [], last: 0}`.
Each time through the array, we check the exceptional condition.
If the value is not exceptional, append it to the state accumulator, and give a new `last` value.
If the value is exceptional, append the last value to the state accumulator, and give the same last value.

When the `reduce` is finished, we end up with an object: `{acc: [Number], last: Number}`.
We don't actually care about `last` after the `reduce` has finished,
so we simply take the `acc` out of the object, and continue on our pipeline.

We can verify that this does the correct thing:

```js
var output14 = algorithm14(data);
console.log(output14);
//=>
// {
//   foos: [2, 10, 5, 8, 2, 8, 9, 6, 6, 0, 0, 6, 9, 0, 6, 5, 8, 0],
//   bars: [12, 4, 4, 2, 2, 17, 18, 14, 14, 18, 18, 12, 12, 20, 5, 15, 0, 9],
//   bazs: [6, 17, 17, 12, 9, 8, 8, 8, 8, 14, 9, 17, 12, 3, 6, 1, 10, 17]
// }
```

Threading state through a `reduce` is not the most elegant solution.
When you come back to this code later,
you have to then understand what the state is, how the state is updated,
and the fact that the state is threaded through,
There are more elegant solutions that only require you to understand what the state is,
and how it is updated.

## Reducing reduce duplication

The last thing to note is that the `foo` pipeline looks different from the others.
In order to promote reuse, we want each pipeline to be basically the same,
and parameterize it by what is different.
Just as we converted the last two pipelines from a `map` to a `reduce`,
we can do the same for the `foo`.

We just want the `foo` `reduce` to have the same interface as the others.
So the `foo` `reduce` needs to return an object with an `acc` field.

```js
function algorithm15(data) {
  var lastBar = 0;
  var lastBaz = 0;

  var foos = data.map(function(datum) {
    return datum.foo;
  }).reduce(function(state, val) {
    if (val === '-999') {
      return {acc: state.acc.concat(0)};
    } else {
      return {acc: state.acc.concat(val)};
    }
  }, {acc: []}).acc.map(function(val) {
    return Math.abs(Number(val));
  });

  var bars = data.map(function(datum) {
    return datum.bar;
  }).reduce(function(state, val) {
    if (val != null) {
      return {acc: state.acc.concat(val), last: val};
    } else {
      return {acc: state.acc.concat(state.last), last: state.last};
    }
  }, {acc: [], last: lastBar}).acc.map(function(val) {
    return Number(val);
  });

  var bazs = data.map(function(datum) {
    return datum.baz;
  }).reduce(function(state, val) {
    if (val != null) {
      return {acc: state.acc.concat(val), last: val};
    } else {
      return {acc: state.acc.concat(state.last), last: state.last};
    }
  }, {acc: [], last: lastBaz}).acc.map(function(val) {
    return Number(val);
  });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

Now we have each pipeline looking the same.
Note that we eschew the `last` field in the state of the `foo` pipeline simply because it's never used.
Let's start to parameterize our pipelines.

Each of the first `map`s simply pulls a field out of the `datum`.
Let's make this its own function.

```js
function field(f) {
  return function(obj) {
    return obj[f];
  };
}

function algorithm16(data) {
  var lastBar = 0;
  var lastBaz = 0;

  var foos = data.map(field('foo')).reduce(function(state, val) {
    if (val === '-999') {
      return {acc: state.acc.concat(0)};
    } else {
      return {acc: state.acc.concat(val)};
    }
  }, {acc: []}).acc.map(function(val) {
    return Math.abs(Number(val));
  });

  var bars = data.map(field('bar')).reduce(function(state, val) {
    if (val != null) {
      return {acc: state.acc.concat(val), last: val};
    } else {
      return {acc: state.acc.concat(state.last), last: state.last};
    }
  }, {acc: [], last: lastBar}).acc.map(function(val) {
    return Number(val);
  });

  var bazs = data.map(field('baz')).reduce(function(state, val) {
    if (val != null) {
      return {acc: state.acc.concat(val), last: val};
    } else {
      return {acc: state.acc.concat(state.last), last: state.last};
    }
  }, {acc: [], last: lastBaz}).acc.map(function(val) {
    return Number(val);
  });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

And make sure we're still good:

```js
var output16 = algorithm16(data);
console.log(output16);
//=>
// {
//   foos: [2, 10, 5, 8, 2, 8, 9, 6, 6, 0, 0, 6, 9, 0, 6, 5, 8, 0],
//   bars: [12, 4, 4, 2, 2, 17, 18, 14, 14, 18, 18, 12, 12, 20, 5, 15, 0, 9],
//   bazs: [6, 17, 17, 12, 9, 8, 8, 8, 8, 14, 9, 17, 12, 3, 6, 1, 10, 17]
// }
```

Next, each pipeline does some cleanup.
The `foo` pipeline does a slightly different cleanup than the others,
but we can still parameterize on this.

```js
function field(f) {
  return function(obj) {
    return obj[f];
  };
}

function cleanFoo(state, val) {
  if (val === '-999') {
    return {acc: state.acc.concat(0)};
  } else {
    return {acc: state.acc.concat(val)};
  }
}

function cleanNulls(state, val) {
  if (val != null) {
    return {acc: state.acc.concat(val), last: val};
  } else {
    return {acc: state.acc.concat(state.last), last: state.last};
  }
}

function algorithm17(data) {
  var lastBar = 0;
  var lastBaz = 0;

  var foos = data.map(field('foo')).reduce(cleanFoo, {acc: []})
    .acc
    .map(function(val) {
      return Math.abs(Number(val));
    });

  var bars = data.map(field('bar')).reduce(cleanNulls, {acc: [], last: lastBar})
    .acc
    .map(function(val) {
      return Number(val);
    });

  var bazs = data.map(field('baz')).reduce(cleanNulls, {acc: [], last: lastBaz})
    .acc
    .map(function(val) {
      return Number(val);
    });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

Now that we've pulled out the cleaning functions,
let's make them look more similar by reversing the exceptional check in `cleanNulls`.

```js
function field(f) {
  return function(obj) {
    return obj[f];
  };
}

function cleanFoo(state, val) {
  if (val === '-999') {
    return {acc: state.acc.concat(0)};
  } else {
    return {acc: state.acc.concat(val)};
  }
}

function cleanNulls(state, val) {
  if (val == null) {
    return {acc: state.acc.concat(state.last), last: state.last};
  } else {
    return {acc: state.acc.concat(val), last: val};
  }
}

function algorithm18(data) {
  var lastBar = 0;
  var lastBaz = 0;

  var foos = data.map(field('foo')).reduce(cleanFoo, {acc: []})
    .acc
    .map(function(val) {
      return Math.abs(Number(val));
    });

  var bars = data.map(field('bar')).reduce(cleanNulls, {acc: [], last: lastBar})
    .acc
    .map(function(val) {
      return Number(val);
    });

  var bazs = data.map(field('baz')).reduce(cleanNulls, {acc: [], last: lastBaz})
    .acc
    .map(function(val) {
      return Number(val);
    });

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

Lastly, each pipeline does some final preparation of the value.

```js
function field(f) {
  return function(obj) {
    return obj[f];
  };
}

function cleanFoo(state, val) {
  if (val === '-999') {
    return {acc: state.acc.concat(0)};
  } else {
    return {acc: state.acc.concat(val)};
  }
}

function cleanNulls(state, val) {
  if (val == null) {
    return {acc: state.acc.concat(state.last), last: state.last};
  } else {
    return {acc: state.acc.concat(val), last: val};
  }
}

function prepareFoo(val) {
  return Math.abs(Number(val));
}

function algorithm19(data) {
  var lastBar = 0;
  var lastBaz = 0;

  var foos = data.map(field('foo')).reduce(cleanFoo, {acc: []})
    .acc
    .map(prepareFoo);

  var bars = data.map(field('bar')).reduce(cleanNulls, {acc: [], last: lastBar})
    .acc
    .map(Number);

  var bazs = data.map(field('baz')).reduce(cleanNulls, {acc: [], last: lastBaz})
    .acc
    .map(Number);

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

Looking at each pipeline, we see that they're nearly identical.
This means we've boiled down our algorithm pretty much to its core.
Let's pull out the algorithm.

```js
function field(f) {
  return function(obj) {
    return obj[f];
  };
}

function cleanFoo(state, val) {
  if (val === '-999') {
    return {acc: state.acc.concat(0)};
  } else {
    return {acc: state.acc.concat(val)};
  }
}

function cleanNulls(state, val) {
  if (val == null) {
    return {acc: state.acc.concat(state.last), last: state.last};
  } else {
    return {acc: state.acc.concat(val), last: val};
  }
}

function prepareFoo(val) {
  return Math.abs(Number(val));
}

function algorithm(f, prep, last, clean, data) {
  return data.map(field(f)).reduce(prep, {acc: [], last: last}).acc.map(clean);
}

function algorithm20(data) {
  var lastBar = 0;
  var lastBaz = 0;

  var foos = algorithm('foo', cleanFoo, 0, prepareFoo, data);
  var bars = algorithm('bar', cleanNulls, lastBar, Number, data);
  var bazs = algorithm('baz', cleanNulls, lastBaz, Number, data);

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```


Another thing we can make better are these clean functions.
Let's try to make `cleanFoo` look more like `cleanNulls`.

```js
function cleanFoo(state, val) {
  if (val === '-999') {
    return {acc: state.acc.concat(0), last: 0};
  } else {
    return {acc: state.acc.concat(val), last: val};
  }
}

function cleanNulls(state, val) {
  if (val == null) {
    return {acc: state.acc.concat(state.last), last: state.last};
  } else {
    return {acc: state.acc.concat(val), last: val};
  }
}
```

We can see that what's happening in each case is that
some value is being appended to the end of `acc` and replacing `last`.

Let's pull this behavior out to its own function.

```js
function field(f) {
  return function(obj) {
    return obj[f];
  };
}

function newState(state, val) {
  return {acc: state.acc.concat(val), last: val};
}

function cleanFoo(state, val) {
  if (val === '-999') {
    return newState(state, 0);
  } else {
    return newState(state, val);
  }
}

function cleanNulls(state, val) {
  if (val == null) {
    return newState(state, state.last);
  } else {
    return newState(state, val);
  }
}

function prepareFoo(val) {
  return Math.abs(Number(val));
}

function algorithm(f, prep, last, clean, data) {
  return data.map(field(f)).reduce(prep, {acc: [], last: last}).acc.map(clean);
}

function algorithm21(data) {
  var lastBar = 0;
  var lastBaz = 0;

  var foos = algorithm('foo', cleanFoo, 0, prepareFoo, data);
  var bars = algorithm('bar', cleanNulls, lastBar, Number, data);
  var bazs = algorithm('baz', cleanNulls, lastBaz, Number, data);

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

And we can of course remove some of the cruft in both of those functions.

```js
function field(f) {
  return function(obj) {
    return obj[f];
  };
}

function newState(state, val) {
  return {acc: state.acc.concat(val), last: val};
}

function cleanFoo(state, val) {
  return newState(state, val === '-999' ? 0 : val);
}

function cleanNulls(state, val) {
  return newState(state, val == null ? state.last : val);
}

function prepareFoo(val) {
  return Math.abs(Number(val));
}

function algorithm(f, prep, last, clean, data) {
  return data.map(field(f)).reduce(prep, {acc: [], last: last}).acc.map(clean);
}

function algorithm22(data) {
  var lastBar = 0;
  var lastBaz = 0;

  var foos = algorithm('foo', cleanFoo, 0, prepareFoo, data);
  var bars = algorithm('bar', cleanNulls, lastBar, Number, data);
  var bazs = algorithm('baz', cleanNulls, lastBaz, Number, data);

  return {
    foos: foos, bars: bars, bazs: bazs,
  };
}
```

We can do some final movement of the program, and give it a better name

```js
function field(f) {
  return function(obj) {
    return obj[f];
  };
}

function newState(state, val) {
  return {acc: state.acc.concat(val), last: val};
}

function cleanFoo(state, val) {
  return newState(state, val === '-999' ? 0 : val);
}

function cleanNulls(state, val) {
  return newState(state, val == null ? state.last : val);
}

function prepareFoo(val) {
  return Math.abs(Number(val));
}

function algorithm(f, prep, last, clean, data) {
  return data.map(field(f)).reduce(prep, {acc: [], last: last}).acc.map(clean);
}

function cleanup(data) {
  return {
    foos: algorithm('foo', cleanFoo, 0, prepareFoo, data),
    bars: algorithm('bar', cleanNulls, 0, Number, data),
    bazs: algorithm('baz', cleanNulls, 0, Number, data),
  };
}
```

## Quick recap

So what have we gained here?
Well, there are just as many lines as before, so we didn't get a smaller codebase.
However, there are many things that actually came out of this refactoring.
At the very least we have the following:

* Each function has one purpose.
* Each function is pure.
* Each function is understandable by itself.
* Each function can be tested in isolation.

## Ramda refactoring

Wait, didn't I mention ramda earlier?
Where is that, and how can that help our situation?

We can start by replacing the `field` function with `prop` from ramda.

```js
function newState(state, val) {
  return {acc: state.acc.concat(val), last: val};
}

function cleanFoo(state, val) {
  return newState(state, val === '-999' ? 0 : val);
}

function cleanNulls(state, val) {
  return newState(state, val == null ? state.last : val);
}

function prepareFoo(val) {
  return Math.abs(Number(val));
}

function algorithm(f, prep, last, clean, data) {
  return data.map(R.prop(f)).reduce(prep, {acc: [], last: last}).acc.map(clean);
}

function cleanup(data) {
  return {
    foos: algorithm('foo', cleanFoo, 0, prepareFoo, data),
    bars: algorithm('bar', cleanNulls, 0, Number, data),
    bazs: algorithm('baz', cleanNulls, 0, Number, data),
  };
}
```

We can also remove the `prepareFoo` function by `compose`ing them with ramda.

```js
function newState(state, val) {
  return {acc: state.acc.concat(val), last: val};
}

function cleanFoo(state, val) {
  return newState(state, val === '-999' ? 0 : val);
}

function cleanNulls(state, val) {
  return newState(state, val == null ? state.last : val);
}

function algorithm(f, prep, last, clean, data) {
  return data.map(R.prop(f)).reduce(prep, {acc: [], last: last}).acc.map(clean);
}

function cleanup(data) {
  return {
    foos: algorithm('foo', cleanFoo, 0, R.compose(Math.abs, Number), data),
    bars: algorithm('bar', cleanNulls, 0, Number, data),
    bazs: algorithm('baz', cleanNulls, 0, Number, data),
  };
}
```

We can make `newState` simpler using `evolve` from ramda.

```js
function newState(state, val) {
  return R.evolve({acc: R.append(val), last: R.always(val)}, state);
}

function cleanFoo(state, val) {
  return newState(state, val === '-999' ? 0 : val);
}

function cleanNulls(state, val) {
  return newState(state, val == null ? state.last : val);
}

function algorithm(f, prep, last, clean, data) {
  return data.map(R.prop(f)).reduce(prep, {acc: [], last: last}).acc.map(clean);
}

function cleanup(data) {
  return {
    foos: algorithm('foo', cleanFoo, 0, R.compose(Math.abs, Number), data),
    bars: algorithm('bar', cleanNulls, 0, Number, data),
    bazs: algorithm('baz', cleanNulls, 0, Number, data),
  };
}
```

And we can make `newState` more explicit in its purpose by
reversing the order of the arguments and not mentioning state at all.

```js
function newState(val) {
  return R.evolve({acc: R.append(val), last: R.always(val)});
}

function cleanFoo(state, val) {
  return newState(val === '-999' ? 0 : val)(state);
}

function cleanNulls(state, val) {
  return newState(val == null ? state.last : val)(state);
}

function algorithm(f, prep, last, clean, data) {
  return data.map(R.prop(f)).reduce(prep, {acc: [], last: last}).acc.map(clean);
}

function cleanup(data) {
  return {
    foos: algorithm('foo', cleanFoo, 0, R.compose(Math.abs, Number), data),
    bars: algorithm('bar', cleanNulls, 0, Number, data),
    bazs: algorithm('baz', cleanNulls, 0, Number, data),
  };
}
```

Finally, we can take the `algorithm` and
convert it into an actual pipeline using `pipe` from ramda.

```js
function newState(val) {
  return R.evolve({acc: R.append(val), last: R.always(val)});
}

function cleanFoo(state, val) {
  return newState(val === '-999' ? 0 : val)(state);
}

function cleanNulls(state, val) {
  return newState(val == null ? state.last : val)(state);
}

function algorithm(f, prep, last, clean, data) {
  return R.pipe(
    R.map(R.prop(f)),
    R.reduce(prep, {acc: [], last: last}),
    R.prop('acc'),
    R.map(clean)
  )(data);
}

function cleanup(data) {
  return {
    foos: algorithm('foo', cleanFoo, 0, R.compose(Math.abs, Number), data),
    bars: algorithm('bar', cleanNulls, 0, Number, data),
    bazs: algorithm('baz', cleanNulls, 0, Number, data),
  };
}
```

Note that we pull the `acc` field out by using `prop` again in the pipeline.

## Final recap

We've reached a point where there isn't much else we can do to make the code better.
What did ramda give us that plain javascript couldn't?

* We got rid of two functions right off the bat.
* We made another function more explicit in what it does.
* Our algorithm became a functional pipeline, rather than a method chain.

The last point is important.
Many times the method chain is preferred.
People like the idea of how it reads.

There are some issues with method chaining.

First, the onus is on the library writer to provide all the possible behavior.
If the library writer does not provide some behavior,
the chain must either be broken and a function written that performs this behavior on its behalf,
or the object must be extended through either composition, inheritance, or some other means.

Another issue is a more specific case of the first.
If you wanted to move part of the method chain out to its own chain (maybe it's more understandable that way, or maybe you have lots of duplication),
then you have to either break the chain and write a function that does this,
or you have to extend the object again.

In either case, the change is not small, isolated, and self contained.

With a functional pipeline, these issues do not exist.

If a library writer forgets or does not forsee some behavior,
a user can extend it without it appearing to be different merely by creating a function.

If a user wants to break out part of the pipeline into its own pipeline,
they can do so with minimal effort because function composition is associative.
Meaning, no matter how you group the composition of functions, you always get the same result.

There are some important things to realize about all of this refactoring.

* Ramda didn't help us much in the bulk of the changes. Most of it was just plain old javascript.
* However, ramda gave us even more improvements over just plain old javascript, so clearly it's a worthwhile library.
* Removing mutation allowed us to find lots of similarities that were being hidden.
