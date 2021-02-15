---
title: Write library for any ECMAScript environment
toc: true
categories: frontend
date: 2021-02-15 16:13:36
update: 2021-02-15 16:13:36
lang: en
tags:
    - esmodule
    - deno
---

"It would be nice if my library can work across different JavaScript environments..." you thought.

Here is a guideline about how to achieve this goal. I'll list the most general principles first then explain them in detail and finally give you a recommended pattern to develop this kind of library.

To make the discussion easier, we will call the ability to _run on any ECMAScript environment_ as having **zero host dependency**.

> When we say _run on any ECMAScript environment_, we mean: users of the library can run this library on a modern ECMAScript environment (at least ES2015) that supports the ECMAScript module with proper setups (e.g. provide IO functions) without modifying the library code.

Restrictions:

1. Do not use anything that is not defined in the ECMAScript specification directly. (e.g. Web or Node APIs).

1. Do not use module specifier (`"./file.js"` in `import x from "./file.js"`) to refer to _any_ module.

1. Be aware of language features (`eval`, `Date.now()`, `import()`, `Math.random()`, etc...) are disabled in some special environments.

If you're strictly following the rules described above, you'll find that you're almost not able to develop anything. But this article is going to introduce a comfortable paradigm for developing libraries with zero host dependency.

<!-- more -->

I'll use an imaginary library called `use-delayed-effect` as an example and migrates this library step by step in this article. At the end of this article, this library will have zero host dependency.

```js
// constant.js
export const _delay = 1000;
// index.js
import { _delay } from "./constant";
import { useEffect } from "react";
export function useDelayedEffect(f, deps, delay = _delay) {
    useEffect(() => {
        const timer = setTimeout(f, delay);
        return () => clearTimeout(timer);
    }, [_delay, ...deps]);
}
```

# Host defined globals

## What thing I can't use?

Did you know that `setTimeout`, `console.log`, and many things you are familiar with are not part of ECMAScript specification but belong to the host (Node, Web, ...)? This means if an ECMAScript environment chooses not to implement it, it is still not violating the specification.

For example, you cannot use `setTimeout` in a [Worklet](https://developer.mozilla.org/en-US/docs/Web/API/Worklet).

Here is the list of things you can use safely. (Updated at 2021/02/14)

<details>
<summary>Values defined in the specification</summary>
<pre>
Infinity
NaN
undefined
globalThis
</pre>
</details>

<details>
<summary>Namespaces defined in the specification</summary>
<pre>
JSON
Math
Reflect
</pre>
</details>

<details>
<summary>Functions defined in the specification</summary>
<pre>
<del>eval</del> (don't use it)
<del>isFinite</del> (use Number.isFinite)
<del>isNaN</del> (use Number.isNaN)
parseFloat
parseInt
decodeURI
decodeURIComponent
encodeURI
encodeURIComponent
</pre>
</details>

<details>
<summary>Classes defined in the specification</summary>
<pre>
AggregateError
Boolean
BigInt
Date
Error
EvalError
FinalizationRegistry
<del>Function</del> (don't use it to construct new functions dynamically).
Map
Number
Object
Promise
Proxy
RangeError
ReferenceError
RegExp
Set
String
Symbol
SyntaxError
TypeError
URIError
WeakMap
WeakRef
WeakSet
</pre>
</details>

<details>
<summary>Array/TypedArray related classes defined in the specification</summary>
<pre>
Array
ArrayBuffer
BigInt64Array
BigUint64Array
DataView
Float32Array
Float64Array
Int8Array
Int16Array
Int32Array
Uint8Array
Uint8ClampedArray
Uint16Array
Uint32Array
</pre>
</details>

The most simple way to find out if you can use a thing is to run it in the [engine262](https://engine262.js.org/) (an ECMAScript engine written in ECMAScript).

> ⚠ Note engine262 [provides a host definition of `print` and `console`](https://github.com/engine262/engine262.github.io/blob/bcec156ef1c1d1a223bfcb6133e2cad5c0c90588/src/worker.js#L58-L98), which are not in the language.

Another way is to ask yourself, is it related to IO? If the answer is yes, it is a host defined global.

## But I need it!

You can receive those functionalities from the outside world.

### A "sys" object

TypeScript compiler uses this manner:

```js
export const sys = { base64encode: null };
// Do some feature detect and
// allow users to provide their implementation
```

Let's use the "sys object" way to refactor our library. We will add a `strange-environment-entry.js` for an imaginary environment that does not have `setTimeout` but has `Timer.createTimer`.

```js
// constant.js
export const _delay = 1000;
export const sys = { createTimer: null };

// core.js
import { _delay, sys } from "./constant";
import { useEffect } from "react";
export function useDelayedEffect(f, deps, delay = _delay) {
    useEffect(() => {
        return sys.createTimer(f, delay);
    }, [_delay, ...deps]);
}

// index.js for Node.JS and Web
import { sys } from "./constant";
sys.createTimer = (f, delay) => {
    const id = setTimeout(f, delay);
    return () => clearTimeout(id);
};
export * from "./core";

// strange-environment-entry.js
import { sys } from "./constant";
sys.createTimer = Timer.createTimer;
export * from "./core";
```

### "Factory pattern"

```js
export function createMyLib({ base64encode, base64decode }) {
    return {
        fn() {},
    };
}
```

Let's use the "factory pattern" way to refactor our library. We will add a `strange-environment-entry.js` for an imaginary environment that does not have `setTimeout` but has `Timer.createTimer`.

```js
// constant.js
export const _delay = 1000;

// core.js
import { _delay } from "./constant";
import { useEffect } from "react";
export function createDelayedEffect({ createTimer }) {
    return {
        useDelayedEffect(f, deps, delay = _delay) {
            useEffect(() => {
                return createTimer(f, delay);
            }, [_delay, ...deps]);
        },
    };
}

// index.js for Node.JS and Web
import { createDelayedEffect } from "./core";
export const { useDelayedEffect } = createDelayedEffect({
    createTimer(f, delay) {
        const id = setTimeout(f, delay);
        return () => clearTimeout(id);
    },
});

// strange-environment-entry.js
export const { useDelayedEffect } = createDelayedEffect({
    createTimer: Timer.createTimer,
});
```

Now our library is much more friendly if there is no `setTimeout` but some other timer functions.

# Import specifiers

```js
import sth from "path";
//              ~~~~~~
import("./file");
//     ~~~~~~~~
```

This string is what we called as import specifiers. Actually, in the language specification, it is just a meanless string. The meaning of this string is defined by the host too. Let's check out what import paths we have used in our library.

```js
import {} from "react";
import {} from "./constant";
import {} from "./core";
```

We already know that `react` import will not work on the browser (without an [import map](https://github.com/WICG/import-maps)). And for a normal web server, `./core` is wrong too. We should use `./core.js`.

Now we refactor our lib in the following way:

```js
// core.js
import { _delay } from "./constant.js";
export function createDelayedEffect({ createTimer, useEffect }) {
    return {
        useDelayedEffect(f, deps, delay = _delay) {
            useEffect(() => {
                return createTimer(f, delay);
            }, [_delay, ...deps]);
        },
    };
}

// index.js for Node.JS
import { createDelayedEffect } from "./core.js";
import { useEffect } from "react";
export const { useDelayedEffect } = createDelayedEffect({
    useEffect,
    createTimer(f, delay) {
        const id = setTimeout(f, delay);
        return () => clearTimeout(id);
    },
});

// web.js for Web
import { createDelayedEffect } from "./core.js";
import { useEffect } from "https://cdn.skypack.dev/react";
export const { useDelayedEffect } = createDelayedEffect({
    createTimer(f, delay) {
        const id = setTimeout(f, delay);
        return () => clearTimeout(id);
    },
    useEffect,
});

// strange-environment-entry.js
import { useEffect } from "runtime:react";
import { createDelayedEffect } from "./core";
export const { useDelayedEffect } = createDelayedEffect({
    createTimer: Timer.createTimer,
    useEffect,
});
```

Is that safe now? If you're not extremely cautious, the answer is yes. Most of the runtime supports relative path import with the correct extension. That's means `import './core.js'` is safe enough in practice. But not theoretically. The support of relative path import is not enforced by the specification. There is even no concept of a file in the spec.

If you're extremely cautious, there're two solutions:

## Use an ES module bundler

Use [rollup](https://rollupjs.org/) to bundle the core of your library and create entry files for each environment (Node, Web, Deno, ...) you want to make out-of-box support.

If we choose to refactor our library in this way, here is the result:

[./dist/core.js created by rollup](https://rollupjs.org/repl/?version=2.39.0&shareable=JTdCJTIybW9kdWxlcyUyMiUzQSU1QiU3QiUyMm5hbWUlMjIlM0ElMjJtYWluLmpzJTIyJTJDJTIyY29kZSUyMiUzQSUyMmltcG9ydCUyMCU3QiUyMF9kZWxheSUyMCU3RCUyMGZyb20lMjAnLiUyRmNvbnN0YW50LmpzJyU1Q25leHBvcnQlMjBmdW5jdGlvbiUyMGNyZWF0ZURlbGF5ZWRFZmZlY3Qoc3lzKSUyMCU3QiU1Q24lMjAlMjAlMjAlMjBjb25zdCUyMCU3QiUyMGNyZWF0ZVRpbWVyJTJDJTIwdXNlRWZmZWN0JTIwJTdEJTIwJTNEJTIwc3lzJTNCJTVDbiUyMCUyMCUyMCUyMHJldHVybiUyMCU3QiU1Q24lMjAlMjAlMjAlMjAlMjAlMjAlMjAlMjB1c2VEZWxheWVkRWZmZWN0KGYlMkMlMjBkZXBzJTJDJTIwZGVsYXklMjAlM0QlMjBfZGVsYXkpJTIwJTdCJTVDbiUyMCUyMCUyMCUyMCUyMCUyMCUyMCUyMCUyMCUyMCUyMCUyMHVzZUVmZmVjdCgoKSUyMCUzRCUzRSUyMCU3QiU1Q24lMjAlMjAlMjAlMjAlMjAlMjAlMjAlMjAlMjAlMjAlMjAlMjAlMjAlMjAlMjAlMjByZXR1cm4lMjBjcmVhdGVUaW1lcihmJTJDJTIwZGVsYXkpJTNCJTVDbiUyMCUyMCUyMCUyMCUyMCUyMCUyMCUyMCUyMCUyMCUyMCUyMCU3RCUyQyUyMCU1Ql9kZWxheSUyQyUyMC4uLmRlcHMlNUQpJTNCJTVDbiUyMCUyMCUyMCUyMCUyMCUyMCUyMCUyMCU3RCUyQyU1Q24lMjAlMjAlMjAlMjAlN0QlM0IlNUNuJTdEJTIyJTJDJTIyaXNFbnRyeSUyMiUzQXRydWUlN0QlMkMlN0IlMjJuYW1lJTIyJTNBJTIyY29uc3RhbnQuanMlMjIlMkMlMjJjb2RlJTIyJTNBJTIyZXhwb3J0JTIwY29uc3QlMjBfZGVsYXklMjAlM0QlMjAxMDAwJTNCJTIyJTJDJTIyaXNFbnRyeSUyMiUzQWZhbHNlJTdEJTVEJTJDJTIyb3B0aW9ucyUyMiUzQSU3QiUyMmZvcm1hdCUyMiUzQSUyMmVzJTIyJTJDJTIybmFtZSUyMiUzQSUyMm15QnVuZGxlJTIyJTJDJTIyYW1kJTIyJTNBJTdCJTIyaWQlMjIlM0ElMjIlMjIlN0QlMkMlMjJnbG9iYWxzJTIyJTNBJTdCJTdEJTdEJTJDJTIyZXhhbXBsZSUyMiUzQW51bGwlN0Q=)

```js
// index.js for Node.JS
import { useEffect } from "react";
import { createDelayedEffect } from "./dist/core.js";
export const { useDelayedEffect } = createDelayedEffect({ ... });

// web.js for Web
import { createDelayedEffect } from "./dist/core.js";
import { useEffect } from "https://cdn.skypack.dev/react";
export const { useDelayedEffect } = createDelayedEffect({ ... });

// strange-environment-entry.js
import { useEffect } from "runtime:react";
import { createDelayedEffect } from "./dist/core.js";
export const { useDelayedEffect } = createDelayedEffect({ ... });
```

## No path specifier

In this way, we treat all relative imports the same way as external dependencies.

```js
// core.js
export function createDelayedEffect({ useEffect, createTimer, contant }) {
    // ...
}

// index.js for Node.JS
import { useEffect } from "react";
import { createDelayedEffect } from "./core.js";
import * as constant from "./constant.js";
export const { useDelayedEffect } = createDelayedEffect({
    constant,
    useEffect,
    createTimer(f, delay) {
        const id = setTimeout(f, delay);
        return () => clearTimeout(id);
    },
});

// web.js for Web
import { createDelayedEffect } from "./core";
import { useEffect } from "https://cdn.skypack.dev/react";
import * as constant from "./constant.js";
export const { useDelayedEffect } = createDelayedEffect({
    createTimer(f, delay) {
        const id = setTimeout(f, delay);
        return () => clearTimeout(id);
    },
    useEffect,
    constant,
});

// strange-environment-entry.js
import { useEffect } from "runtime:react";
import { createDelayedEffect } from "./core.js";
import * as constant from "./constant.js";
export const { useDelayedEffect } = createDelayedEffect({
    createTimer: Timer.createTimer,
    useEffect,
    constant,
});
```

As you can notice we're not using any import in `constant.js` and `core.js`. Keep your core logics export-only is very annoying, I suggest using a bundler if the project is big.

# About TypeScript and Deno

If your library is written in TypeScript and you want to ship TypeScript directly to Deno users, it's not easy.

Deno follows the same module resolution strategy as Web browsers, which means you must add `.ts` extension.

```ts
import {} from "./core.ts";
```

> An import path cannot end with a '.ts' extension. Consider importing './core.js' instead.(2691)

Unfortunately, the TypeScript compiler will complain about the error above, and it emitting ECMAScript files are keeping the .ts extension.

Although there is some hacky way (e.g. using a transformer) to solve this problem, I suggest using [rollup to bundle files](#Use-an-ES-module-bundler) and use [rollup-plugin-dts](https://www.npmjs.com/package/rollup-plugin-dts) to bundle type definitions into one single file, finally, add a triple-slash comment at top of the output file.

```ts
/// <reference types="./output.d.ts" />
```

Here is [an example of the output JS file](https://cdn.jsdelivr.net/npm/async-call-rpc@5.0.0/out/base.min.js) that links to [the type definition](https://cdn.jsdelivr.net/npm/async-call-rpc@5.0.0/out/base.min.d.ts).

> ⚠ The Deno module resolution strategy applies for `.d.ts` too so please make sure your `.d.ts` is an export-only file or it will not be able to share with normal TypeScript code.

# Other notes

-   `Error.prototype.stack` is not standard.
-   `import.meta` is in the ES standard but `import.meta.url` is not.
-   `eval`, `new Function()`, etc are not available under a strict [Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP).
-   [dynamic import() does not work in Service Worker](https://github.com/w3c/ServiceWorker/issues/1356)

## [SES (Secure ECMAScript)](https://github.com/tc39/proposal-ses) and [XS](https://blog.moddable.com/blog/secureprivate/)

-   No `Math.random()`
-   No `Date.now()`
-   `new Date()` will throw a TypeError
-   `Date(...)` will throw a TypeError
-   No `RegExp` static methods
-   All builtin function/classes are frozen and not extensible

# End of the tour

Now our library has zero host dependency. If someone wants to use this library in a special environment, they can import the core file and create their instance without patching your library.

```js
// some unusual env
// use-delayed-effect-wrapper.js
import { createDelayedEffect } from "lib:use-delayed-effect/core.js";
import * as constant from "lib:use-delayed-effect/contant.js";
export const { useDelayedEffect } = createDelayedEffect({
    createTimer,
    constant,
});
```

You don't have to follow all recommendations dogmatically. You can use `Math.random()` without worrying about compatibility with SES. Aware of those platform exists and make smart decisions, you are the library author after all. If you choose to not support some environments, that's not a fault. But if you _do_ want to support _any_ possible ES environment, this article can help.

# Ad time

There are some libraries I designed with zero host dependency in mind.

-   [async-call-rpc](https://github.com/Jack-Works/async-call-rpc): A JSON RPC server and client.
-   [react-refresh-typescript](https://github.com/Jack-Works/react-refresh-transformer/tree/main/typescript)
-   [ttypescript-browser-like-import-transformer](https://github.com/Jack-Works/ttypescript-browser-like-import-transformer)
