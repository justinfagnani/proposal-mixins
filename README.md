# Maximally Minimal Mixins
@justinfagnani

Last Updated: 2018-01-26

Status: Stage 1

[View TC39 Slides](https://docs.google.com/presentation/d/e/2PACX-1vSg1tyW2ALwSGR83qcKRQ4__T36CzvwOoPP9ywPpjuFxljmDaBHqKPAY6UFuLor4GHgxuB8WYj3OoIL/pub?start=false&loop=false&delayms=3000)

## Goals

  * Pave existing cowpath of the subclass factory pattern with a declarative syntax
  * Enable static analysis of mixins and mixin applications
  * Make mixin inheritance more attractive than it is now, to be an easy-to-use alternative to subclass inheritance
  * Orthogonality to classes - class features naturally apply to mixins
  * Provide space to evolve with protocol/trait-like features
  * Avoid problems with makeMethod, non-linear prototype chains, etc

## Subclass Factory Pattern

JavaScript arguably already has mixins, made possible by class expressions and extends expressions.

```js
let M = (base) => class extends base {
  field = 'abc';

  constructor() {
    super();
    // ...
  }

  method() {
    super.method();
    // ...
  }
}
class S {}
class C extends M(S) {}
```

Details and the benefits of this pattern are explained in this 2015 blog post: http://justinfagnani.com/2015/12/21/real-mixins-with-javascript-classes/

## Syntax Sugar for Subclass Factories

Maximally minimal mixins is simple syntax that de-sugars the following to the above:

```js
mixin M {
  field = 'abc';

  constructor() {
    super();
    // ...
  }

  method() {
    super.method();
    // ...
  }
}
class S {}
class C extends S with M {}
```

These mixins are strictly less powerful than imperative subclass factories. For instance, there's no way to imperatively mutate the prototype of the subclass, no way to do mixin composition, etc. These abilities are reserved for future evolution.

## Why?

If declarative mixins are just less-powerful sugar for a pattern that's already possible, the obvious question is: why add them at all?

### Enable Static Analysis

Possibly the most immediate benefit of a declarative syntax is improved toolability. The current mixin pattern is so imperative that it's very difficult for static analysis to recognize and properly understand the pattern.

Typed-variants of JavaScript, like Closure and TypeScript, have recently added support for the imperative pattern, but it's extremely fragile and cumbersome in both. See the Appendix below for examples of how code needs to be structured to satisfy various tools.

### Improve Ergonomics

Even outside of type checking, the ergonomics of declarative mixins are just better than the imperative subclass factory pattern.

### Encourage Class-Compatible Mixins

Subclass factory mixins leverage classes and integrate with the language in ways that other mixin patterns don't. Because subclass factory mixins use actual class declarations/expressions, they automatically get all new features of classes such as public and private fields and decorators.

Specific syntax for the pattern will encourage its use, and possibly allow platforms to ship mixins for the first time ever once they're an official part of the language.

As an example, the DOM standard recently added an `EventTarget` base class that can be used to inherit the web platform's `addEventListener()` and related APIs. This is great, but has the usual problems of sharing code via subclass inheritance. With language-level mixins there's a better chance that the DOM can add an `EventTargetMixin`.

### Enable Future Improvements

Like Maximally Minimal Classes, Maximally Minimal Mixins lay the foundation for further evolution. Other proposals in this space (of mixins, traits, interfaces, and protocols) have included additional features, like requirements, namespacing, aliasing, etc. These features can be added to classes and mixins, but it seems a simple starting point would help.

## Details

### Mixin Declaration

A _mixin_ is conceptually an abstract subclass - a class without a known superclass at declaration time. For maximal compatibility with current JavaScript language features, we define a mixin as a subclass factory - a function from a class to a new subclass. The declaration defines the difference between the superclass and generated subclass.

A mixin is declared with the new `mixin` keyword:

```js
mixin Name {
}
```

As with classes, `mixin` can be used in a declaration or expression. A mixin declaration or expression is syntactically similar to a class declaration or expression. It has an optional BindingIdentifier and a ClassBody.

A mixin defines an arrow function (so that they cannot be constructors) with one parameter, the _superclass_. Mixins return a new subclass of the argument as if the mixin body was evaluated as a class expression with the value of argument used as _superclass_ rather than evaluating ClassHeritage as in the ClassDefinitionEvaluation runtime semantics.

### Mixin Application

A mixin is used by _applying_ it to a concrete superclass with the `with` keyword. (This maybe confused with the `with` statement, so another keyword could be used)

`A with M` calls the mixin function `M` with the argument `A`. This can be used in the extends clause of a class declaration:

```js
class C extends A with M {}
```

Applying multiple mixins in one class declaration will be common, so we could offer special syntax for a list of mixins:

```js
class C extends A with M1, M2, M3 {}
```

### `Symbol.mixin`

Like the `constructor` property, it should be possible to identify the mixin used to create a prototype. `constructor` itself is not suitable to identify the _mixin_, since it will refer to the constructor generated by the mixin application. Mixin application will set the `Symbol.mixin` property of the constructor's `prototype` to reference the mixin function.

```js
mixin M {}
class C extends A with M {}
Object.getPrototypeOf(C).prototype.hasOwnProperty(Symbol.mixin); // true
Object.getPrototypeOf(C).prototype[Symbol.mixin] === M; // true
```

### `instanceof`

`instanceof` can be defined so that it works with mixins:

```js
mixin M {}
class C extends Object with M {}
new C() instanceof M; // true
```

### Mixin Composition

Since mixins are just functions, they naturally compose with function syntax:

```js
mixin A {}
mixin B {}
let AB = (superclass) => A(B(superclass));
```

Composition is commonly desired inline with declaration. We can allow mixins to have an `extends` clause like classes, but which must evaluate to a mixin:

```js
mixin A extends B {}
```

### Desugaring

#### Mixin Declaration

```js
mixin M {
  #a;
  b;
  c();
}
```

Desugars to:

```js
Symbol.mixin = Symbol.mixin || Symbol('mixin');

let M = (superclass) => {
  const _M = class extends superclass {
    #a;
    b;
    c() {}
  }
  Object.defineProperty(_M.prototype, Symbol.mixin, {value: M});
  return _M;
};

Object.defineProperty(M, 'prototype', {});

Object.defineProperty(M, Symbol.hasInstance, {
  value(o) {
    while (o !== undefined) {
      if (o.hasOwnProperty(Symbol.mixin) && o[Symbol.mixin] === this) {
        return true;
      }
      o = Object.getPrototypeOf(o);
    }
    return false;
  }
});
```

#### Mixin Application

```js
C with M;
```

Desugars to:

```js
M(C)
```

Therefore:

```js
class A extends B with M {
  // ...
}
```

Desugars to:

```js
class extends M(B) {
  // ...
}
```

#### Mixin Composition

```js
mixin A extends B {
  method() {}
}
```

Desugars to:

```js
let A = (superclass) => class extends B(superclass) {
  method() {}
}
```

## Interesting Notes

### Orthogonality to Classes

One of the main implications of this proposal is that mixins are mostly orthogonal to classes. Mixins are a way to declare abstract subclasses and leverage class syntax and semantics. All new declarative features of classes are automatically inherited by mixins.

This orthogonality can be used in the evolution of both classes and mixins. Some parts of features discussed and/or proposed previously could be broken out and added classes, independently of mixins.

### Mixin methods are copied on application

It sounds like previous discussions around sharing prototypes/methods has had to deal with the question of function identity and home objects. This proposal side-steps these questions by making a copy of methods via the evaluation of a class expression on each mixin application.

This means that methods defined in mixins get fresh copies for each mixin application:

```js
mixin M {
  method() {}
}

class A extends Object with M {}
class B extends Object with M {}

A.prototype.method === B.prototype.method; // false
```

### Mixins are functions, but not constructors

Mixins syntactically may look like classes, and users may expect them to have a `.prototype` property, but they don't. It would be useful to have access to a prototype for the mixin, but in this proposal there's no easy way to allow that - the prototype doesn't exist until application time. It's really a _potential_ prototype.

To reserve space for a future feature where they is a prototype object at mixin declaration time, we could define `prototype` property of mixins as `undefined`, not-writable, not-configurable. This is shown in the desugaring of the mixin declaration above.

## Evolution

Maximally Minimal Mixins aim to allow for evolution, in the same way as Maximally Minimal Classes successfully have, including adding features from many previously proposed mixin or related features.

### First-Class Protocols

See: https://github.com/michaelficarra/proposal-first-class-protocols

The First-Class Protocols proposal adds a few interesting features to classes/inheritance:

  * Interfaces
  * Requirements
  * Mixin-like composition
  * Automatically namespaced members

Requirements could be added to classes and mixins (see Traits below).

Automatically namespaced members (members that declare a Symbol and a member named by that Symbol in one go), seem generally useful independent of Protocols. If they can be added to classes, then mixins would get them naturally.

This can be done today with a helper, and syntax could possibly be added later:

```js
let makeSymbol = (o, name) => o[name] = Symbol(name);
class A {
  [makeSymbol(A, 'myMethod')]() { /* ... */ }
}
class B extends A {
  [A.myMethod]() {
    super[A.myMethod]();
    // ...
  }
}
```

### Traits

See: http://zqsmm.qiniucdn.com/data/20110512092812/index.html#Traits

#### Requirements

Traits can require that the class implement certain members:

```js
trait ComparableTrait {
  requires lessThan;
  requires equals;
}
```

Maximally minimal mixins leave the door open to add such features to classes in general, and therefore to mixins.

Roughly, a `requires`-like modifier on class members would presumably be tied to some notion of abstract classes. Requirements would be checked when declaring a concrete subclass. Mixins with requirements would be checked when applying the mixin to a concrete class.

```js
abstract class A {
  require x;
}
abstract class B extends A {} // OK
class C extends A {} // Error: Must implement foo;
class D extends A { // OK
  foo() {}
}
```

#### Aliasing

_TBD: Declarative aliasing / renaming requires some syntax to describe the renaming..._

## Appendix: Analysis of Subclass Factories

#### TypeScript

TypeScript added support for mixins in 2.2. It models mixins as intersections of the argument to the subclass factory and the class defined in the factory. This isn't entirely accurate as it doesn't model override semantics, but it's usually close enough.

```ts
export type Constructor<T=object> = new(...args: any[]) => T;

class S {
  baseMethod() {/*...*/}
}

let M = <T extends Constructor>(base: T) => class extends base {
  constructor(...args: any[]) {
    super(...args);
  }

  mixinMethod() {/*...*/}
};

class C extends M(Object) {
  subclassMethod() {/*...*/}
}

let c = new C();
```

TypeScript correctly infers that `c` has members inherited from `S`, `M`, and `C`.

In TypeScript classes define a name in both the value and type namespaces, but functions don't, so `M` is a value, but not a type. If you want to use `M` as a type you have to duplicate the declaration with an interface, and cast the new class to the interface:

```ts
interface M {
  mixinMethod();
}

let M = <T extends Constructor>(base: T) => class extends base {
  constructor(...args: any[]) {
    super(...args);
  }

  mixinMethod() {/*...*/}
} as T & Constructor<M>;
```

This is obviously more cumbersome as the mixin interface becomes larger.

#### Closure

Closure requires more boilerplate to describe the mixin and application to the compiler:

```js
class S {
  baseMethod() {/*...*/}
}

let M = (base) => {
  /*
   * @implements {MInterface}
   */
  class M extends base {
    constructor(...args: any[]) {
      super(...args);
    }

    mixinMethod() {/*...*/}
  }
  return M;
};

/**
 * In Closure the interface declaration is required.
 * @interface
 */
function MInterface(){}

/** @return {void} */
MInterface.prototype.mixinMethod = function(){};

/**
 * Closure requires you to separately define and cast the mixin application.
 * @constructor
 * @extends S
 * @implements {MInterface}
 */
const CBase = M(Base);

class C extends CBase {
  subclassMethod() {/*...*/}
}

let c = new C();
```

#### Polymer Analyzer

The Polymer Analyzer has specific support for analyzing mixins via custom JSDoc annotations:

```js
class S {
  baseMethod() {/*...*/}
}

/**
 * @mixinFunction 
 */
let M = (base) => {
  /**
   * @mixinClass
   */
  class M extends base {
    constructor(...args: any[]) {
      super(...args);
    }

    mixinMethod() {/*...*/}
  }
  return M;
};

/**
 * @extends Object
 * @appliesMixin M
 */
class C extends M(Object) {
  subclassMethod() {/*...*/}
}
```
