---
title: Glue Angular and React with Web Components
date: 2019-05-01
updated: 2019-11-23 22:41:30
tags:
    - angular
    - react
    - web component
lang: en
toc: true
cover: https://github.com/Jack-Works/hybird-angular-react-custom-element/blob/master/dom.png?raw=true
categories: frontend
---

[Demo: hybird-angular-react-custom-element](https://github.com/Jack-Works/hybird-angular-react-custom-element)

There's a lot of discussion about all of those front-end libraries/frameworks. I noticed that there is a common argument saying "React is just a view library" and "Angular is a full-function framework".

> Angular is considered a framework because it **offers strong opinions as to how your application should be structured**. It also has much more functionality **“out-of-the-box”**. You don’t need to decide which routing libraries to use or other such considerations – you can just start coding.

-- [React vs. Angular: The Complete Comparison](https://programmingwithmosh.com/react/react-vs-angular/)

They're referenceable. It raises an interesting question regarding whether we use React just as a view library to handle queries regarding UI, and use Angular to structure all others?

<!-- more -->

<i-exp status="success"></i-exp>

![DOM tree with Angular + React](https://github.com/Jack-Works/hybird-angular-react-custom-element/blob/master/dom.png?raw=true)

# The zeroth step

Create a new empty project with Angular CLI, then install React.

Let's start with an example of a counter.

# The first step: A static UI

The very first step is to transform React Component into Custom Element then render the custom element in the template of Angular. This step is easy to do.

https://github.com/Jack-Works/hybird-angular-react-custom-element/commit/388509da56c749cd2d9b4b540db41a61c23cde11

```ts
export function ReactToCustomElement<T>(ReactComponent: React.ComponentType<T> & CustomElementOptions) {
    if (ReactComponent.displayName === undefined || ReactComponent.displayName.indexOf("-") === -1)
        throw new TypeError('The "displayName" property must have a "-" in the middle.');
    class CustomElement extends HTMLElement {
        render(props: any) {
            ReactDOM.render(<ReactComponent {...(props as T)} />, this);
        }
        connectedCallback() {
            this.render({});
        }
    }
    customElements.define(ReactComponent.displayName, CustomElement, ReactComponent.customElementOptions);
    return CustomElement;
}
```

After registered the React component to the custom element registry, use it in the angular template.

Result:

```html
<!-- Angular host -->
<app-root _nghost-wyd-c0="" ng-version="8.2.11">
    <!-- Angular Component -->
    <app-counter _ngcontent-wyd-c0="">
        <!-- Custom Element -->
        <app-counter-ui>
            <!-- React Component mounted under the app-counter-ui -->
            <span>Current count: </span>
        </app-counter-ui>
    </app-counter>
</app-root>
```

# Props from Angular to React

Next step, let's try to feed props from the Angular template to React Component

```html
<!-- Angular template -->
<react-ui-counter [count]="count"></react-ui-counter>
```

🔽

```jsx
// JSX
<react-ui-counter count={count}></react-ui-counter>
```

By writing the template above, Angular will set the props as a property on the custom element. Now, it's time to research how to collect the properties and transform to React props?

## attributeChangedCallback?

[attributeChangedCallback](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements#Using_the_lifecycle_callbacks) is one of the life cycle callbacks of the custom element.

Unfortunately, this method needs the props to be watched hard coded (by `static get observedAttributes`). It can be used with `ReactComponent.propTypes`.

## Dirty check?

Check all props every animation frame.

It works in a ugly way.

## Proxy

By replacing the prototype of the custom element to a Proxy, any attribute sets on the element can be caught by the proxy.

https://github.com/Jack-Works/hybird-angular-react-custom-element/commit/0ee3cfd6d44d79e0f7dd28d5f9a938c81e726da5#diff-930d0a8e38f7669b96cdf1bdf816dd57

```ts
class CustomElement extends HTMLElement {
    private props: any = {};
    constructor() {
        super();
        Object.setPrototypeOf(
            this,
            new Proxy(HTMLElement.prototype, {
                set: (target, key, value, receiver) => {
                    this.props[key] = value;
                    ReactDOM.render(<ReactComponent {...this.props} />, this);
                    return Reflect.set(target, key, value, receiver);
                },
            })
        );
    }
}
```

Problem: React will also set some properties on the root element (like `__reactInternalInstance$sgc4z2mg06o`) in this manner a loop will occur.

Solution: Mount React in a child element.

```html
<!-- Angular host -->
<app-root _nghost-wyd-c0="" ng-version="8.2.11">
    <!-- Angular Component -->
    <app-counter _ngcontent-wyd-c0="">
        <!-- Custom Element -->
        <app-counter-ui>
            <!-- An invalid HTML element used to host the React Components -->
            <host>
                <span>Current count: 0.7299600627007301</span>
            </host>
        </app-counter-ui>
    </app-counter>
</app-root>
```

# Event listener from Angular to React

Next step, translate the React `onEvent` pattern to HTML's `addEventListener` pattern.

## On Angular 7.2

Angular will call `element.addEventListener`. Overwrite the `addEventListener` method when creating custom element is enough.

## On Angular 8.2

Angular will call `EventTarget.prototype.addEventListener` instead of our overwritten version.

Possible solutions:

### Provide Proxy as props to React.

Found it is impossible. `React.createElement` will make a copy of the props with all properties on the props object.

### Hack addEventListener

Another problem: Angular is not going to access the `EventTarget.prototype.addEventListener` in the runtime. Angular is keeping the reference since the app is initialized.

Solution: Run the hack code before Angular (write it in `src/index.html`)

https://github.com/Jack-Works/hybird-angular-react-custom-element/commit/75e05c0c00132677c4f167f3530aebcf1990321e

```ts
EventTarget.prototype.addEventListener = new Proxy(EventTarget.prototype.addEventListener, {
    apply(target, thisArg, args) {
        const tagName = thisArg?.tagName;
        if (typeof tagName === "string") {
            if (tagName.match("-")) {
                const constructor = customElements.get(tagName.toLowerCase());
                const addEventListener = constructor?.prototype?.addEventListener;
                if (addEventListener) {
                    return Reflect.apply(addEventListener, thisArg, args);
                }
            }
        }
        return Reflect.apply(target, thisArg, args);
    },
});
```

Now, Angular will use the overwritten version of `addEventListener`, it is possible to transform listeners from Angular to React props.

https://github.com/Jack-Works/hybird-angular-react-custom-element/commit/3cbb91f35f123bdcae7108d9f0662549c1a2f76f

```ts
addEventListener(event: string, handler, options) {
    this[props][event] = (...args) => {
        handler(
            new Proxy(new CustomEvent(event, { detail: args.length > 1 ? args : args[0] }), {
                get: (target, key, receiver) => {
                    if (key === 'target') return this
                    return target[key]
                }
            })
        )
    }
}
```

Until now, Angular and React can communicate with `property={p}` <==> `[property]="p"` and `onChange={f}` <==> `(onChange)="onChange($event)"`.

# Type level enforcement

A major advantage of Angular is it is using TypeScript with a custom compiler to integrate the template with TypeScript.

To use custom elements in Angular, a [CUSTOM_ELEMENTS_SCHEMA](https://angular.io/api/core/CUSTOM_ELEMENTS_SCHEMA) must be set in the module. By using this schema, type checking in the template is disabled (if I recall it correctly). It is also not possible to "define" the type of custom element in Angular.

## Enforce type of Angular component from the type of the React component

```diff
- export class CounterComponent implements OnInit {
+ export class CounterComponent implements OnInit, ReactComponentProps<typeof Counter> {
```

Done. If the props of `Counter` changed, TypeScript will complain `Class 'CounterComponent' incorrectly implements interface 'ReactComponentProps<typeof Counter>'.`.

https://github.com/Jack-Works/hybird-angular-react-custom-element/commit/2292197b62c323d80702715ff9a9c971933539ab

## Generate Angular template from React props

Since it is impossible to define the type of a custom element in the Angular template, generate the template for the Angular component is a way to resolve this problem.

In the last step, the Angular component is enforced to implement React props, there is some information available on the class in runtime.

For methods, it will appear on the prototype of the component class. For class fields, TypeScript will transform it like the following:

```ts
class I {
    x = 1;
}
// to
class I {
    constructor() {
        this.x = 1;
    }
}
```

A hacky way is to call `.toString()` and scan all `this.*` to collect all properties.

https://github.com/Jack-Works/hybird-angular-react-custom-element/commit/da9527113b4b5eb92fceb2abde597e85c590c3ad#diff-930d0a8e38f7669b96cdf1bdf816dd57

Now, this solution is fully typed.

# Upgrade to Angular 9

This solution needs dynamically generate the Angular in runtime so it is impossible to be statically analyzed.

Disable the AOT and Ivy compiler.

# A Todo MVC

To prove this solution is working, I write a Todo MVC demo.

Problems:

## Changes on a mutable object can not invoke a re-render

Solution: Invoke a re-render after an event in the `addEventListener`

https://github.com/Jack-Works/hybird-angular-react-custom-element/commit/5f95df8eba3cfeea61196b24da46900936e947f3#diff-930d0a8e38f7669b96cdf1bdf816dd57

## Props modifications in async operations can not invoke a re-render

Angular itself is using Zone.js to schedule a check after the async task is complete. It is also a good approach to schedule a check.

Solution: hack `Zone.current._zoneDelegate.__proto__.invokeTask` to get angular zone then add a callback on all async task has done.

https://github.com/Jack-Works/hybird-angular-react-custom-element/commit/be836cb482c360af81bae1770224568656b36ffb

```ts
const onAngularZoneCallbackMap = new Set<(...args: any[]) => void>();
{
    let angularZone: Zone = undefined!;
    // @ts-ignore
    Zone.current._zoneDelegate.__proto__.invokeTask = new Proxy(Zone.current._zoneDelegate.__proto__.invokeTask, {
        apply(target, thisArg, args) {
            if (args[0].name === "angular") {
                angularZone = args[0];
                // @ts-ignore
                const original = angularZone._zoneDelegate._hasTaskZS.onHasTask;
                // @ts-ignore Patch Angular Zone here.
                angularZone._zoneDelegate._hasTaskZS.onHasTask = function (...args) {
                    onAngularZoneCallbackMap.forEach((x) => x());
                    return Reflect.apply(original, this, args);
                } as ZoneSpec["onHasTask"];
                // @ts-ignore restore the object
                Zone.current._zoneDelegate.__proto__.invokeTask = target;
            }
            return Reflect.apply(target, thisArg, args);
        },
    });
}
```

```ts
// in the custom element's constructor
onAngularZoneCallbackMap.add(() => {
    // @ts-ignore Don't use requestAnimationFrame, it's patched by Zone.
    globalThis[Zone.__symbol__("requestAnimationFrame")](() => render(ReactComponent, this[props], this[host]));
});
```
