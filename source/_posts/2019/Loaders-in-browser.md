---
title: Loaders in the browser
date: 2019-10-20 17:10:15
updated: 2019-10-20 17:10:15
tags:
    - transformer
    - service worker
lang: en
toc: true
categories: frontend
---

[Demo: Loader with ESModule example](https://github.com/Jack-Works/loader-with-esmodule-example)

In a frontend group, there was a discussion about how to use a template (like a Vue template or [efml](https://ef.js.org) template) without a bundler directly.

```js
import markdown from "./markdown.md";
```

Unfortunately, there is no proposal such as a custom import handler.
(The [import-map](https://github.com/WICG/import-maps) can't load other types than JS)

TLDR: `import.meta.url` is the core part.

<!-- more -->

<i-exp status="success"></i-exp>

# Currently blocked by

-   [ ] [Proposal: Top level await](https://github.com/tc39/proposal-top-level-await#implementations)
-   [ ] [Crbugs 824647: Implement ES Modules for service workers](https://crbug.com/824647)

# `import.meta.url`

The very first demo is a simple [Markdown loader](https://github.com/Jack-Works/loader-with-esmodule-example/commit/dcea9100956729df37de6de19fb539d86e65ec6a).

```ts
// index.js
import article from "./markdown-loader.js?src=./article.md";

document.body.appendChild(document.createElement("div")).appendChild(article);
```

```ts
// markdown-loader.js
import { Remarkable } from "https://cdn.skypack.dev/remarkable/v2";

const src = new URL(import.meta.url).searchParams.get("src");
const container = document.createElement("p");

fetch(src)
    .then((x) => x.text())
    .then((x) => new Remarkable().render(x))
    .then((x) => (container.innerHTML = x));

export default container;
```

Then a [CSS Loader](https://github.com/Jack-Works/loader-with-esmodule-example/commit/5b55f2c7dec75c0c84736fddda55725d0166c509) which is a polyfill for the [CSS Modules proposal](https://gist.github.com/developit/689aa4415bd688f3fce923cb8ae9abe7).

```ts
const src = new URL(import.meta.url).searchParams.get("src");
const container = new CSSStyleSheet();

fetch(src)
    .then((x) => x.text())
    .then((x) => container.replace(x));

export default container;
```

Then a [JSON Loader](https://github.com/Jack-Works/loader-with-esmodule-example/commit/7557344e2036f1e0868974a848a4736605f2c884) for the [JSON Module proposal](https://github.com/whatwg/html/issues/4315), a problem is encountered.

The content must be loaded synchronous, or it must be put in a container.

The container in the Markdown case, is an `HTMLElement`, in the CSS case, it is a `CSSStyleSheet` object ([Constructable Stylesheet](https://developers.google.com/web/updates/2019/02/constructable-stylesheets)).

What container should the JSON Module Loader use? A Promise? A getter?

```ts
export default new Promise(...)
// or
export let isJSONReady = false
export let JSON = undefined
```

Not good, developers except to `import JSON from './json-loader.js?src=./file.json'` to get the plain JSON object.

The answer is synchronous XHR.

> ❌ **Do not use synchronous XHR in production.**

> ❔ After the [Top-level await](https://github.com/tc39/proposal-top-level-await#implementations) shipped, this limit will be resolved.

```ts
const src = new URL(import.meta.url).searchParams.get("src");

const req = new XMLHttpRequest();
req.open("GET", src, false);
req.send(null);
export default JSON.parse(req.responseText);
```

# Next step: Use Service Worker to compile the file

By using the service worker it will be easier to transform a file to JavaScript Module because it can generate the output asynchronously.

Here is [the first example](https://github.com/Jack-Works/loader-with-esmodule-example/commit/a2bd6a8b26ce6c90e8bebc589780d415ab6f3eba) of the service worker transformed a CSS file into the equivalent JavaScript Module

```ts
addEventListener("fetch", (e) => {
    const url = new URL(e.request.url);
    if (url.pathname === "/css-module-loader.js") {
        const request = fetch(url.searchParams.get("src"));
        const css = request
            .then((x) => x.text())
            .then((x) => {
                const headers = new Headers(request.headers);
                const cssModule = `let s = new CSSStyleSheet()
s.replace(${JSON.stringify(x)})
export default s`;
                headers.set("content-type", "application/javascript");
                headers.set("content-length", cssModule.length);
                return new Response(cssModule, { headers });
            });
        e.respondWith(css);
    }
});
```

Now, if the browser has no service worker installed, it will run the ESModule `/css-loader.js?src=./file.css` and the code in the `css-loader.js` will provide the CSS content.

If the browser has a service worker installed, the worker will transform the CSS in the worker and provide it directly.

Next step the service worker [becomes more general](https://github.com/Jack-Works/loader-with-esmodule-example/commit/f55b52509f3b8de77fb5ea3d7737ccb4aec808da). It becomes easier to extend. A "Loader" is defined:

```ts
class Loader {
    /**
     * @param {string} fallbackURL The fallback URL
     * @param {(res: Response) => Promise<string>} transpiler The transpiler
     */
    constructor(fallbackURL, transpiler) {
        this.fallbackURL = fallbackURL;
        this.transpiler = transpiler;
    }
    /** @type {Loader[]} */
    static loaders = [];
    /** @param {Loader} loader */
    static add(loader) {
        this.loaders.push(loader);
    }
}
Loader.add(
    new Loader(
        "/css-module-loader.js",
        async (res) =>
            `const container = new CSSStyleSheet()
container.replace(${JSON.stringify(await res.text())})
export default container`
    )
);
```

By this abstraction, it is easier to create a new loader for a new type of file.

The [markdown loader in Service Worker](https://github.com/Jack-Works/loader-with-esmodule-example/commit/fb65dec2031557a45684294e2bf2d8c0510a2c4f):

```ts
Loader.add(
    new Loader(
        "/markdown-loader.js",
        async (res) => `const container = document.createElement('p')
container.innerHTML = ${JSON.stringify(marked(await res.text()))}
export default container`
    )
);
```

# TypeScript loader

?> Limitation: There is no way to create exports dynamically in ESModule. So it is impossible to translate the `export` declaration in the entry file. Any other non-entry file handled by the transformer is not limited.

Transforming TypeScript to JavaScript in the browser is archived by import the TypeScript compiler (`<script src="https://www.unpkg.com/typescript@3.6.2/lib/typescript.js">`) and call the `ts.transpileModule`.

There are different module targets in the `compilerOptions.module`: CommonJS, AMD, UMD, System, ES2015, and ESNext.

-   The transformed code will be evaluated by `eval`, so ES2015 or ESNext is not a choice
-   Besides ESNext, only the [SystemJS](https://github.com/systemjs/systemjs) format [supports the transformation of `import.meta`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-6.html#importmeta-support-in-systemjs).

By using `--target=system` with tsc,

```ts
import a from "./d";
const b = import(a);

export const y = 1;
```

will be transformed into

```js
System.register(["./d"], function (exports_1, context_1) {
    "use strict";
    var d_1, b, y;
    var __moduleName = context_1 && context_1.id;
    return {
        setters: [
            function (d_1_1) {
                d_1 = d_1_1;
            },
        ],
        execute: function () {
            b = context_1.import(d_1.default);
            exports_1("y", (y = 1));
        },
    };
});
```

By providing a `System` object, the problem of the import and export are resolved.

## Wrap the transformed code

1. Call `ts.transpileModule` transform the ESModule / TypeScript code into SystemJS format and JavaScript ([Source](https://github.com/Jack-Works/loader-with-esmodule-example/blob/master/src/typescript/typescript-shared-compiler.ts#L53))
2. Wrap it like this: ([Source](https://github.com/Jack-Works/loader-with-esmodule-example/blob/master/src/typescript/typescript-shared-compiler.ts#L21))

```ts
const System = new ECMAScriptModule(filePath);
const script = `(function (System) {
    'use strict'
    ${outputText}
})`;
eval(script)(System);
return System;
```

Here is the [first working implementation](https://github.com/Jack-Works/loader-with-esmodule-example/commit/49859a4167f3a384e68123809b0d8bddf9550e08#diff-6e4512a8b6c1de28c533004f1cbff879) and [the final implementation](https://github.com/Jack-Works/loader-with-esmodule-example/blob/master/src/typescript/typescript-shared-compiler.ts).

## Implement a SystemJS compatible `System` object

The `System` object should look like this:

```ts
interface SystemJSLoader {
    register(
        staticImports: string[],
        SystemJSModule: (
            /** The objects that this module exports */
            exported: (bindingName: string, value: any) => void,
            meta: {
                /** import.meta */
                meta: object;
                /** Dynamic import */
                import(src: string): Promise<any>;
            }
        ) => {
            /** When the imports resolve, call this function to pass the item in */
            setters: Array<(val: any) => void>;
            /** Execute the module */
            execute(): any;
        }
    ): void;
}
```

In the SystemJS object implementation, it needs

-   Recursively load all static dependencies
-   Prepare the `import.meta` object
-   Prepare the dynamic `import()` function
-   Execute the module and save all the `exports`

### Load a module

1. Prepare an `import.meta` object with a `url` property. ([Source](https://github.com/Jack-Works/loader-with-esmodule-example/blob/master/src/typescript/typescript-shared-compiler.ts#L94))
2. Prepare a dynamic `import()` function, it will call the custom module resolver internally. ([Source](https://github.com/Jack-Works/loader-with-esmodule-example/blob/master/src/typescript/typescript-shared-compiler.ts#L95-L102))
3. Provide the `import.meta` and `import()` to the 2nd parameter of the `System.register`, it will return a `{ setters, execute }`
4. Check all dependencies of this module

    i. if the module is loaded by the static import, fetch the dependencies by Sync XHR ([Source](https://github.com/Jack-Works/loader-with-esmodule-example/blob/master/src/typescript/typescript-shared-compiler.ts#L124))

    ii. if the module is loaded by the dynamic import, fetch the dependencies by `fetch` ([Source](https://github.com/Jack-Works/loader-with-esmodule-example/blob/master/src/typescript/typescript-shared-compiler.ts#L138))

    iii. after the dependencies loaded, bind their `exports` to the dependee's `import`. (This demo doesn't handle this well.)

5. After all the dependencies resolved, execute the module

# TypeScript loader in Service Worker

The problem becomes easier for a service worker. Compile the file content from TypeScript to JavaScript (keep it ESModule), then the browser itself will be able to run the ESModule format file.

And rewrite the `import` path.

By visiting the AST tree, all static imports are transformed in the following way.

[Static import transforms](https://github.com/Jack-Works/loader-with-esmodule-example/blob/master/src/typescript/typescript-shared-compiler.ts#L195)

```ts
// before
import { sth } from "./x";

// after
import { sth } from "/typescript-loader.js?src=/basepath/x";
```

[Dynamic import transforms](https://github.com/Jack-Works/loader-with-esmodule-example/blob/master/src/typescript/typescript-shared-compiler.ts#L226)

By traveling through the AST tree, all function calls are checked, if the callee is `ts.SyntaxKind.ImportKeyword` that means this expression is `import(....)`. Transform them in the following way:

```ts
// before
const data = import(a_complex_expression);

// after
const data = import(
    ((x) =>
        x.startsWith(".") || x.startsWith("/") ? "/typescript-loader.js?src=" + new URL(x, "/baseURL").pathname : x)(
        a_complex_expression
    )
);
```

# Shared loader between runtime and the Service Worker loader

Define both of them in the same file!

There is an example of the shared CSS Loader.

```ts
import { parseSrc } from "../loader-utils/load.js";
export default typeof document === "object" ? runtimeCompile() : undefined;

function runtimeCompile() {
    const src = parseSrc(import.meta.url, location.origin);
    const css = new CSSStyleSheet();

    fetch(src)
        .then((x) => x.text())
        .then((x) => css.replace(x));
    return css;
}

export async function swCompile(res) {
    return `const css = new CSSStyleSheet()
css.replace(${JSON.stringify(await res.text())})
export default css`;
}
```

# Let Service Worker itself loaded by the TypeScript loader!

The first try is to use ESModule in Service Worker directly (See: [W3C: ServiceWorkerContainer > RegistrationOptions > WorkerType](https://w3c.github.io/ServiceWorker/#dom-registrationoptions-type))

But when `navigator.serviceWorker.register(src, { type: 'module' })` get called, Chrome throws:

> Uncaught (in promise) DOMException: type 'module' in RegistrationOptions is not implemented yet.See https://crbug.com/824647 for details.

It seems Chrome is not implemented ESModule for Workers yet, so a classic Service Worker is written to load the TypeScript compiler and compile the real Service Worker.

[typescript-serviceworker-loader.js](https://github.com/Jack-Works/loader-with-esmodule-example/commit/9e7f9d66dc12a36e576986f918bd30ac8b1be686#diff-e3ede779f8b739eb1220185368ed030b)

# Browser Compatibility

<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import.meta#Browser_compatibility">import.meta</a>

<!-- <can-i-use feature="mdn-javascript_statements_import_meta"> -->
<!-- </can-i-use> -->
