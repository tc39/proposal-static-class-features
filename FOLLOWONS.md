# Potential follow-on proposals

This document describes some possible follow-on proposals which can be added to classes. All of these proposals are consistent and compatible with static public fields, [class fields](https://github.com/tc39/proposal-class-fields), [private instance methods and accessors](https://github.com/tc39/proposal-private-methods), [decorators](https://github.com/tc39/proposal-decorators/) and each other.

### Context

Historically, the development of static class features began based on the combination of static public fields, static private fields, and static private methods, which were retracted to Stage 2 and the November 2017 TC39 meeting. Due to issues with static private, we've considered variations of static private and lexically scoped declarations in classes to meet similar use cases to what static private fields and methods might support. The current proposal is to decouple static public fields from these additional class features.

All of these ideas have been discussed intermittently in TC39 for multiple years, most even since before ES2015 was finalized. As other class features become more complete, and as we've been having more discussion on the static class features repository, these proposals have been becoming more concrete as well, getting down to technical tradeoffs on edge cases.

## Static private

Previously, private static fields and methods were proposed. The original proposal for private static fields and methods was that the constructor object where they were defined would have these as private fields, and no other objects would have it. However, Justin Ridgewell, [pointed out](https://github.com/tc39/proposal-class-fields/issues/43) that this would be unexpectedly throw TypeErrors in the case of subclassing.

Lexical declarations enable the known use cases of static private fields and methods. Between the cost in complexity and counter-intuitive semantics and the lack of motivation, this proposal posits that static private features are unlikely to be a justified addition in the future.

See [ALTERNATIVES.md](https://github.com/tc39/proposal-static-class-features/blob/master/ALTERNATIVES.md#static-private) for an in-depth look at various options that were considered to support static private fields and methods.

This proposal does not include static private fields, methods or accessors. An investigation into various semantic alternatives found none that were satisfactory. Instead, lexically scoped declarations in classes are proposed to fill most of the use cases of static private.

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

### Static private fields with a TypeError on access from subclasses

#### Semantics

Unlike static private methods, static private fields are private fields just of the constructor. If a static private field is read with the receiver being a subclass, a TypeError will be triggered.

As with static public fields, the initializer is evaluated in a scope where the binding of the class is available--unlike in computed property names, the class can be referred to from inside initializers without leading to a ReferenceError. As described in [Why only initialize static fields once](#why-only-initialize-static-fields-once), the initializer is evaluated only once.

#### Use case

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

In [Issue #1](https://github.com/tc39/proposal-static-class-features/issues/1), there is further discussion about whether this feature is well-motivated. In particular, static private fields can typically be subsumed by lexically scoped variables outside the class declaration. Static private fields are motivated mostly by a desire to allow free ordering of things inside classes and consistency with other class features. It's also possible that some initialzer expressions may want to refer to private names declared inside the class, e.g., in [this gist](https://gist.github.com/littledan/19c09a09d2afe7558cdfd6fdae18f956).

The behavior of private static methods have, in copying to subclass constructors, does not apply to private static fields. This is due to the complexity and lack of a good option for semantics which are analogous to static public fields, as explained below.

#### TypeError case

Justin Ridgewell [expressed concern](https://github.com/tc39/proposal-class-fields/issues/43) about the TypeError that results from static private field access from subclasses. Here's an example of that TypeError, which occurs when code ported from the above static public fields example is switched to private fields:

```js
static Counter {
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

A TypeError is used here because no acceptable alternative would have the semantics which are analogous to static public fields. Some alternatives are discussed below.

#### Why this TypeError might not be so bad

Daniel Ehrenberg has argued that the TypeError would not be so bad for the following reasons:
- Programmers can avoid the issue by instead writing `ClassName.#field`. This phrasing should be easier to understand, anyway--no need to worry about what `this` refers to.
- It is not so bad to repeat the class name when accessing a private static field. When implementing a recursive function, the name of the function needs to be repeated; this case is similar.
- It is statically known whether a private name refers to a static or instance-related class field. Therefore, implementations should be able to make helpful error messages for instance issues that say "TypeError: The private field #foo is only present on instances of ClassName, but it was accessed on an object which was not an instance", or, "TypeError: The static private method #bar is only present on the class ClassName; but it was accessed on a subclass or other object", etc.
- Linters could flag uses of `foo.#field` where `foo` is not the name of the class and `#field` is a static private field, and discourage its use that way.
- Type systems which want to be slightly more accepting could trigger an error on any class with a class which is subclassed and has public method which uses a private method or field without the class's name being the receiver. In the case of TypeScript, [this is already not polymorphic](https://github.com/Microsoft/TypeScript/issues/5863) so it would already flag instances of `this.#field` for a private static method call within a public static method.
- Beginners can just learn the rule (helped by linters, type systems and error messages) to refer to the class when calling static private methods. Advanced users can learn the simple mental model that corresponds to the specification: private things are represented by a WeakMap, and public things by ordinary properties. When it's by the instance, the WeakMap has a key for each instance; when it's static, the WeakMap has a single key, for the constructor.
- A guaranteed TypeError when the code runs on the subclass is a relatively easy to debug failure mode. This is how JS responds, at a high level, when you access an undefined variable as well, or read a property on undefined. A silent other interpretation would be the real footgun.
- TC39 has worked hard on other guaranteed exceptions, such as the "temporal dead zone"--a ReferenceError from reading `let`- or `const`-declared variables whose declaration has not yet been reached. Here, a runtime error was considered very useful for programmers and a way of preventing a "footgun". In this case, we're getting such a guaranteed error.

#### Downsides

Despite those reasons above, it still may be confusing that a TypeError is given. Additionally, it could be confusing if private static fields inherit and private static methods do not.

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

## Lexical declarations in class bodies

Fundamentally, static private allows for some kind of functions and variables to be included in class bodies. The idea of lexical declarations in class bodies is to provide this capability based on JavaScript's lexical scoping mechanism, rather than an extension of private fields.

The most basic version of this follow-on proposal adds lexically scoped `function` declarations (including async functions and generators) in class bodies. To make the syntax intuitively unambiguous, these declarations are prefixed by a yet-to-be-determined keyword. In this explainer, `local` is used in place of a particular token; see [#9](https://github.com/tc39/proposal-static-class-features/issues/9) for discussion of which token should be selected.

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

### Additional lexically declared forms

In addition to `function` declarations, class bodies could support `let`, `class` and `const` declarations.

#### Complexities

`let`, `class` and `const` declarations add another time when code observably executes. This execution would need to tie in with [Brian Terlson and Yehuda Katz's broader proposal for class evaluation order](https://github.com/tc39/tc39-notes/blob/master/es7/2016-05/classevalorder.pdf). For the same reason as for static field initializers, it is helpful if the class has an initialized binding when they are executed. Therefore, it would seem logical to execute the statements interspersed with static field initializers (and possibly [element finalizers](https://github.com/tc39/proposal-decorators/issues/42) from decorators), top-to-bottom. Until they evaluate, accessing the variable causes a ReferenceError ("temporal dead zone"). It's unclear, however, how this top-to-bottom initialization order [would be integrated](https://github.com/tc39/proposal-decorators/issues/44) with the decorators proposal.

The scope of these lexical declarations would also be observable in potentially confusing ways. If we use typical lexical scoping rules, it would be the same as the scope of the `extends` clause and computed property names: It is a lexical scope inheriting from outside the class, which includes using `this`, `super`, `await`, `yield` and `arguments`. These features are hidden by function declarations, avoiding the issue. Further discussion is in [#13](https://github.com/tc39/proposal-static-class-features/issues/13).

Finally, the most important use cases that we were able to identify in the development of this proposal were instances of procuedural decomposition. This is typically represented by functions. Although it's possible to write code which would take advantage of the other declarations, it's unclear whether these needs are worth the complexity of the above two issues.

## Static blocks

A static block is a block of code which runs once, inside the class, when the class is declared. It's useful to run code inside the class, as this is the scope that can access private fields and methods (similarly to how static private and lexical declarations in classes can access those fields and methods). The concept is borrowed from languages like Java, which include static block syntax.

The earlier example could be rewritten using static blocks as such:

```js
const registry = new JSDOMRegistry();
let finalizeFactoryCreated;
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
  
  static {
    finalizeFactoryCreated = (body, options, factoryName) => {
      normalizeOptions(options);
      let jsdom = new JSDOM(body, options):
      jsdom.#createdBy = factoryName;
      jsdom.#registerWithRegistry(registry);
      return jsdom;
    }
  }
}
```

Static blocks can also be used to expose private fields externally in limited ways, for example:

```js
let getter;

class X {
  #x;
  static {
    getter = arg => arg.#x;
  }
}
```

There are some details to finalize, such as what exact scope static blocks are run in (e.g., what is `arguments`, `this`, etc), when they are executed relative to other class elements, whether multiple blocks are permitted, etc. Ron Buckton has made [a solid proposal](https://github.com/tc39/proposal-static-class-features/issues/23#issuecomment-360599243) for these details.

See further motivation, examples and discussion in [Issue #23](https://github.com/tc39/proposal-static-class-features/issues/23).

## Private names declared outside of classes

Another option for making use of lexical scoping to be able to get at private fields and methods is to enable private names to be declared more flexibly, not just at the class level. In this strawperson below, the syntax `outer #name` is used in a class declaration to refer, in a class declaration, to a private name declared in an outer scope, rather than redefine one in an inner scope.

```js
const registry = new JSDOMRegistry();

private #createdBy;
private #registerWithRegistry;

function finalizeFactoryCreated (body, options, factoryName) {
  normalizeOptions(options);
  let jsdom = new JSDOM(body, options):
  jsdom.#createdBy = factoryName;
  jsdom.#registerWithRegistry(registry);
  return jsdom;
}

export class JSDOM {
  outer #createdBy;
  
  outer #registerWithRegistry(registry) {
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
}
```

There's been some discussion about whether the syntax for private field and method declarations in classes should always start with `private`, so that `private #x` can consistently be how all private names are declared. However, some counterarguments to this proposal are:
- When a private field declaration is decorated to make a public accessor, it would be strange to have `private` syntactically present, e.g., in the case of `@reader private #x;` when the purpose of the declaration is to make something public.
- This alternative makes the by far most common case more verbose in a way which doesn't have much intuitive meaning. `#` already indicates privacy.

For these reasons, the private fields and methods proposals have stuck with their current syntax.

For some more extended examples, see [this gist](https://gist.github.com/littledan/d3030534cf96075d47228955828f932e).
