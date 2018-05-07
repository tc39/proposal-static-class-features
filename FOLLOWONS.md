# Potential follow-on proposals

This document describes some possible follow-on proposals which can be added to classes. All of these proposals are consistent and compatible with static public fields, static private fields and methods, [class fields](https://github.com/tc39/proposal-class-fields), [private instance methods and accessors](https://github.com/tc39/proposal-private-methods), [decorators](https://github.com/tc39/proposal-decorators/) and each other.

## Context

Historically, the development of static class features began based on the combination of static public fields, static private fields, and static private methods, which were retracted to Stage 2 and the November 2017 TC39 meeting. Due to issues with static private, we've considered variations of static private and lexically scoped declarations in classes to meet similar use cases to what static private fields and methods might support. The current proposal is to decouple static public fields from these additional class features.

All of these ideas have been discussed intermittently in TC39 for multiple years, most even since before ES2015 was finalized. As other class features become more complete, and as we've been having more discussion on the static class features repository, these proposals have been becoming more concrete as well, getting down to technical tradeoffs on edge cases.

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
