# Static private fields and lexical declarations in classes

Champion: Daniel Ehrenberg

Stage 2

This proposal:
1. **Adds Public static fields as previously proposed**. Public static fields are own data properties of the constructor in which they are defined, which is evaluated after the class's TDZ has ended.
1. **Adds lexical declarations in class bodies**. Lexical declarations are prefixed by a contextual keyword to differentiate them from other class elements.
1. **Does not add private static fields and private static methods**. The use cases presented for private static are met by lexical declarations.

This proposal was created to track the "static" (i.e., of the constructor) aspects of the [class fields](http://github.com/tc39/proposal-class-fields) and [private methods](https://github.com/tc39/proposal-private-methods) proposals. In the November 2017 TC39 meeting, the static dimensions of these proposals were demoted to Stage 2, to be broken out into a separate proposal while the instance dimensions remain at Stage 3. Although lexical declarations are significantly different from the previous proposal of static private methods and fields, 

## Static public fields

Like static public methods, static public fields take a common idiom which was possible to write without class syntax and make it more ergonomic, have more declarative-feeling syntax (although the semantics are quite imperative), and allow free ordering with other class elements.

### Use case

```js
class CustomDate {
  // ...
  static epoch = new CustomDate(0)
}
```

JavaScript programmers can currently accomplish the same thing through this syntax:

```js
class CustomDate {
  // ...
}
CustomDate.epoch = new CustomDate(0);
```

Declaring static properties in the class body is hoped to be cleaner and doing a better job of meeting programmer expectations of what classes should be for. It's claimed that the latter workaround is a common idiom, and it would be a nice convenience for programmers if the property declaration could be lifted into the class body, matching how methods are placed there.

### Semantics

Define an own property on the constructor which is set to the value of the initializer expression. The initializer is evaluated in a scope where the binding of the class is available--unlike in computed property names, the class can be referred to from inside initializers without leading to a ReferenceError.

Alternate semantics are discussed in ["Initializing fields on subclasses"](#initializing-fields-on-subclasses). However, they are not proposed here, for reasons detailed in that section. These semantics are justified in more detail in ["Static field initialization"](#static-field-initialization).

### Why only initialize static fields once

Kevin Gibbons [raised a concern](https://github.com/tc39/proposal-class-fields/issues/43#issuecomment-340517955) that JavaScript programmers may not be used to having objects with mutated properties which are exposed on the prototype chain, the way that static fields are inherited and may be overwritten. Some reasons why this is not that bad:
- Many JS classes add a property to a constructor after a class definition. A subclass of Number, for example, would make this issue observable.
- Lots of current educational materials, e.g., by Kyle Simpson and Eric Elliott, explain directly how prototypical inheritance of data properties in JS works to newer programmers.
- This proposal is more conservative and going with the grain of JS by not adding a new time when code runs for subclassing, preserving identities as you'd expect, etc.

### Semantics in an edge case with Set and inheritance

Example of how these semantics work out with subclassing (this is not a recommended use of constructors as stateful objects, but it shows the semantic edge cases):

```js
static Counter {
  static count = 0;
  static inc() { this.count++; }
}
class SubCounter extends Counter { }

Counter.hasOwnProperty("count");  // true
SubCounter.hasOwnProperty("count");  // false

Counter.count; // 0, own property
SubCounter.count; // 0, inherited

Counter.inc();  // undefined
Counter.count;  // 1, own property
SubCounter.count;  // 1, inherited

// ++ will read up the prototype chain and write an own property
SubCounter.inc();

Counter.hasOwnProperty("count");  // true
SubCounter.hasOwnProperty("count");  // true

Counter.count;  // 1, own property
SubCounter.count;  // 2, own property

Counter.inc(); Counter.inc();
Counter.count;  // 3, own property
SubCounter.count;  // 2, own property
```

### Why not to reinitialize public fields on subclasses

Kevin Gibbons has proposed that class fields have their initialisers re-run on subclasses. This would address the static private subclassing issue by adding those to subclasses as well, leading to no TypeError on use.

With this alternate, the initial counter example would have the following semantics:

```js
// NOTE: COUTNERFACTUAL SEMANTICS BELOW
static Counter {
  static count = 0;
  static inc() { this.count++; }
}
class SubCounter extends Counter { }

Counter.hasOwnProperty("count");  // true
SubCounter.hasOwnProperty("count");  // true

Counter.count; // 0, own property
SubCounter.count; // 0, own property

Counter.inc();  // undefined
Counter.count;  // 1, own property
SubCounter.count;  // 0, own property

// ++ is just dealing iwth own properties the whole time
SubCounter.inc();

Counter.hasOwnProperty("count");  // true
SubCounter.hasOwnProperty("count");  // true

Counter.count;  // 1, own property
SubCounter.count;  // 1, own property

Counter.inc(); Counter.inc();
Counter.count;  // 3, own property
SubCounter.count;  // 1, own property
```

However, these alternate semantics have certain disadvantages:
- Subclassing in JS has always been "declarative" so far, not actually executing anything from the superclass. It's really not clear this is the kind of hook we want to add to suddenly execute code here.
- The use cases that have been presented so far for expecting the reinitialization semantics seem to use subclassing as a sort of way to create a new stateful class (e.g., with its own cache or counter, or copy of some other object). These could be accomplished with a factory function which returns a class, without requiring that this is how static fields work in general for cases that are not asking for this behavior.

## Lexically scoped declarations in classes

This proposal adds lexically scoped function, let, const and class declarations in class bodies. To make the syntax intuitively unambiguous, these declarations are prefixed by a yet-to-be-determined keyword. In this explainer, `<placeholder>` is used  in place of a particular token; see [#9](https://github.com/tc39/proposal-static-class-features/issues/9) for discussion of which token should be selected.

### Use case

Based on the example at [#4](https://github.com/tc39/proposal-static-class-features/issues/4) by Domenic Denicola:

```js
export class JSDOM {
  #createdBy;
  
  #registerWithRegistry() {
    // ... elided ...
  }
 
  async static fromURL(url, options = {}) {
    url = normalizeFromURLOptions(url, options);
    
    const body = await getBodyFromURL(url);
    return finalizeFactoryCreated(body, options, "fromURL");
  }
  
  static async fromFile(filename, options = {}) {
    const body = await getBodyFromFilename(filename);
    return finalizeFactoryCreated(body, options, "fromFile");
  }
  
  <placeholder> const registry = new JSDOMRegistry();
  <placeholder> function finalizeFactoryCreated(body, options, factoryName) {
    normalizeOptions(options);
    let jsdom = new JSDOM(body, options):
    jsdom.#createdBy = factoryName;
    jsdom.#registerWithRegistry(registry);
    return jsdom;
  }
}
```

### Semantics

Class bodies already contain a lexical scope, in which the class name is bound to the value of the class. In this same lexical scope, this feature adds additional bindings.

These lexical bindings add another time when code observably executes. This execution is proposed to tie in with [Brian Terlson and Yehuda Katz's broader proposal for class evaluation order](https://github.com/tc39/tc39-notes/blob/master/es7/2016-05/classevalorder.pdf) as follows:
- Function declarations (including async functions, generators and async generators) are hoisted up to the top of the class scope, initialized before the `extends` clause is evaluated.
- `let`, `const` and `class` delarations involve some evaluation. For the same reason as for static field initializers, it is helpful if the class is no longer in TDZ when they are executed. Therefore, the statements are executed interspersed with static field initializers (and possibly [element finalizers](https://github.com/tc39/proposal-decorators/issues/42) from decorators), top-to-bottom. Until they evaluate, the variables defined are in TDZ.

## No static private fields, methods and accessors

Previously, private static fields and methods were proposed. The original proposal for private static fields and methods was that the constructor object where they were defined would have these as private fields, and no other objects would have it. However, Justin Ridgewell, [pointed out](https://github.com/tc39/proposal-class-fields/issues/43) that this would be unexpectedly through TypeErrors in the case of subclassing.

Lexical declarations enable the known use cases of static private fields and methods. Due to the combination of the hazards of use and the lack of motivation, this proposal posits that static private features are unlikely to be a justified addition in the future.

## Public instance methods remain as previously proposed

Lexical declarations can sometimes be used for situations that would otherwise be used for private methods. However, the champion do not plan to withdraw the proposal even if lexically scoped function declarations in classes are accepted by TC39 for the following reasons:
- Private methods provide an easy path to refactoring from public to private--no need to rephrase methods or their callsites, just add a # before the definition and usages.
- Private methods give access to the right `this` value naturally, without using `Function.prototype.call`, which is core to making the refactoring easy.
- Private methods support super property access, which is a syntax error in function declarations.
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

Static public fields are implemented in Babel and V8 behind a flag, and had test262 tests written (though currently have been removed due to this proposal being at Stage 2).

Lexical declarations in classes do not currently have any implementations or tests.

Alternatives to this proposal are considered in [ALTERNATIVES.md].
