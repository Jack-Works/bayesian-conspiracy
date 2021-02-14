---
title: ES Module for Web Extensions
date: 2019-10-02 12:12:22
updated: 2020-10-06 09:51:59
tags:
    - esmodule
    - web extension
lang: en
toc: true
categories: frontend
---

TLDR:

If you only need to support Chrome, it is pretty easy if you're using manifest v2 🎇.

For background page, popup, ..etc:

```html
<script type="module" src="./background.js"></script>
```

For content scripts:

```js
import(chrome.runtime.getURL("..."));
```

It's much harder to support Firefox. Please read the full article to get details.

<!-- more -->

<i-exp status="fail">
Please read the Conclusion section below.
</i-exp>

## About Firefox

To use ESModule in WebExtension with Firefox🦊, please wait for Firefox to fix [the bug][fx-bug].

> Update on 05/19/2020: I'm tired of waiting for Mozilla to fix this, and I decided to compile the code into SystemJS. SystemJS is fully compatible with ES Modules (after transform) that supports live binding, `import.meta` and dynamic import. I write a custom SystemJS runtime for Web Extension: [@magic-works/webextension-systemjs](https://www.npmjs.com/package/@magic-works/webextension-systemjs)

> Update on 10/6/2020: It works but the solution is very ugly. You also need to take aware of the value of `import.meta.url`. If the ESM build is in the `es` folder and SystemJS build is in the `system` folder, you need to hook [System.constructor.prototype.createContext](https://github.com/systemjs/systemjs/blob/master/docs/hooks.md#createcontexturl---object) to rewrite it to `es` therefore you can load resource by `fetch('./cat.png', import.meta.url)` when the resource is in the `es` folder.

## About node_modules

To use npm packages, notice that `snowpack` is not strong enough as the webpack to handle so many cases, it may fail to build your dependencies but please have a try!

> Update on 05/19/2020: I'm going to pack dependencies by webpack and distribute them in UMD format. I wrote [a custom typescript transformer](https://www.npmjs.com/package/@magic-works/ttypescript-browser-like-import-transformer) for converting `import`s into a UMD access. [Here is a template project](https://github.com/Jack-Works/ttsc-browser-import-template) to use it with webpack.

# Intuition

In Mar 2019, [@pika/web (renamed to snowpack) released](https://www.pika.dev/blog/pika-web-a-future-without-webpack).
I read the article few months later, and learned the concept of unbundled development.

I felt interested and inspired by that idea. That's so cool! You can run ES Modules directly in the browser and enjoy the benefits of it.

That time I'm working on [a browser extension called Maskbook](https://github.com/DimensionDev/Maskbook), and I found it's possible to [try the unbundled development in our project](https://github.com/DimensionDev/Maskbook/issues/221). Browser extensions are pre-downloaded as a ZIP file to the computer therefore it's no need to worry if the unbundled project will send too many HTTP requests and slow down the app. All files are loaded by local IO.

# Problem to solve

During the development of Maskbook, we encountered many problems with webpack chunk splitting.

> Update on 05/19/2020: Please check out [neutrino-webextension](https://www.npmjs.com/package/neutrino-webextension), it's a Web Extension preset for Webpack that support chunk splitting or dynamic import.
>
> Update on 10/06/2020: We're now using [webpack-target-webextension](https://www.npmjs.com/package/webpack-target-webextension). It resolved all the problems. [neutrino-webextension](https://www.npmjs.com/package/neutrino-webextension) mentioned above is based on this package too.

There're 5 entries.

-   `Content script`: run in the isolated high privileged sandbox in the target page
-   `Background page`: run as the "server" of the extension
-   `Options page`: a normal webpage page but in the `-extension://` protocol
-   `Popup`: another normal webpage
-   `Workers`: do some heavy works

It is a common practice to share dependencies (like React) in different entries to reduce the size.
Webpack has built-in support for this so we turn it on.
Then the nightmare comes.

-   webpack wants to load chunks by injecting the `<script>` tag, which will let the chunk content goes into another environment instead of the isolated environment of content scripts. (Resolved by a hack on `HTMLScriptElement.prototype.src`)
-   The Webpack-way of loading `import('...')` breaks our code.
-   ... and so many bugs caused by the chunk splitting

Chunk splitting is designed for normal webpages, not for WebExtension (which have it's own sandbox, protocol, CSP, etc...), and since it cause so many bugs in Maskbook thus decided to totally close the chunk splitting completely.
But we still need a way to share dependencies.

# But how to resolve it

There are some nature ideas to resolve this problem.

-   Manually splitting the dependencies (this idea is too boring and I'm not going to talk about them. (but I'm still considering it before this experiment has succeeded))
-   Other module loaders like AMD (😕Nah, like living in the 19th century)
-   ESModule.

# (✔️) ESModule YES

First, checkout the browser compatibility <can-i-use feature="es6-module"><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import#Browser_compatibility">of static import</a></can-i-use> <can-i-use feature="es6-module-dynamic-import">and <a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import#browser_compatibility">dynamic import</a> at mdn.io.</can-i-use>

Nice. But that's for the normal webpage. Let's try it in WebExtension.

# (✔️) Background page

Firstly I tried the following code.

```ts
import "./shared.js";
```

And failed with `Uncaught SyntaxError: Cannot use import statement outside a module`.

So the file need to be declared as ESModule. Renaming the file to `.mjs`, not working.

Looking for the document of `manifest.json`, but nowhere mentioned ESModule.
There is a field in the `background` section: [`manifest.json/background.page`](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/manifest.json/background). We can declare a HTML file for the background.

`background.html`:

```html
<script type="module" src="/background.js"></script>
```

And that's working!

Browser supporting:

|                      | Chrome | Firefox | Firefox for Android |
| -------------------- | ------ | ------- | ------------------- |
| `import { } from ''` | ✔️\*   | ✔️\*    | ❓                  |
| `import('')`         | ✔️\*   | ✔️\*    | ❓                  |

\*: Need to use a HTML file

# (❌ only works on Chrome!) Content script

Same as the background page, tried to run `import './shared.js'` directly.

And failed with `Uncaught SyntaxError: Cannot use import statement outside a module` again.

Unfortunately, solutions for the background page doesn't work for the content script because content script runs in other webpages and don't have their own HTML.

Then I tried:

```ts
import("/content.js");
```

Failed with a network error. Chrome tries to load the script from `https://example.com/content.js` instead of `chrome-extension://extension-id/content.js`. Hmm, interesting 🤔, so I changed it to

```ts
import(chrome.runtime.getURL("/content.js"));
```

You need to set [web_accessible_resources](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/manifest.json/web_accessible_resources) in the manifest so the JS file is accessible in normal webpage. Open that in your own cautions.

And it works! 🎇

**But, it doesn't work on Firefox.** Firefox throws `No ScriptLoader found for the current context`. What?

I searched the error message in the source code of Firefox.

It shows when Firefox tries to execute `import(...)`, it needs a `ScriptLoader`(for resource loading). The `ScriptLoader` comes from the `Document` and the `Document` comes from a `Window`.

By the document of WebExtension of [content script](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Content_scripts#Content_script_environment), it indicates content script may not have its own `Document`.

By [globalThis in WebExtension content script doesn't implements Window](https://bugzilla.mozilla.org/show_bug.cgi?id=1577400) and [this !== window within content_scripts](https://bugzilla.mozilla.org/show_bug.cgi?id=1208775), content script even may not have its own `Window`!

I'm not familiar with C++ so I'm not able to debug it to confirm my hypothesis or fix it.

Now I'm blocked by [dynamic module import doesn't work in web extension content scripts][fx-bug].

Browser supporting:

|                      | Chrome | Firefox | Firefox for Android |
| -------------------- | ------ | ------- | ------------------- |
| `import { } from ''` | ❌     | ❌      | ❓                  |
| `import('')`         | ✔️\*   | ❌      | ❓                  |

\*: Need to wrap with `chrome.runtime.getURL()`

# (✔️❌ partly works) Use npm modules in the browser

A GitHub organization `@pikapkg/` has already made a solution for this.

[A Future Without Webpack: snowpack installs npm packages that run natively in the browser. Do you still need a bundler?](https://www.pika.dev/blog/pika-web-a-future-without-webpack/)

Experiment in production: [DimensionDev/Maskbook:feature/experiment-pika](https://github.com/DimensionDev/Maskbook/tree/feature/experiment-pika) and there are 2(or 3) problems.

## Import path

Browsers only know how to load an absolute URL or relative URL and the `.js` cannot be omitted. But we already used to `import('lib')` or `import('./code')`. That's an easy problem to resolve. Snowpack provides a solution to do it but I'm using `tsc` to compile Maskbook thus I wrote a [TypeScript custom transformer](https://www.npmjs.com/package/@magic-works/ttypescript-browser-like-import-transformer) and load it by [ttypescript](https://npmjs.com/package/ttypescript) to transform the `import` path.

### Note on `import './file.json'`

<del>JSON import is rewritten to `data:application/javascript,export default {"json": "content"}`</del>

JSON import is rewritten to `const json = JSON.parse("JSON file content")`

> Update on 05/19/2020: Sorry for the mistake. The data url import is banned by the CSP in the extension environment. I switch to inline the JSON, see the example at [the document of @magic-works/ttypescript-browser-like-import-transformer](https://jack-works.github.io/ttypescript-browser-like-import-transformer/config.pluginconfigs.jsonimport.html).

### Note on "folder import"

When writing `import './sth'` in TypeScript, this import declaration may means

-   `import './sth.js'`
-   `import './sth.jsx'`
-   `import './sth.ts'`
-   `import './sth.tsx'`
-   `import './sth/index.js'`
-   `import './sth/index.jsx'`
-   `import './sth/index.ts'`
-   `import './sth/index.tsx'`

When writing an path transform plugin, don't forget to cover all the cases!

> Update on 05/19/2020: Good news, you don't need to reinvent the wheel. [@magic-works/ttypescript-browser-like-import-transformer](https://jack-works.github.io/ttypescript-browser-like-import-transformer/config.pluginconfigs.folderimport.html) also supports folder import.

## 💥 No Tree-shaking

> This whole section about tree-shaking is out-dated.
>
> Snowpack provides [tree-shaking](https://www.snowpack.dev/#bundle-for-production) now.
>
> I also implements tree shaking for `@magic-works/ttypescript-browser-like-import-transformer`! ([Link to documentation](https://jack-works.github.io/ttypescript-browser-like-import-transformer/config.rewriterulesumd.treeshake.html))

It seems impossible to use tree-shaking with `@pika/web`. It doesn't scan your code to drop all unused dependencies. It tries to transform all packages in `dependencies` to ESModule in your `package.json`.

Full packages of `lodash-es`, `@material-ui/core` and `@material-ui/icons` are generated with a horrifying size.

🤔 Doesn't know how to resolve it yet.

## ❌ Cannot omit optional dependencies

It's a common pattern to do in the npm packages.

```ts
try {
    // faster but need native binding
    require("./native");
} catch {
    // slower
    require("./js");
}
```

[`snowpack` currently cannot config to omit some of the packages](https://www.pika.dev/packages/snowpack/discuss/1113). And it will try to build everything even it is an optional dependency. This makes the compilation process slow even not available to work.

# Conclusion (in Nov 2020)

## Snowpack

I tried to pack all our dependencies by snowpack. It failed. Some of our dependencies let Rollup throws (e.g. Web3.js), some of our dependencies let snowpack out of response. I moved to another route: if Snowpack can pack this dependencies, I covert the import route to "/web_modules". If not, I rewrite the import statement into a global variable access.

I made a [TypeScript custom transformer](https://www.npmjs.com/package/@magic-works/ttypescript-browser-like-import-transformer) to do the AST transformation.

It will transform the code

```js
console.log(a);
import a from "a";
```

into this

```js
const a = _import(globalThis["a"], ["default"], "a", "globalThis.a", false).default;
console.log(a);
import { _import } from "https://cdn.jsdelivr.net/npm/@magic-works/ttypescript-browser-like-import-transformer@2.3.0/es/ttsclib.min.js";
```

Feel ironic huh? I want to import libraries as ES Modules, but finally my solution is using global variables.

What ever, the dependency problem is resolved. My final solution doesn't have snowpack in it.
I wrote [a custom Webpack loader](https://github.com/DimensionDev/Maskbook/blob/2501477d686e3fb7a3aeeadcd6ebc220da8df236/scripts/gulp/tree-shake-loader.ts) will drop all normal codes and add something like:

```js
import { something } from "some_library";
__using("some_library", { something });
```

to track what dependencies webpack should bundle, then `__using` will set those references into global variables.

## Browser support

Despite the problem I met in the tool chain, the browser also have problem of running ES module directly. [Firefox does not support using dynamic import in the content script][fx-bug] therefore it is impossible to load a module as ES module. This bug has three year history so I think Mozilla won't fix it in short time. For this, my solution is, I also translating code into [SystemJS](https://www.npmjs.com/package/systemjs) (SystemJS is a full compatible module format to ES Module). Now, simple `tsc -p` is not enough for my need. Finally I build a complex build system based on [gulp](https://www.npmjs.com/package/gulp), and it finally works on Firefox and Chrome.

But the solution is very ugly after applying so many workaround.

## Next step: HMR

After it is being able to work, the next step is support HMR. It is possible to HMR in ES Module but I'm not willing to investigate more time in inventing tool chain. I gave up and return to the webpack.

## Reference

Here is some useful resources.

### Articles

-   [Hot reloading native ES2015 modules](https://itnext.io/hot-reloading-native-es2015-modules-dc54cd8cca01)

### Issues

-   Chrome: [chrome.tabs.executeScript doesn't work when file name contains "~"](https://bugs.chromium.org/p/chromium/issues/detail?id=1108199) which `~` is used by Webpack chunk splitting by default.
-   Chrome: [Extensions fail to install if the temp directory is under an NTFS junction](https://bugs.chromium.org/p/chromium/issues/detail?id=13044)
-   Firefox: [Dynamic module import doesn't work in webextension content scripts][fx-bug]
-   Firefox: [this !== window within content_scripts](https://bugzilla.mozilla.org/show_bug.cgi?id=1208775)
-   Firefox: [WebExtension executeScript fails with symlink when loaded temporarily on linux](https://bugzilla.mozilla.org/show_bug.cgi?id=1420286)
-   Firefox: [Extension error of content script doesn't appear in DevTool console](https://bugzilla.mozilla.org/show_bug.cgi?id=1469304)
-   Agoric/realms-shim: [Any signal about supporting ESModules?](https://github.com/Agoric/realms-shim/issues/47)
-   Snowpack: [Support ignore field in commonjs support](https://www.pika.dev/packages/snowpack/discuss/1113)

### Packages and repos

-   pikapkg / [esm-hmr](https://npmjs.com/package/esm-hmr), ES Module Hot Module Reload specification.
-   Jack-Works / [ttypescript-browser-like-import-transformer](https://www.npmjs.com/package/@magic-works/ttypescript-browser-like-import-transformer), compile ES Module into UMD.
-   Jack-Works / [webextension-systemjs](https://npmjs.com/package/@magic-works/webextension-systemjs), SystemJS runtime for WebExtension content script.
-   Jack-Works / [commonjs-import.meta](https://npmjs.com/package/@magic-works/commonjs-import.meta), support `import.meta` in CommonJS via a transformer.
-   Jack-Works / [web-extension-esmodule-test](https://github.com/Jack-Works/web-extension-esmodule-test), a test suit for ES Module in WebExtension.
-   cevek / [ttypescript](https://npmjs.com/package/ttypescript), a wrapped `tsc` to use transformers easily.
-   crimx / [webpack-target-webextension](https://npmjs.com/package/webpack-target-webextension), a Webpack preset for WebExtension.
-   crimx / [neutrino-webextension](https://npmjs.com/package/neutrino-webextension), a Webpack based toolchain for WebExtension.

[fx-bug]: https://bugzilla.mozilla.org/show_bug.cgi?id=1536094
