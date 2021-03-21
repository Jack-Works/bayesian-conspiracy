---
title: 'Memo: Transfrom Do Expression'
toc: true
categories: frontend
date: 2021-03-21 15:58:12
update: 2021-03-21 15:58:12
lang: en
tags:
---

This is a memo to me on how to implement the down level compiling of
[ECMAScript proposal Do Expression](https://github.com/tc39/proposal-do-expressions/) and
[Async do Expression](https://github.com/tc39/proposal-async-do-expressions) in
[TypeScript](https://github.com/microsoft/TypeScript/pull/42437).

Need to be reviewed to make sure I'm not missing anything.

[Report error](https://github.com/tc39/proposal-do-expressions/issues/63)

<!-- more -->

# Part 1: Tracking completion values

First, add a temp var (let's call it `_C`) to store the completion value.

For every _ExpressionStatement_ `E` in the do expression block, replace it with `_C = _E`.

## Try statements

```js
try {
    statements
} catch {
    statements
} finally {
    statements
}
// into
try {
    _C = undefined
    tracked_statements
} catch {
    tracked_statements
} finally {
    statements
}
```

## If and Switch statements

```js
if (expr) { statements; }
// into
if (((_C = undefined), expr)) { tracked_statements; }

switch (expr) { statements; }
// into
switch (((_C = undefined), expr)) { tracked_statements; }
```

## Skipped statements

The following syntactic structures never contribute to the completion value of the do expression therefore not tracking
for completion value insides it.

-   _ClassLike_
-   _FunctionLike_
-   Any _Declaration_
-   Any for loops
-   _Finally_ block in `try-finally`

## Other AST elements

Recursively visit them to track the completion value deeply.

## Optimization

-   For continuous _ExpressionStatement_ I should only replace the last one.
-   Maybe I should only track the last meaningful _ExpressionStatement_ in every possible code branch.

## Example

```js
// For code
const x = do {
    let a = Math.random()
    a * a
}
// It should emit
var _CompletionValue
const x =
    ((() => {
        let a = Math.random()
        _CompletionValue = a * a
    })(),
    _CompletionValue)
```

# Part 2: Create the correct wrapper for do expression

There're 5 control flows I need to cover:

-   `await`
-   `yield`
-   `break`
-   `continue`
-   `return`

## await

await is valid in the following cases:

1. It is an _async do expression_
2. It is using _Top Level Await_
3. It is inside the _Await_ context

For case 2 and 3, await is valid to use, therefore I transfrom them as:

```js
const x = do { ... }
// into
const x = await ((async () => { ... })(), _CompletionValue)
```

For case 1, just create an async IIFE and returning the Promise.

## yield

yield is only valid in normal do expression in a _Yield_ context.

Note there is no arrow function version of generator syntax so I need to take care of the this value.

```js
const x = (yield* (function*() { ... }).call(this)), _CompletionValue
```

## await+yield

It is only valid in a normal do expression in both _Yield_ and _Await_ context.

```js
const x = (yield* (async function*() { ... }).call(this)), _CompletionValue
```

This should be enough. `yield*` should delegate both _Yield_ and _Await_ to the inner function.

## break/continue/return

This is the tricky part of the transform.

There is no way to `break/continue/return` across the function boundary and I'll use exceptions to do this.

Let's talk about `return` first because it's only valid in a _Return_ context.

So for return in a do expression, it should generate the code in the following way:

Note this is not an in-place transform, it will transform the containing function entirely to make sure the variable
scopes etc.

```js
function a() {
    const x = do {
        const a = 1
        return a
    }
}
// into
function a() {
    var _CompletionValue, _ControlFlowType, _ControlFlowValue
    try {
        const x =
            ((() => {
                const a = 1
                ;(_ControlFlowType = "return"), (_ControlFlowValue = a), null._
            })(),
            _CompletionValue)
    } catch (e) {
        if (_ControlFlowType == "return") return _ControlFlowValue
        throw e
    }
}
```

## break/continue

This part is the same as the `return` case but works for _LabeledStatement_, _SwitchStatement_ and For loops.

I guess the following transform are safe. Please point out if I'm wrong.

### LabeledStatement

```js
outer: {
    inner: {
        const val = (do {
            // TLA
            const val = await Math.random();
            if (val > 0.8) break inner;
            if (val > 0.6) break outer;
            1;
        })
    }
    const x = do { if (quit) break outer; 1; }
}
// Into
var _CompletionValue, _ControlFlowType, _ControlFlowValue, _CompletionValue2, _ControlFlowType2, _ControlFlowValue2
outer: {
    inner: {
        try {
            try {
                const val = ((await async (() => {
                    // TLA
                    const val = await Math.random();
                    if (val > 0.8) ((_ControlFlowType = "break"), (_ControlFlowValue = "inner"), null._);
                    if (val > 0.6) ((_ControlFlowType = "break"), (_ControlFlowValue = "outer"), null._);
                    _CompletionValue = 1;
                })()), _CompletionValue);
            } catch(e) {
                if (_ControlFlowType == "break") {
                    if (_ControlFlowValue == "inner") break inner
                    if (_ControlFlowValue == "outer") break outer
                }
                throw e
            }
            const x = (() => {
                if (quit) ((_ControlFlowType2 = "break"), (_ControlFlowValue2 = "outer"), null._);
                _CompletionValue2 = 1;
            })(), _CompletionValue2
        } catch (e) {
            if (_ControlFlowType2 == "break") {
                if (_ControlFlowValue2 == "outer") break outer
            }
            throw e
        }
    }
}
```

### Switch and for loops

I guess it basically the same as LabeledStatement so I'm not going to transfrom it by hand.
