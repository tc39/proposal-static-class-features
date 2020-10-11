# Static fields: Why these semantics?

Static public fields are proposed as ordinary data properties of constructors. This document explores the edge cases, implications and some rejected alternative proposals for their semantics.

## Current proposal: Static fields are initialized only once

Kevin Gibbons [raised a concern](https://github.com/tc39/proposal-class-fields/issues/43#issuecomment-340517955) that JavaScript programmers may not be used to having objects with mutated properties which are exposed on the prototype chain, the way that static fields are inherited and may be overwritten. Some reasons why this is not that bad:
- Many JS classes add a property to a constructor after a class definition. A subclass of Number, for example, would make this issue observable.
- Lots of current educational materials, e.g., by Kyle Simpson and Eric Elliott, explain directly how prototypical inheritance of data properties in JS works to newer programmers.
- This proposal is more conservative and going with the grain of JS by not adding a new time when code runs for subclassing, preserving identities as you'd expect, etc.

## Semantics in an edge case with Set and inheritance

Example of how these semantics work out with subclassing (this is not a recommended use of constructors as stateful objects, but it shows the semantic edge cases):

```js
class Counter {
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

## Why not to reinitialize public fields on subclasses

Kevin Gibbons has proposed that class fields have their initialisers re-run on subclasses. This would address the static private subclassing issue by adding those to subclasses as well, leading to no TypeError on use.

With this alternate, the initial counter example would have the following semantics:

```js
// NOTE: COUNTERFACTUAL SEMANTICS BELOW
class Counter {
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

// ++ is just dealing with own properties the whole time
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

## Switching all fields to being based on accessors

The idea here is to avoid the subclassing overwriting hazard by changing the semantics of all field declarations: Rather than being own properties that are shadowed by a Set on a subclass, they are accessors (getter/setter pairs). In the case of instance fields, the accessor would read or write on the receiver. In the case of static fields, presumably, the read and write would happen on the superclass where they are defined, ignoring the receiver (otherwise, the TypeError issue from private static fields is then ported to public static fields as well, as the subclass constructor would not have its own value!). Private static fields would also follow this accessor pattern. In all cases, the getter would throw a TypeError when the receiver is an object which does not "have" the private field (e.g., its initializer has not run yet), an effectively new TDZ, which could reduce programmer errors.

Some downsides of this proposal:
- **Public fields would no longer be own properties**. This may be rather confusing for programmers, who may expect features like object spread to include public instance fields.
- **Does not help implementations and might hurt startup time (maybe)**. This idea was initially proposed as part of a concept for having 'static shape', which could provide more predictability for implementers and programmers. At least on the implementation side, however, there would either have to be checks on each access to see if the field was initialized, or the initialization state would have to show up in its "hidden class". Either way, there's no efficiency gain if the fields are "already there, just in TDZ". In some implementations, startup time could be even worse than with own properties, until the system learns to optimize out the accessors.
- **Loses the object model--we'd have to start again**. Data properties have an object model permitting non-writable, non-enumerable and non-configurable properties, including its use by `Object.freeze`. If we want to provide these sorts of capabilities to public fields, they would have to be built again separately.

For these reasons, the public fields proposal has been based on own properties.

## Accessor-like semantics only for static fields

Justin Ridgewell [proposed](https://github.com/tc39/proposal-static-class-features/issues/24) accessor-like semantics for static fields, with own property semantics for instance fields. In addition to the downsides listed in the previous section, this creates a new inconsistency between static and instance, where they are otherwise generally analogous.

## Edge cases with built-in properties

Due to public static fields being ordinary data properties of a constructor function there are edge cases with names occupied by  built-in properties of the [`Function`](https://tc39.es/ecma262/#sec-ecmascript-function-objects) object. For example users may attempt to declare `static` class fields

1. [`constructor` or `prototype`](https://tc39.es/ecma262/#sec-makeconstructor),
1. [`name`](https://tc39.es/ecma262/#sec-setfunctionname) or [`length`](https://tc39.es/ecma262/#sec-setfunctionlength),
1. [`arguments` or `caller`](https://tc39.es/ecma262/#sec-addrestrictedfunctionproperties).

With respect to these and static public fields/methods the following is worth to note:

1. `static constructor` or `static prototype` produce [early errors](https://tc39.es/ecma262/#sec-class-definitions-static-semantics-early-errors) (see the [Note at Static Semantics: Constructor Method](https://tc39.es/ecma262/#sec-static-semantics-constructormethod))
1. `static name` or `static length`, if present in the class body, will be created with [`CreateDataProperty`](https://tc39.es/ecma262/#sec-createdataproperty) which results in a PropertyDescriptor
   ~~~
   { [[Value]]: V, [[Writable]]: true, [[Enumerable]]: true, [[Configurable]]: true }
   ~~~
   Otherwise, and only then, they are created and initialized implicitely by the runtime with [`SetFunctionName`](https://tc39.es/ecma262/#sec-setfunctionname) or [`SetFunctionLength`](https://tc39.es/ecma262/#sec-setfunctionlength) which results in a different PropertyDescriptor
   ~~~
   { [[Value]]: name, [[Writable]]: false, [[Enumerable]]: false, [[Configurable]]: true }
   ~~~
   The practical implications are that for `class MyClass {...}`
   -  `MyClass.name` / `MyClass.length` are writable by an assignment expression only if `MyClass` *has* a class body with a `static name` / `static length` class field

   - `MyClass.name` is initialized with the class name *if and only if* the class *has not* a body with a public `static name` field

   - `MyClass.length` is initialized *if and only if* the class *has not* a body with a public `static length` field


1. `arguments` and `caller` aren't properties of a constructor function derived from the  `class` keyword. Hence, there is no conflict when those are declared as public static class fields or methods *by the spec*.

   When using transpilers to "downlevel" `class MyClass {...}` syntax to pre ES2015 syntax, they *may* produce output using ordinary `function MyClass {}` syntax in which case `MyClass` *will have* `arguments` and `caller` properties at runtime, each with a property descriptor

   ~~~
   { [[Get]]: thrower, [[Set]]: thrower, [[Enumerable]]: false, [[Configurable]]: true }
   ~~~
   It's up to transpilers to handle this as part of their translation process.
