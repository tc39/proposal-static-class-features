# Static public fields

Champions: Shu-Yu Guo, Daniel Ehrenberg

Stage 2

This proposal adds public static fields as previously proposed. Public static fields are own data properties of the constructor. 

## Motivation

Like static public methods, static public fields take a common idiom which was possible to write without class syntax and make it more ergonomic, have more declarative-feeling syntax (although the semantics are quite imperative), and allow free ordering with other class elements.

Declaring static properties in the class body is hoped to be cleaner and doing a better job of meeting programmer expectations of what classes should be for. The latter workaround is a somewhat common idiom, and it would be a nice convenience for programmers if the property declaration could be lifted into the class body, matching how methods are placed there.

## Example

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

## Semantics

Define an own property on the constructor which is set to the value of the initializer expression. The initializer is evaluated in a scope where the binding of the class is available--unlike in computed property names, the class can be referred to from inside initializers without leading to a ReferenceError. The `this` value in the initializer is the constructor.

See [STATICPUBLIC.md](https://github.com/tc39/proposal-static-class-features/blob/master/STATICPUBLIC.md) for an explanation of some of the edge cases and alternatives considered.

## Follow-on proposals

This proposal is designed to be compatible with several possible follow-on proposals, which are documented at [FOLLOWONS.md](https://github.com/tc39/proposal-static-class-features/blob/master/FOLLOWONS.md).

## Proposal status

This proposal is at Stage 2.

This proposal was created to track the "static" (i.e., of the constructor) aspects of the [class fields](http://github.com/tc39/proposal-class-fields) and [private methods](https://github.com/tc39/proposal-private-methods) proposals, namely static public fields, static private fields, and static private methods. In the November 2017 TC39 meeting, the static dimensions of these proposals were demoted to Stage 2, to be broken out into a separate proposal while the instance dimensions remain at Stage 3.

Draft specification text is published at [https://tc39.github.io/proposal-static-class-features/].

Static public fields are implemented in Babel and V8 behind a flag, and had test262 tests written (though currently have been removed due to this proposal being at Stage 2).
