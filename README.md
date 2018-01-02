# Static class features

Champion: Daniel Ehrenberg

Stage 2

This proposal tracks the "static" (i.e., of the constructor) aspects of the [class fields](http://github.com/tc39/proposal-class-fields) and [private methods](https://github.com/tc39/proposal-private-methods) proposals. In the November 2017 TC39 meeting, the static dimensions of these proposals were demoted to Stage 2, to be broken out into a separate proposal while the instance dimensions remain at Stage 3. This repository is created to explore the aspects which are still at Stage 2, to determine how to proceed to Stage 3 with some version of them.

The current proposal is to continue with the previously proposed semantics, where
1. **Public and private static fields are initialized only once in the class where they are defined, not additionally in subclasses**. [see below](https://github.com/tc39/proposal-static-class-features/blob/master/README.md#static-field-initialization)
1. **Private static fields and methods can be called only on the class; TypeError when used with anything else**. [see below](https://github.com/tc39/proposal-static-class-features/blob/master/README.md#static-private-access-on-subclasses)

## Static public fields

Like static public methods, static public fields take a common idiom which was possible to write without class syntax and make it more ergonomic, have more declarative-feeling syntax (although the semantics are quite imperative), and allow free ordering with other class elements.

### Semantics

Define an own property on the constructor which is set to the value of the initializer expression. The initializer is evaluated in a scope where the binding of the class is available--unlike in computed property names, the class can be referred to from inside initializers without leading to a ReferenceError.

Alternate semantics are discussed in ["Initializing fields on subclasses"](https://github.com/tc39/proposal-static-class-features#initializing-fields-on-subclasses). However, they are not proposed here, for reasons detailed in that section. These semantics are justified in more detail in ["Static field initialization"](https://github.com/tc39/proposal-static-class-features#static-field-initialization).

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

## Static private methods

### Semantics

The class has an own private method, similar to private instance methods. Conceptually, private fields and methods can be thought of as being based on a WeakMap mapping objects to values; here, the WeakMap has just one key, which is the constructor where the private method was declared. This method is not installed on subclasses, which means that calling a static private method with a subclass as the receiver will lead to a TypeError. For the reasons in ["Static private access on subclasses"](#static-private-access-on-subclasses), the champion considers this not to be a significant problem.

### Use case

Static private methods can be useful whenever there is shared behavior to extract into a function which uses private fields, but which doesn't work cleanly as an instance method. For example, multiple factory static methods may share part of their implementation, including parts which run both before and after construction of the instance. See [#4](https://github.com/tc39/proposal-static-class-features/issues/4) for more context about the following example.

```js
export const registry = new JSDOMRegistry();

export class JSDOM {
  #createdBy;
  
  #registerWithRegistry() {
    // ... elided ...
  }
 
  async static fromURL(url, options = {}) {
    normalizeFromURLOptions(options);
    normalizeOptions(options);
    
    const body = await getBodyFromURL(url);
    return JSDOM.#finalizeFactoryCreated(new JSDOM(body, options), "fromURL");
  }
  
  static fromFile(filename, options = {}) {
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

In [Issue #1](https://github.com/tc39/proposal-static-class-features/issues/1), there is further discussion about whether this feature is well-motivated. In particular, static private methods can typically be replaced by either lexically scoped function declarations outside the class declaration, or by private instance methods. However, the current proposal is to include them, due to the use cases in [#4](https://github.com/tc39/proposal-static-class-features/issues/4).

## Static private fields

### Semantics

The class has an own private field, similar to private instance fields. This field is not installed on subclasses, and the initializer is only ever evaluated once. As with static public fields, the initializer is evaluated in a scope where the binding of the class is available--unlike in computed property names, the class can be referred to from inside initializers without leading to a ReferenceError.

### Use case

```js
class ColorFinder {
  static #red = "#ff0000";
  static #blue = "#00ff00";
  static #green = "#0000ff";
  
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

In [Issue #1](https://github.com/tc39/proposal-static-class-features/issues/1), there is further discussion about whether this feature is well-motivated. In particular, static private fields can typically be subsumed by lexically scoped variables outside the class declaration. Static private fields are motivated mostly by a desire to allow free ordering of things inside classes and consistency with other class features.

## Why these semantics?

### Static field initialization

Kevin Gibbons [raised a concern](https://github.com/tc39/proposal-class-fields/issues/43#issuecomment-340517955) that JavaScript programmers may not be used to having objects with mutated properties which are exposed on the prototype chain, the way that static fields are inherited and may be overwritten. Some reasons why this is not that bad:
- Many JS classes add a property to a constructor after a class definition. A subclass of Number, for example, would make this issue observable.
- Lots of current educational materials, e.g., by Kyle Simpson and Eric Elliott, explain directly how prototypical inheritance of data properties in JS works to newer programmers.
- This proposal is more conservative and going with the grain of JS by not adding a new time when code runs for subclassing, preserving identities as you'd expect, etc.

### Static private access on subclasses

Justin Ridgewell [raised a concern](https://github.com/tc39/proposal-class-fields/issues/43) that static fields and methods will lead to a TypeError when `this` is used as the receiver from within a static method, and they are invoked from a subclass. This concern is hoped to be not too serious because:
- Programmers can avoid the issue by instead writing `ClassName.#method`. This phrasing should be easier to understand, anyway--no need to worry about what `this` refers to.
- It is not so bad to repeat the class name when accessing a private static method or field. When implementing a recursive function, the name of the function needs to be repeated; this case is similar.
- It is statically known whether a private name refers to a static or instance-related class element. Therefore, implementations should be able to make helpful error messages for instance issues that say "TypeError: The private field #foo is only present on instances of ClassName, but it was accessed on an object which was not an instance", or, "TypeError: The static private method #bar is only present on the class ClassName; but it was accessed on a subclass or other object".
- Linters could flag uses of `foo.#method` where `foo` is not the name of the class, and discourage its use that way.
- Type systems which want to be slightly more accepting could trigger an error on any class with a class which is subclassed and has public method which uses a private method or field without the class's name being the receiver. In the case of TypeScript, [this is already not polymorphic](https://github.com/Microsoft/TypeScript/issues/5863) so it would already flag instances of `this.#method` for a private static method call within a public static method.
- Beginners can just learn the rule (helped by linters, type systems and error messages) to refer to the class when calling static private methods. Advanced users can learn the simple mental model that corresponds to the specification: private things are represented by a WeakMap, and public things by ordinary properties. When it's by the instance, the WeakMap has a key for each instance; when it's static, the WeakMap has a single key, for the constructor.
- A guaranteed TypeError when the code runs on the subclass is a relatively easy to debug failure mode. This is how JS responds, at a high level, when you access an undefined variable as well, or read a property on undefined. A silent other interpretation would be the real footgun.
- TC39 has worked hard on other guaranteed exceptions, such as the "temporal dead zone"--a ReferenceError 

## "Back pocket" alternatives

These alternatives are not currently proposed by the champion, but are considered possibly feasible options.

### Restricting private access to `static.#foo`

Jordan Harband suggested that we make `static.` a syntax for referring to a property of the immediately enclosing class. If we add this feature, we could say that private static fields and methods may *only* be accessed that way. This has the disadvantage that, in the case of nested classes, there is no way to access the outer class's private static methods without copying that method into another local variable before entering into another nested class.

### Restricting static private access to `ClassName.#`

The syntax for accessing private static fields and methods would be restricted to using the class name textually as the receiver. However, this would be a somewhat new kind of way to use scopes for early errors. Unlike var/let conflict early errors, this is much more speculative--the class name might actually be shadowed locally, which the early error would not catch, leading to a TypeError.

## Alternate proposals not selected

Several alternatives have been discussed within TC39. This repository is not pursuing these ideas further. Some may be feasible, but are not selected by the champion for reasons described below.

### Initializing fields on subclasses

Kevin Gibbons has proposed that class fields have their initialisers re-run on subclasses. This would address the static private subclassing issue by adding those to subclasses as well, leading to no TypeError on use. However, this proposal has certain disadvantages:
- Subclassing in JS has always been "declarative" so far, not actually executing anything from the superclass. It's really not clear this is the kind of hook we want to add to suddenly execute code here.
- The use cases that have been presented so far for expecting the reinitialization semantics seem to use subclassing as a sort of way to create a new stateful class (e.g., with its own cache or counter, or copy of some other object). These could be accomplished with a factory function which returns a class, without requiring that this is how static fields work in general for cases that are not asking for this behavior.

### Prototype chain walk for private fields and methods

Ron Buckton [has proposed](https://github.com/tc39/proposal-private-methods/issues/18) that private field and method access could reach up the prototype chain. There are two ways that this could be specified, both of which have significant issues:
- **Go up the normal prototype chain with \[\[GetPrototypeOf]]**. This would be observable and interceptible by proxies, violating a design goal of private fields that they not be manipulated that way.
- **Use a separate, parallel, immutable prototype chain**. This alternative would add extra complexity, and break the way that other use of classes consistently works together with runtime mutations in the prototype chain.

### Lexically scoped variables and function declarations in classes

Allen Wirfs-Brock has proposed lexically scoped functions and variables within class bodies. The main issue here is that the proposed syntax may be unintuitive, [as explained here](https://github.com/tc39/proposal-static-class-features/issues/4#issuecomment-354515761).

### Banning static private fields and methods

This alternative was previously proposed in this repository. The alternative is dispreferred due to [the use cases in Issue #4](https://github.com/tc39/proposal-static-class-features/issues/4).

A related alternative is moving ahead with static public fields while continuing development on static private fields. This repository is devoted to a full exploration of the problem space and is currently at Stage 2 which has been requested by committee members to proceed; it is not current proposed to advance some parts and not others.

### Install private static methods on subclasses; omit private static fields

This alternative would meet many of the presented use cases, but it is pretty ad-hoc.
