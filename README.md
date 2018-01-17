# Static private fields and lexical declarations in classes

Champion: Daniel Ehrenberg

Stage 2

This proposal:
1. **Adds public static fields** as previously proposed. Public static fields are own data properties of the constructor.
1. **Adds lexical declarations in class bodies**. Lexical declarations are prefixed by a keyword to differentiate them from methods and fields.
1. **Does not add private static fields and private static methods**. Lexical declarations are intended to fill static private use cases.

This proposal was created to track the "static" (i.e., of the constructor) aspects of the [class fields](http://github.com/tc39/proposal-class-fields) and [private methods](https://github.com/tc39/proposal-private-methods) proposals. In the November 2017 TC39 meeting, the static dimensions of these proposals were demoted to Stage 2, to be broken out into a separate proposal while the instance dimensions remain at Stage 3. Although lexical declarations are significantly different from the previous proposal of static private methods and fields, this feature set is working to accomplish the same use cases, so it is being pursued as part of the same TC39 proposal.

## Static public fields

Like static public methods, static public fields take a common idiom which was possible to write without class syntax and make it more ergonomic, have more declarative-feeling syntax (although the semantics are quite imperative), and allow free ordering with other class elements.

### Use case

```js
class CustomDate {
  // ...
  static epoch = new CustomDate(0);
}
```

JavaScript programmers can currently accomplish the same thing through this syntax:

```js
class CustomDate {
  // ...
}
CustomDate.epoch = new CustomDate(0);
```

Declaring static properties in the class body is hoped to be cleaner and doing a better job of meeting programmer expectations of what classes should be for. The latter workaround is a somewhat common idiom, and it would be a nice convenience for programmers if the property declaration could be lifted into the class body, matching how methods are placed there.

### Semantics

Define an own property on the constructor which is set to the value of the initializer expression. The initializer is evaluated in a scope where the binding of the class is available--unlike in computed property names, the class can be referred to from inside initializers without leading to a ReferenceError. The `this` value in the initializer is the constructor.

See [ALTERNATIVES.md](https://github.com/tc39/proposal-static-class-features/blob/master/ALTERNATIVES.md#static-fields) for an explanation of some of the edge cases and alternatives considered.

## Function scoped declarations in classes

This proposal adds lexically scoped `function` declarations (including async functions and generators) in class bodies. To make the syntax intuitively unambiguous, these declarations are prefixed by a yet-to-be-determined keyword. In this explainer, `local` is used in place of a particular token; see [#9](https://github.com/tc39/proposal-static-class-features/issues/9) for discussion of which token should be selected.

### Use case

Based on the example at [#4](https://github.com/tc39/proposal-static-class-features/issues/4) by Domenic Denicola:

```js
const registry = new JSDOMRegistry();
export class JSDOM {
  #createdBy;
  
  #registerWithRegistry(registry) {
    // ... elided ...
  }
 
  static async fromURL(url, options = {}) {
    url = normalizeFromURLOptions(url, options);
    
    const body = await getBodyFromURL(url);
    return finalizeFactoryCreated(body, options, "fromURL");
  }
  
  static async fromFile(filename, options = {}) {
    const body = await getBodyFromFilename(filename);
    return finalizeFactoryCreated(body, options, "fromFile");
  }
  
  local function finalizeFactoryCreated(body, options, factoryName) {
    normalizeOptions(options);
    let jsdom = new JSDOM(body, options):
    jsdom.#createdBy = factoryName;
    jsdom.#registerWithRegistry(registry);
    return jsdom;
  }
}
```

### Semantics

Class bodies already contain a lexical scope, in which the class name is bound to the value of the class. In this same lexical scope, this feature adds additional bindings. Function declarations (including async functions, generators and async generators) are hoisted up to the top of the class scope, initialized before the `extends` clause is evaluated. They are defined within this scope, and able to access private instance fields and be accessed by private methods.

### Omitting other lexically declared forms

`let`, `class` and `const` declarations add another time when code observably executes. This execution would need to tie in with [Brian Terlson and Yehuda Katz's broader proposal for class evaluation order](https://github.com/tc39/tc39-notes/blob/master/es7/2016-05/classevalorder.pdf). For the same reason as for static field initializers, it is helpful if the class has an initialized binding when they are executed. Therefore, it would seem logical to execute the statements interspersed with static field initializers (and possibly [element finalizers](https://github.com/tc39/proposal-decorators/issues/42) from decorators), top-to-bottom. Until they evaluate, accessing the variable causes a ReferenceError ("temporal dead zone"). It's unclear, however, how this top-to-bottom initialization order [would be integrated](https://github.com/tc39/proposal-decorators/issues/44) with the decorators proposal.

The scope of these lexical declarations would also be observable in potentially confusing ways. If we use typical lexical scoping rules, it would be the same as the scope of the `extends` clause and computed property names: It is a lexical scope inheriting from outside the class, which includes using `this`, `super`, `await`, `yield` and `arguments`. These features are hidden by function declarations, avoiding the issue. Further discussion is in [#13](https://github.com/tc39/proposal-static-class-features/issues/13).

Finally, the most important use cases that we were able to identify in the development of this proposal were instances of procuedural decomposition. This is typically represented by functions. Although it's possible to write code which would take advantage of the other declarations, it's unclear whether these needs are worth the complexity of the above two issues.

## No static private fields, methods and accessors

Previously, private static fields and methods were proposed. The original proposal for private static fields and methods was that the constructor object where they were defined would have these as private fields, and no other objects would have it. However, Justin Ridgewell, [pointed out](https://github.com/tc39/proposal-class-fields/issues/43) that this would be unexpectedly throw TypeErrors in the case of subclassing.

Lexical declarations enable the known use cases of static private fields and methods. Between the cost in complexity and counter-intuitive semantics and the lack of motivation, this proposal posits that static private features are unlikely to be a justified addition in the future.

See [ALTERNATIVES.md](https://github.com/tc39/proposal-static-class-features/blob/master/ALTERNATIVES.md#static-private) for an in-depth look at various options that were considered to support static private fields and methods.

## Public instance methods remain as previously proposed

Lexical declarations can sometimes be used for situations that would otherwise be used for the Stage 3 [private methods and accessors proposal](https://github.com/tc39/proposal-private-methods). However, the champions do not plan to withdraw the proposal even if lexically scoped function declarations in classes are accepted by TC39 for the following reasons:
- Private methods provide an easy path to refactoring from public to private--no need to rephrase methods or their callsites, just add a # before the definition and usages.
- Private methods give access to the right `this` value naturally, without using `Function.prototype.call`, which is core to making the refactoring easy. Many programmers feel comfortable programming JavaScript in an object-oriented style, which private methods support.
- Private methods support super property access, which is a syntax error in function declarations. It's unclear how lexically scoped declarations could support super property access, as the "home object" would be rather ambiguous--the prototype or the constructor?
- Private methods are terse and convenient to use, with semantics which is naturally analogous to othre class elements. They should be sufficient for the majority of shared but not exposed behavior in classes.
- Private methods do not have the TypeError hazards which private static methods have--they are installed in the constructor, so they are present according to the prototype chain of the class hierarchy at the time the instance was constructed.

### Use case

The [private methods explainer](https://github.com/tc39/proposal-private-methods) walks through an example of incrementally refactoring code to make more use of private methods and accessors. The following is the final stage of the code, with the full refactoring in place.

```js
class Counter extends HTMLElement {
  #xValue = 0;

  get #x() { return #xValue; }
  set #x(value) {
    this.#xValue = value; 
    window.requestAnimationFrame(this.#render.bind(this));
  }

  #clicked() {
    this.#x++;
  }

  constructor() {
    super();
    this.onclick = this.#clicked.bind(this);
  }

  connectedCallback() { this.#render(); }

  #render() {
    this.textContent = this.#x.toString();
  }
}
window.customElements.define('num-counter', Counter);
```

## Proposal status

This proposal is at Stage 2. In the November 2017 TC39 meeting, it was [split off](https://github.com/tc39/tc39-notes/blob/master/es8/2017-11/nov-30.md#10iva-continued-inheriting-private-static-class-elements-discussion-and-resolution) from the Stage 3 [class fields](https://github.com/tc39/proposal-class-fields) and [private methods](https://github.com/tc39/proposal-private-methods) proposals--this proposal handles the static aspects, while the other proposals handle instance aspects. This proposal will be presented to TC39 in the January 2018 meeting as an update.

Draft specification text is published at [https://tc39.github.io/proposal-static-class-features/].

Static public fields are implemented in Babel and V8 behind a flag, and had test262 tests written (though currently have been removed due to this proposal being at Stage 2).

Lexical declarations in classes do not currently have any implementations or tests.

Private methods and accessors are [a separate Stage 3 proposal](https://github.com/tc39/proposal-private-methods).

Alternatives to this proposal are considered in [ALTERNATIVES.md](https://github.com/tc39/proposal-static-class-features/blob/master/ALTERNATIVES.md).
