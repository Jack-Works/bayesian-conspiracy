---
title: Generate type definition for matrix-js-sdk
date: 2020-05-01 08:13:04
updated: 2020-05-01 08:13:04
tags:
    - codegen
lang: en
toc: true
categories: frontend
cover: https://user-images.githubusercontent.com/5390719/69128591-a0d10f00-0ae7-11ea-8456-5b5c3f28aac0.png
---

[The generator](http://github.com/jack-Works/generate-matrix-js-sdk-type/) and [the generated type definitions](https://github.com/Jack-Works/matrix-js-sdk-type) are available at GitHub.

On Feb 2020, [@huan](https://github.com/huan) contribute his type definition to DefinitelyTyped [@types/matrix-js-sdk](https://www.npmjs.com/package/@types/matrix-js-sdk), you can compare and choose one.

---

# Motivation

Matrix is a decentralized IM protocol, the [matrix-js-sdk](https://www.npmjs.com/package/matrix-js-sdk) is the official SDK for this protocol. Back time to Nov 2019, I need to integrate this SDK into my working project.

As a TypeScript fan, work with a huge library with totally no typing is a pain so I was going to look for @types/matrix-js-sdk. Unfortunately, there is none. I also looked for the GitHub issues and found [[TypeScript] Typing Support](https://github.com/matrix-org/matrix-js-sdk/issues/983). @huan supplied a hand-written type definition, I tried it in our project, and found too few APIs is typed, and somehow becomes useless.

The matrix-js-sdk is well-documented with JSDoc, and in [Typescript 3.7](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#--declaration-and---allowjs) there is a new feature that allows developers to generate declaration file for JS projects. Therefore, I decided to generate one.

<!-- more -->

# Codegen

The very first try is to run the compiler to see the output. The result leads to tons of `any`.
<img src="https://user-images.githubusercontent.com/5390719/69128591-a0d10f00-0ae7-11ea-8456-5b5c3f28aac0.png" alt="The first try" />
There are three main reasons leads to the "any" hell.

## Mix the use of CommonJS and ES Module

> ⚠ Outdated: In Jan 2020, the matrix team has moved to ES6 modules in their 4.0 release.

```ts
import { something } from "./files";
const promise = require("bluebird");
```

Reason: if there are any `import` or `export` declaration in the file, the compiler will consider the file as ES Module. `require` function only resolve type definition when the file is CommonJS format.

Export is same the case, once there is `import` or `export` in the file, `module.exports` no longer considered as the export of this file.

### Solution: Write a codemod to consistent module style

It transforms the code into the following steps.

1.  `import x from 'y'` (default import) becomes `const x = require('y')`
2.  `import * as x from 'y'` (namespace) becomes `const x = require('y')`
3.  `import { x as y, a as b } from 'y'` (named import) becomes `const {x as y, a as b} = require('y')`
4.  Call the TypeScript language service programmatically, apply the code fix `File is a CommonJS module; it may be converted to an ES6 module. ts(80001)`
5.  Traverse the AST, if there are any require call left (e.g. in an `if` block), collect them, give them a temporary name, and add an ES import in the file.

!> Warning: this code transformation is not equal to its origin semantics. We only need type information here, so it is safe to do the incorrect transform.

After those 5 steps, all files in the project becomes ES Modules and become friendly to static analysis.

## ES5 style class definition

In the codebase, matrix team wrote classes in ES5 style. TypeScript cannot generate good type for it.

**Solution**: Apply all the code fix `This constructor function may be converted to a class declaration. ts(80002)`

But there is more work to do other than the solution above, matrix-js-sdk use a pattern that TypeScript language service cannot upgrade to ES6, for example:

```ts
function X() {}
X.prototype = {
    method(): {},
}
```

I created a PR to TypeScript [to support this pattern](https://github.com/microsoft/TypeScript/pull/35219)

Finally, I did string replacement to patch the source code before I feed them to the compiler.

## JSDoc style type import

In the JSDoc world, developers use comments to define and reference modules, the type analyzer does not recognize the JSDoc style module.

```ts
/** @module my/module */
/** @types {module:another/module.SomeType} */
const x = something;
```

To make the type concise, I implemented JSDoc module resolution and transform them into code that TypeScript can analyze.
A library called doctrine can parse JSDoc for me.

1. Access each file to get the mapping between the JSDoc module and the real file path.
2. Recursively transform the JSDoc AST into a TypeScript type, meanwhile, record all usage of JSDoc type reference.
3. Replace JSDoc type reference into a generated name.
4. Add corresponding ES imports to bind the actual type to the transform in the step 3.

```ts
import { SomeType } from "./another/module";
/** @types { SomeType } */
const x = something;
```

# Other trivia

## import Promise from 'bluebird'

> The matrix team has switched to ES native Promise

This library uses the famous bluebird for Promise, which is not friendly to generate async function signatures so I use a code mod to remove it, therefore the output code will use the ES6 Promise as its type.

## Sequela of module system consistency hack

Due to the code mod mentioned above, the compiler will generate code like this.

```ts
import X from "./path";
const _X = X;
export { _X as X };
```

In most cases, it works well, but if `X` is a class, this kind of re-export will make it no longer become a type. In another word, the symbol `X` now become `typeof _X` and `const x: X = new X()` becomes invalid.

Further, when developer import the generated types as an external module, TypeScript will complain `Declaration emit for this file requires using private name 'X' from module`

The solution is simple, I wrote a code mod on the generated type definitions to transform the content above into

```ts
export { X };
```

## Other issues (all solved)

-   [TypeScript#37703: Bad JSDoc comments leads to generate invalid declaration file](https://github.com/microsoft/TypeScript/issues/37703)

-   [TypeScript#35932: Wrong declaration file generated for JS + subclass](https://github.com/microsoft/TypeScript/issues/35932)

-   [TypeScript#35228: Crashes on module.exports.Class.prototype pattern](https://github.com/microsoft/TypeScript/issues/35228)

# Next step

matrix-js-sdk itself is also migrating from ES5 to ES6 (and to TypeScript) therefore the code generation work will be easier in the future. In the end, matrix-js-sdk will ship its high-quality type definition and I can archive it.

-   Matrix-js-sdk moves to ES6 Module in 4.0.0 (Jan 2020)
-   A PR [rewrite all classes to ES6 class](https://github.com/matrix-org/matrix-js-sdk/pull/1229)
