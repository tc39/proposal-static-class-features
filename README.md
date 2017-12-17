# Static class features

Stage 2

This proposal tracks the "static" (i.e., of the constructor) aspects of the [class fields](http://github.com/tc39/proposal-class-fields) and [private methods](https://github.com/tc39/proposal-private-methods) proposals. In the November 2017 TC39 meeting, the static dimensions of these proposals were demoted to Stage 2, to be broken out into a separate proposal while the instance dimensions remain at Stage 3. This repository is created to explore the aspects which are still at Stage 2, to determine how to proceed to Stage 3 with some version of them.

## Features proposed for ECMAScript

### Static public fields

This proposal will be focusing on adding only static public fields. Like static public methods, static public fields take a common idiom which was possible to write without class syntax and make it more ergonomic, have more declarative-feeling syntax (although the semantics are quite imperative), and allow free ordering with other class elements.

#### Semantics

Define an own property on the constructor which is set to the value of the initializer expression. The initializer is executed after the class TDZ is ended.

Alternate semantics are discussed in [Issue #2](https://github.com/tc39/proposal-static-class-features/issues/2).

#### Use case

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

## Features not proposed for further advancement

Although they were present in the previous class fields and private methods proposals, static private class features are *not* proposed here for further advacement due to the issues discussed in [Issue #1](https://github.com/tc39/proposal-static-class-features/issues/1). There is a combination of an inherent difficult with determining reasonable semantics and lack of motivation due to adequate idioms for these use cases already existing. Please join the discussion in that issue to share your point of view.

### Static private methods

#### Semantics

The class has an own private method, similar to private instance methods. This method is not installed on subclasses.

#### Use case

```js
class Point {
  #x;
  #y;
  
  // ...constructor, public methods, etc...
  
  static #equal(a, b) {
    return a.#x === b.#x && a.#y === b.#y
  }
  
  static assertEqual(a, b) {
    if (!Point.#equal(a, b)) alert("Different!")
  }
}
```

In [Issue #1](https://github.com/tc39/proposal-static-class-features/issues/1), there is further discussion about whether this feature is well-motivated. In particular, static private methods can typically be replaced by either lexically scoped function declarations outside the class declaration, or by private instance methods.

### Static private fields

#### Semantics

The class has an own private field, similar to private instance fields. This field is not installed on subclasses, and the initializer is only ever evaluated once. As with static public fields, the initializer is evaluated after the TDZ has ended.

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

In [Issue #1](https://github.com/tc39/proposal-static-class-features/issues/1), there is further discussion about whether this feature is well-motivated. In particular, static private fields can typically be subsumed by lexically scoped variables outside the class declaration.
