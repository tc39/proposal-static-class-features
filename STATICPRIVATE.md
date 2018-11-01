# Static private

Static private fields and methods were originally proposed with the semantics described in this repository, that private fields and methods are defined on the constructor and no other objects.

We've discussed at length [a concern raised by Justin Ridgewell](https://github.com/tc39/proposal-class-fields/issues/43) that it could be unexpected that these semantics throw TypeErrors in an inheritance-related case. This document discusses various alternatives, none of which have been found to be acceptable, and ultimately concludes that these errors are not problematic, describing beginner and expert mental models.

## Proposed semantics

### Semantics

Private static fields and methods are defined on the constructor. If a static private field or method is accessed with the receiver being a subclass, a TypeError will be triggered.

As with static public fields, the initializer is evaluated in a scope where the binding of the class is available--unlike in computed property names, the class can be referred to from inside initializers without leading to a ReferenceError. As described in [Why only initialize static fields once](#why-only-initialize-static-fields-once), the initializer is evaluated only once.

### TypeError case

Justin Ridgewell [expressed concern](https://github.com/tc39/proposal-class-fields/issues/43) about the TypeError that results from static private field access from subclasses. Here's an example of that TypeError, which occurs when code ported from the above static public fields example is switched to private fields:

```js
class Counter {
  static #count = 0;
  static inc() { this.#count++; }
  static get count() { return this.#count; }
}
class SubCounter extends Counter { }

Counter.inc();  // undefined
Counter.count;  // 1

SubCounter.inc();  // TypeError

Counter.count;  // 1
SubCounter.count;  // TypeError
```

An analogous example could be constructed with private static methods.

A TypeError is used here because no acceptable alternative would have the semantics which are analogous to static public fields. Some alternatives are discussed below.

## Alternatives to avoid the TypeError

### Static private methods and accessors which are inherited

#### Semantics

Like [instance private methods](https://github.com/tc39/proposal-private-methods/blob/master/DETAILS.md), static private methods are conceptually non-writable private fields on objects whose value is the method. There is no prototype chain access in private fields or methods; instead, the methods are installed in each object that would be able to access the method through prototype chain lookups.

In the case of instance private methods, this is instances of the class and subclasses; for static private methods, this is the constructor and subclass constructors. Invoking a private method on an object where it is missing causes a TypeError.

In specification terms, when a subclass is created which extends another class, the non-writable and accessor private 'fields' (namely, the private methods and accessors) of the super class are copied to the subclass. This makes it so that the private methods and accessors can be called on the subclass.

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
    url = normalizeFromURLOptions(url, options);
    
    const body = await getBodyFromURL(url);
    return JSDOM.#finalizeFactoryCreated(body, options, "fromURL");
  }
  
  static async fromFile(filename, options = {}) {
    const body = await getBodyFromFilename(filename);
    return JSDOM.#finalizeFactoryCreated(body, options, "fromFile");
  }
  
  static #finalizeFactoryCreated(body, options, factoryName) {
    normalizeOptions(options);
    let jsdom = new JSDOM(body, options):
    jsdom.#createdBy = factoryName;
    jsdom.#registerWithRegistry(registry);
    return jsdom;
  }
}
```

In [Issue #1](https://github.com/tc39/proposal-static-class-features/issues/1), there is further discussion about whether this feature is well-motivated. In particular, static private methods can typically be replaced by either lexically scoped function declarations outside the class declaration, or by private instance methods. However, the current proposal is to include them, due to the use cases in [#4](https://github.com/tc39/proposal-static-class-features/issues/4).

#### Subclassing use case

As described above, static private methods may be invoked with subclass instances as the receiver. Note that this does *not* mean that subclass bodies may call private methods from superclasses--the methods are private, not protected, and it would be a SyntaxError to call the method from a place where it's not syntactically present.

In the below example, a public static method `from` is refactored, using a private static method. This private static method has an extension point which can be overridden by subclasses, which is that it calls the `of` method. All of this is in service of factory functions for creating new instances. If the private method `#from` were only callable with `MyArray` as the receiver, a `TypeError` would result.

```js
class MyArray {
  static #from(obj) {
    this.of.apply(...obj);
  }
  static from(obj) {
    // This function gets large and complex, with parts shared with
    // other static methods, so an inner portion #from is factored out
    return this.#from(obj);
  }
}

class SubMyArray extends MyArray {
  static of(...args) {
    let obj = new SubMyArray();
    let i = 0;
    for (let arg of args) {
      obj[i] = arg;
      i++;
    }
  }
}

let subarr = MySubArray.from([1, 2, 3]);
```

#### Downsides

Static private methods in this form have fairly complicated semantics--it's the act of extending a class from another class that copies them onto the subclass. It does not seem like it will be feasible to use similar semantics for object literals. The use cases for static private methods are very similar to the use cases for lexically scoped function declarations.

### Allow static private fields to be read from subclasses

If we want to get static private fields to be as close as possible to static public fields when it comes to inheritance semantics: The shadowing property of writes to properties on the prototype chain is very weird and complicated, and wouldn't make sense to replicate; on the other hand, reads up the prototype chain are simple and intelligible. We could allow reads, but not writes, to take place from subclasses. The reads would reflect writes that happen later to the superclass's static private field.

This alternative is currently not selected because it would be pretty complicated, and lead to a complicated mental model. It would still not make public and private static fields completely parallel, as writes from subclasses are not allowed.

Here's an example of code which would be enabled by this alternative (based on code by Justin Ridgewell):

```js
class Base {
  static #field = 'hello';

  static get() {
    return this.#field;
  }

  static set(value) {
    return this.#field = value;
  }
}

class Sub extends Base {}

Base.get(); // => 'hello'
Base.set('xyz');

Sub.get();  // => 'xyz' in this alternative, TypeError in the main proposal
Sub.set('abc');  // TypeError
```

### Imitate a prototype chain for static private field semantics

Ron Buckton [has](https://github.com/tc39/proposal-class-fields/issues/43#issuecomment-348045445) [proposed](https://github.com/tc39/proposal-private-methods/issues/18) that private field and method access could reach up the prototype chain. There are two ways that this could be specified, both of which have significant issues:
- **Go up the normal prototype chain with \[\[GetPrototypeOf]]**. This would be observable and interceptible by proxies, violating a design goal of private fields that they not be manipulated that way.
- **Use a separate, parallel, immutable prototype chain**. This alternative would add extra complexity, and break the way that other use of classes consistently works together with runtime mutations in the prototype chain.

A positive aspect of Ron's proposal is that it preserves identical behavior with respect to subclassing as public static fields.

In [this PR](https://github.com/tc39/proposal-static-class-features/pull/7), logic is drafted to give inherited static private fields semantics which are similar to inherited public static fields: There is, in effect, a "private prototype chain" which is maintained; reads to private static fields go up that prototype chain, while writes are reflected only in that one constructor.

This alternative is rejected due to its high complexity and implementation requirements, which do not seem justified by use cases or programmer demand, especially in the presence of the lexically scoped function alternative.

### Restricting static private field access to `static.#foo`

Jordan Harband [suggested](https://github.com/tc39/proposal-class-fields/issues/43#issuecomment-328874041) that we make `static.` a syntax for referring to a property of the immediately enclosing class. If we add this feature, we could say that private static fields and methods may *only* be accessed that way. This has the disadvantage that, in the case of nested classes, there is no way to access the outer class's private static methods. However, as a mitigation, programmers may copy that method into another local variable before entering into another nested class, making it still available.

With this alternative, the above code sample could be written as follows:

```js
class Base {
  static #field = 'hello';

  static get() {
    return static.#field;
  }

  static set(value) {
    return static.#field = value;
  }
}

class Sub extends Base {}

Base.get(); // => 'hello'
Base.set('xyz');

Sub.get();  // => 'xyz'
Sub.set('abc');

Base.get();  // 'abc'
```

Here, `static` refers to the base class, not the subclass, so issues about access on subclass instances do not occur.

### Restricting static field private access to `ClassName.#foo`

The syntax for accessing private static fields and methods would be restricted to using the class name textually as the receiver. This would make it a SyntaxError to use `this.#privateMethod()` within a static method, for example. However, this would be a somewhat new kind of way to use scopes for early errors. Unlike var/let conflict early errors, this is much more speculative--the class name might actually be shadowed locally, which the early error would not catch, leading to a TypeError. In the championed semantics, such checks would be expected to be part of a linter or type system instead.

With this alternative, the above code sample could be written as follows:

```js
class Base {
  static #field = 'hello';

  static get() {
    return Base.#field;
  }

  static set(value) {
    return Base.#field = value;
  }
}

class Sub extends Base {}

Base.get(); // => 'hello'
Base.set('xyz');

Sub.get();  // => 'xyz'
Sub.set('abc');

Base.get();  // 'abc'
```

Here, explicit references the base class, not the subclass, so issues about access on subclass instances do not occur. A reference like `this.#field` would be an early error, helping to avoid errors by programmers.

## Why the TypeError is not so bad

The alternatives for static private run into issues, described above. This section concludes that the original TypeError "hazard" is not so bad, and that we should stick with the original semantics:
- Programmers can avoid the issue by instead writing `ClassName.#field`. This phrasing should be easier to understand, anyway--no need to worry about what `this` refers to.
- It is not so bad to repeat the class name when accessing a private static field. When implementing a recursive function, the name of the function needs to be repeated; this case is similar.
- It is statically known whether a private name refers to a static or instance-related class field. Therefore, implementations should be able to make helpful error messages for instance issues that say "TypeError: The private field #foo is only present on instances of ClassName, but it was accessed on an object which was not an instance", or, "TypeError: The static private method #bar is only present on the class ClassName; but it was accessed on a subclass or other object", etc.
- Linters could flag uses of `foo.#field` where `foo` is not the name of the class and `#field` is a static private field, and discourage its use that way.
- Type systems which want to be slightly more accepting could trigger an error on any class with a class which is subclassed and has public method which uses a private method or field without the class's name being the receiver. In the case of TypeScript, [this is already not polymorphic](https://github.com/Microsoft/TypeScript/issues/5863) so it would already flag instances of `this.#field` for a private static method call within a public static method.
- Beginners can just learn the rule (helped by linters, type systems and error messages) to refer to the class when calling static private methods. Advanced users can learn the simple mental model that corresponds to the specification: private things are represented by a WeakMap, and public things by ordinary properties. When it's by the instance, the WeakMap has a key for each instance; when it's static, the WeakMap has a single key, for the constructor.
- A guaranteed TypeError when the code runs on the subclass is a relatively easy to debug failure mode. This is how JS responds, at a high level, when you access an undefined variable as well, or read a property on undefined. A silent other interpretation would be the real footgun.
- TC39 has worked hard on other guaranteed exceptions, such as the "temporal dead zone"--a ReferenceError from reading `let`- or `const`-declared variables whose declaration has not yet been reached. Here, a runtime error was considered very useful for programmers and a way of preventing a "footgun". In this case, we're getting such a guaranteed error.

## Programmer mental models

These proposed semantics are designed to be learnable by beginners as well as understandable at a deep level by experts, without any mismatch. The semantics here aren't just "something that works well in the spec", but rather a design which remains intuitive and consistent through multiple lenses.

See the [blog post](https://rfrn.org/~shu/2018/05/02/the-semantics-of-all-js-class-elements.html) by Shu-Yu Guo for a detailed look at class features in general, including the complete semantics and how they fit into a mental model scheme.

### Beginner, simplified semantics

To make something inaccessible from outside the class, put a `#` at the beginning of the name. When a field or method has `static`, that means it is on the class, not on instances. To access static class elements, use the syntax `ClassName.element` or `ClassName.#element`, depending on whether the name contains a `#`. Note that the latter case is available only inside the class declaration. If you make a mistake in accessing private class elements, JS will throw `TypeError` when the code is run to make it easier to catch the programming error. To catch errors earlier in development, try using a linter or type system.

### Expert, complete semantics

Public and private class elements are analogous to each other, with the difference being that public class elements are implemented as properties on the object, and private class elements are like a WeakMap, or internal slot. The rest of this section is written in terms of a WeakMap, but could be phrased equally in terms of internal slots.

Public class elements are defined with \[\[DefineOwnProperty]] when they are added to the relevant object, and private class elements have a corresponding WeakMap, mapping objects to their values, which an object is added to when the element is defined for the object. For instance elements, this relevant object is either the prototype or the instance; for static elements, this relevant object is the constructor.

Static fields are syntax sugar for defining a property or private field of a constructor after the rest of the class definition runs. As such, they are able to access or instantiate the class in their initializer. Static fields are installed on just the particular constructor, whether as an own property or a key in a singleton WeakMap.

The above semantics are orthogonal and consistent among private vs public, static vs instance, and field vs method, making it easy for advanced programmers to do appropriate refactorings between features.
