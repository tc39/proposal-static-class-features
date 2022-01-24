# Static class features

Champions: Shu-yu Guo, Daniel Ehrenberg

Stage 4

This proposal adds three features to JavaScript classes, building on the previous [class fields](http://github.com/tc39/proposal-class-fields) and [private methods](https://github.com/tc39/proposal-private-methods) proposals:
- Static public fields
- Static private methods
- Static private fields

## Current status

This proposal is at Stage 4.

This proposal was created to track the "static" (i.e., of the constructor) aspects of the [class fields](http://github.com/tc39/proposal-class-fields) and [private methods](https://github.com/tc39/proposal-private-methods) proposals, namely static public fields, static private fields, and static private methods. In the November 2017 TC39 meeting, the static dimensions of these proposals were demoted to Stage 2, to be broken out into a separate proposal while the instance dimensions remain at Stage 3. In the May 2018 TC39 meeting, this proposal was promoted to Stage 3. In the April 2021 TC39 meeting, this proposal was promoted to Stage 4.

Draft specification text is published at [https://tc39.github.io/proposal-static-class-features/].

This proposal is stable and with a variaty of implementations shipped.

|Implementation|Status|
|---|---|
|Babel|[Babel 7.1](https://babeljs.io/blog/2018/09/17/7.1.0#private-static-fields-stage-3) shipped Private Static Fields<br>[Babel 7.4](https://babeljs.io/blog/2019/03/19/7.4.0#static-private-methods-9446-https-githubcom-babel-babel-pull-9446) shipped Private Static Methods<br>[Babel 7.6](https://babeljs.io/blog/2019/09/05/7.6.0#private-static-accessors-getters-and-setters-10217-https-githubcom-babel-babel-pull-10217) shipped Private Static Accessors|
|Moddable XS|[XS](https://blog.moddable.com/blog/secureprivate/) shipped full implementation|
|QuickJS|[QuickJS](https://www.freelists.org/post/quickjs-devel/New-release,82) shipped full implementation|
|Chrome| Full implementation [shipping](https://www.chromestatus.com/feature/6001727933251584) |
|Firefox| [Firefox v75](https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Releases/75) shipped [`static` public fields](https://bugzilla.mozilla.org/show_bug.cgi?id=1535804) |
|Safari| Public and private static fields [shipping](https://webkit.org/blog/11364/release-notes-for-safari-technology-preview-117/) in Technology Preview 117 |

## Static public fields

### Motivation

Like static public methods, static public fields take a common idiom which was possible to write without class syntax and make it more ergonomic, have more declarative-feeling syntax (although the semantics are quite imperative), and allow free ordering with other class elements.

Declaring static properties in the class body is hoped to be cleaner and doing a better job of meeting programmer expectations of what classes should be for. The latter workaround is a somewhat common idiom, and it would be a nice convenience for programmers if the property declaration could be lifted into the class body, matching how methods are placed there.

### Example

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

### Semantics

Define an own property on the constructor which is set to the value of the initializer expression. The initializer is evaluated in a scope where the binding of the class is available--unlike in computed property names, the class can be referred to from inside initializers without leading to a ReferenceError. The `this` value in the initializer is the constructor.

See [STATICPUBLIC.md](https://github.com/tc39/proposal-static-class-features/blob/master/STATICPUBLIC.md) for an explanation of some of the edge cases and alternatives considered.

## Static private methods

### Semantics

The class has an own private method, similar to private instance methods. Conceptually, private fields and methods can be thought of as being based on a WeakMap mapping objects to values; here, the WeakMap has just one key, which is the constructor where the private method was declared. This method is not installed on subclasses, which means that calling a static private method with a subclass as the receiver will lead to a TypeError. For the reasons in [the STATICPRIVATE.md document](https://github.com/tc39/proposal-static-class-features/blob/master/STATICPRIVATE.md), the champion considers this not to be a significant problem.

### Use case

Static private methods can be useful whenever there is shared behavior to extract into a function which uses private fields, but which doesn't work cleanly as an instance method. For example, multiple factory static methods may share part of their implementation, including parts which run both before and after construction of the instance. See [#4](https://github.com/tc39/proposal-static-class-features/issues/4) for more context about the following example.

```js
export const registry = new JSDOMRegistry();

export class JSDOM {
  #createdBy;
  
  #registerWithRegistry() {
    // ... elided ...
  }
 
  static async fromURL(url, options = {}) {
    normalizeFromURLOptions(options);
    normalizeOptions(options);
    
    const body = await getBodyFromURL(url);
    return JSDOM.#finalizeFactoryCreated(new JSDOM(body, options), "fromURL");
  }
  
  static async fromFile(filename, options = {}) {
    normalizeOptions(options);
    
    const body = await getBodyFromFilename(filename);
    return JSDOM.#finalizeFactoryCreated(new JSDOM(body, options), "fromFile");
  }
  
  static #finalizeFactoryCreated(jsdom, factoryName) {
    jsdom.#createdBy = factoryName;
    jsdom.#registerWithRegistry(registry);
    return jsdom;
  }
}
```

## Static private fields

### Semantics

The class has an own private field, similar to private instance fields. This field is not installed on subclasses, and the initializer is only ever evaluated once. As with static public fields, the initializer is evaluated in a scope where the binding of the class is available--unlike in computed property names, the class can be referred to from inside initializers without leading to a ReferenceError.

### Use case

```js
class ColorFinder {
  static #red = "#ff0000";
  static #green = "#00ff00";
  static #blue = "#0000ff";
  
  static colorName(name) {
    switch (name) {
      case "red": return ColorFinder.#red;
      case "blue": return ColorFinder.#blue;
      case "green": return ColorFinder.#green;
      default: throw new RangeError("unknown color");
    }
  }
  
  // Somehow use colorName
}
```

Note that the intended use is to access private static fields on the class *by name*, rather than via `this`: since the fields are not installed on subclasses, writing `this.#red` in the previous example would throw a TypeError when calling colorName on a subclass of ColorFinder without overriding the implementation.

## Follow-on proposals

This proposal is designed to be compatible with several possible follow-on proposals, which are documented at [FOLLOWONS.md](https://github.com/tc39/proposal-static-class-features/blob/master/FOLLOWONS.md).
