# Double-Ended Iterator and Destructuring

Stauts: Stage 1

Author: HE Shi-Jun (hax)

Champion: HE Shi-Jun (hax)

## Motivation

Python and Ruby support `(first, *rest, last) = [1, 2, 3, 4]`, CoffeeScript supports `[first, rest..., last] = [1, 2, 3, 4]`, and Rust supports `[first, rest @ .., last] = [1, 2, 3, 4]`, all resulting in `first` be `1`, `last` be `4`, and `rest` be `[2, 3]`. But [surprisingly](https://stackoverflow.com/questions/33064377/destructuring-to-get-the-last-element-of-an-array-in-es6) `[first, ...rest, last] = [1, 2, 3, 4]` doesn't work in JavaScript.

And in some cases we really want to get the items from the end, for example getting `matchIndex` from [String.prototype.replace when using a function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace#Specifying_a_function_as_a_parameter):

```js
string.replace(pattern, (fullMatch, ...submatches, matchIndex, fullString) => {
  // `matchIndex` is always the second to last param (the full string is the last param).
  // There may be many submatch params, depending on the pattern.
})
```

A simple solution is making `let [first, ...rest, last] = iterable` to work as

```js
let [first, ...rest] = iterable
let last = rest.pop()
```

The concern is it requires saving all items in a `rest` array, although you may only need `last`. A possible mitigation is supporting `[..., last] = iterable` which saves the memory of `rest`, but you still need to consume the entire iterator. In the cases where `iterable` is a large array or something like `Number.range(1, 100000)`, it's very inefficient. And in case like `let [first, ..., last] = repeat(10)` (suppose `repeat` is a generator returns infinite sequence of a same value), theoretically both `first` and `last` could be `10`.

## Possible solution

Instead of the simple solution, we could introduce the double-ended iterator (like Rust std::iter::DoubleEndedIterator). A double-ended iterator could be consumed from both ends.

```js
let a = [1, 2, 3, 4, 5, 6]
let deiter = a.values() // suppose values() would be upgraded to return a double-ended iterator
deiter.next() // {value: 1}
deiter.next() // {value: 2}
deiter.next('back') // {value: 6}
deiter.next() // {value: 3}
deiter.next('back') // {value: 5}
deiter.next('back') // {value: 4}
deiter.next('back') // {done: true}
deiter.next() // {done: true}
```

With double-ended iterators, `let [a, b, ..., c, d] = iterable` would roughly work as

```js
let iter = iterable[Symbol.deIterator]()
let a = iter.next().value
let b = iter.next().value
let d = iter.next('back').value
let c = iter.next('back').value
iter.return()
```

## Generator

To implement double-ended iterator in userland, we could use a generator with the [`function.sent` feature](https://github.com/tc39/proposal-function.sent).

```js
Array.prototype.values = function *values() {
  // only for demo, for real upgrading of Array.prototype.values 
  // to double-ended iterator, it need to deal with the edge cases 
  // of mutating the array while iterating
  for (let start = 0, end = this.length; start < end;) {
    if (function.sent === 'back') yield this[--end]
    else yield this[start++]
  }
}
```

## Iterator helpers and reverse iterator

Double-ended iterator could have some extra [iterator helpers](https://github.com/tc39/proposal-iterator-helpers) like `reversed` and `reduceRight`.

```js
DoubleEndedIterator.prototype.reversed = function *reversed() {
  for (;;) {
    let result
    if (function.sent === 'back') result = this.next()
    else result = this.next('back')
    if (result.done) return result.value
    else yield result.value
  }
}
```

We could also easily have a default implementation for [reverse iterator](https://github.com/tc39/proposal-reverseIterator) if the object already supports double-ended iterator.

```js
Object.assign(X.prototype, {
  *[Symbol.reverseIterator]() {
    const iter = this[Symbol.deIterator]()
    for (;;) {
      let result
      if (function.sent === 'back') result = iter.next()
      else result = iter.next('back')
      if (result.done) return result.value
      else yield result.value
    }
  }
)
```

## FAQ

### I like the idea of allowing `...` in the middle of the destructuring pattern, but why introduce "double-ended" iterator?

Because JavaScript `[first, ...rest] = sequence` destructuring is based on iterable protocol, so we should make `[first, ...rest, last] = sequence` also based on iterable protocol. And `[a, b, ...rest, c, d, e] = sequence` could be perfectly interpreted as "take first two elements from the sequence, then take last three, and the rest", aka. allowing take elements from the the other end of a sequence, which conceptually same as what "double-ended" mean in the common data structure [deque](https://en.wikipedia.org/wiki/Double-ended_queue).  Note JavaScript already have some Array APIs behave as double-ended: 'indexOf/lastIndexOf, reduce/reduceRight, find/findLast', etc. So generalizing the concept could increase the consistency of all APIs (include user-land libraries) which may based on similar abstraction. See [Optional Mechanisms for Double-ended Destructructing](https://github.com/tc39/proposal-deiter/blob/main/why-deiter.md) for further analysis about the consideration of performance, mental burden and design cost.

### How could a iterator/generator move back to the previous status?

It's not "move back" or "step back", it's "consume the next value from the other end" or "shorten range of values from the other end". This is why it's `next('back')` not `previous()`. 

There are two concepts very easy to confuse, _bidirectional_ vs. _double-ended_. Bidirectional means you can invoke `next()` (move forward) or `previous()` (move backward). Double-ended means you can invoke `next()` (take the value from the front-end) or `next('back')` (take the value from the back-end).

### What is "double-ended", how it differ to "bidirectional"?

To help understand the concepts, you could imagine you use cursors point to positions of a sequence and get value at the position. Normal iteration need only one cursor, and initally the cursor is at the most left side of the sequence. You are only allowed to move the cursor to right direction and get the value of the position via `next()`. Bidrectional means you could also move the cursor to left direction via `previous()`, so go back to the previous position of the sequence, and get the value (again) at the position. 

Double-ended means you have **two** cursors and initally one is at the most left side and can only move to right direction, the other is at most right side and can only move to left direction. So you use `next()` move the first cursor to right and get the value at its position, use `next('back')` move the second cursor to left and get the value at its position. If two cursors meet the same postion, the sequence is totally consumed. 

You could find these two concept are actually orthogonal, so theorcially we could have both bidirectional and double-ended. So `next()`/`previous()` move the first cursor right/left, `next('back')/`previous('back')` move the second cursor left/right.

Note, even these two things could coexist, bidirectional is **not** compatible with JavaScript iterator protocol, because iterators are one-shot consumption, and produce `{done: true}` if all values are consumed, and it is required that `next()` always returns `{done: true}` after that, but `previous()` actually require to restore to previous, undone state. 

## Prior art
- Python [iterable unpacking](https://www.python.org/dev/peps/pep-3132/)
- Ruby [array decomposition](https://docs.ruby-lang.org/en/2.7.0/doc/syntax/assignment_rdoc.html#label-Array+Decomposition)
- CoffeeScript [destructuring assignment with splats](https://coffeescript.org/#destructuring)
- Rust [subslice pattern](https://rust-lang.github.io/rfcs/2359-subslice-pattern-syntax.html)
- Rust [std::iter::DoubleEndedIterator](https://doc.rust-lang.org/std/iter/trait.DoubleEndedIterator.html)
- Rust [Macro improved_slice_patterns::destructure_iter](https://docs.rs/improved_slice_patterns/2.0.1/improved_slice_patterns/macro.destructure_iter.html)

## Previous discussions
- https://github.com/tc39/proposal-array-last/issues/31
- https://github.com/tc39/proposal-reverseIterator/issues/1
- https://es.discourse.group/t/bidirectional-iterators/339

## Old discussions
- https://esdiscuss.org/topic/early-spread-operator
- https://esdiscuss.org/topic/parameter-lists-as-arguments-destructuring-sugar#content-3
- https://mail.mozilla.org/pipermail/es-discuss/2012-June/023353.html
- http://web.archive.org/web/20141214094119/https://bugs.ecmascript.org/show_bug.cgi?id=2034
- https://esdiscuss.org/topic/rest-parameter-anywhere
- https://esdiscuss.org/topic/rest-parameters
- https://esdiscuss.org/topic/strawman-complete-array-and-object-destructuring
- https://esdiscuss.org/topic/an-update-on-rest-operator
