# JavaScript Prototypes

The prototype is an essential mechanism of JavaScript. Even with the advent
of `class`, it is not superseded and remains just as fundamental. I highly
recommend [Jeremy Kyle's book](https://github.com/getify/You-Dont-Know-JS/blob/master/this%20%26%20object%20prototypes/ch5.md) for a more detailed treatment of the subject. In this
post, I attempt to summarize the prototype mechanism. I won't discuss the
newer `class` mechanics as they don't change how prototypes work.

## What is a Prototype ?

A prototype is just an object that is referenced by _other_ objects via an
internal property. This allows one object access to properties (including
functions) which are actually set on _another_ object. The JavaScript spec
calls this internal property "`[[Prototype]]` internal slot"
[(1)](http://www.ecma-international.org/ecma-262/6.0/index.html#sec-ordinary-object-internal-methods-and-internal-slots)
and it is _not_ the same thing as the `prototype` property. When we say that
"`foo` is prototype-linked to `bar`" it means that `foo`'s internal
`[[Prototype]]` is a reference to `bar`.

## The Prototype Chain

When we link `foo` and `bar`, `foo` will not only have access to `bar's`
properties, but also to the properties on `bar`'s `[[Prototype]]`, and so on
until one of the prototypes has a `null` `[[Prototype]]`. This forms the
prototype chain. The last "link" in this chain is normally
`Object.prototype`, and it's `[[Prototype]]` is `null`.

How can we create a "prototype chain" ? There are multiple ways; one would be
the familiar pattern of adding properties to the `prototype` of a `function`
object. In JavaScript, functions are also objects, so you can add properties
to them. We don't have to add `prototype` explicitly though, JavaScript does
it for us when we declare a function. Later, if we call the function with
`new`, the created object will be prototype-linked to the function's
"prototype" property:

```js
function foo() {} // foo is a function object
typeof foo.prototype // => "object" (implicitly created by JavaScript)
foo.prototype.x = 42 // linked objects will be able to access 'x'
foo.y = 100 // because functions are objects, so we can set props on them
const bar = new foo() // bar is prototype-linked to foo.prototype
bar.x // => 42 ("inherited" from foo's prototype)
bar.y // => undefined
```

So in the code above we have two prototype-linked objects. However, we didn't
link `foo` and `bar` directly. If we add a `y` property to `foo`, `bar` won't
"inherit" it. What we're actually linking is _another_ object, the one
created implicitly by JavaScript and accessible at `foo.prototype`.

## Accessing Linked Prototype Objects

Here's where things can get confusing: non-function objects don't normally
have a "prototype" property, even though they _do_ have a prototype. So how
can we get a reference to this object ? In other words, how can we get the
value of the `[[Prototype]]` internal property ? The old and deprecated way
is with `__proto__`; the new and recommended way is by using
`Object.getPrototypeOf`:

```js
function foo() {}
const bar = new foo()
foo.prototype // => {} - an empty object
bar.prototype // => undefined
bar.__proto__ === foo.prototype // => true
bar.__proto__ === Object.getPrototypeOf(bar) // => true
foo.prototype !== foo.__proto__ // => true - two completely different objects
```

## Linking Objects Explicitly

Note how we had to create a function object, but the properties which we want
to be "inherited" are not set on it directly, but on another object
JavaScript creates automatically and attaches it to the function object as
the value of the "prototype" property. But can't we just link two objects
directly ? Yes we can, using `Object.create`:

```js
const foo = { x: 42 }
const bar = Object.create(foo) // bar is prototype-linked to foo
bar.x // => 42 - inherited from foo, through the prototype chain
Object.getPrototypeOf(bar) === foo // true - direct, explicit link !
```

We can also use `Object.setPrototypeOf` to re-link existing objects, but note
that this is not recommended because [changing the prototype is an expensive
operation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf):

```js
const foo = { x: 42 }
const bar = {}
Object.setPrototypeOf(bar, foo)
Object.getPrototypeOf(bar) === foo // => true
bar.x // => 42
```

## Implementing Object-Oriented Prototypal Inheritance

Prototypes are often used "in the wild" to implement object-oriented patterns
common in other languages. So you'd see something like this:

```js
function Foo (initialCount) {
  this.count = initialCount
}
Foo.prototype.incrementCount = function () {
  this.count++
}

const instance = new Foo(0)
```

`Foo` is just a normal function, but if we call with `new`, it will act as a
constructor. In JavaScript, **any** function becomes a constructor when
called with `new`. As we saw previously, even a function like `function foo = {}` will still return
a new object, prototype-linked to `foo.prototype`.

Just before executing a constructor call, JavaScript will create a brand new
object and it will set `this` to point to it for the duration of the
constructor function's execution. It will also set the object's internal
`[[Prototype]]` property to point to the function's `prototype`. At the end
of the execution, the return value of the function is this newly created
object; the class _instance_. JavaScript will automatically set it as the
return value. Note that this happens only if a function is called with `new`:

```js
const o1 = new Foo(10) // o1: { count: 42 } 
o1.incrementCount()
o1.count // => 11
const o2 = Foo() // o2: undefined; side-effect: window.x === 42
```

To make our JavaScript pseudo-classes behave as in most OO languages, we
need to do a bit more work. One such thing is calling the "parent"
constructor; when the sub-class is instantiated, one would expect the
super-class constructor to be called. Therefore, we need to make sure to call
the parent function, which is also the super-class constructor:

```js
function Bar (initialCount) {
  Foo.call(this, initialCount)
}
```
We also need a way for `Bar` to inherit from `Foo`; this is done via the
prototype chain:

```js
Bar.prototype = Object.create(Foo)
```

Why not just `Bar.prototype = Foo.prototype` ? Because then the "prototype"
property of both function objects would reference the exact same object. If
you add something to `Bar`'s prototype, it would also be inherited by all
`Foo` instances. This is not desirable because super classes should not
inherit from their sub-classes.

We could also try `Bar.prototype = new Foo()`; this would also make
inheritance happen. However, this can have unwanted side-effects. Let's say
we create a `Foo` instance when the user clicks a button, and we log that
action in the `Foo` function. However, if we check the logs, we will see that
we always get an entry without the user doing anything - it's because we run
the `Foo` function to setup `Bar`'s prototype.

## The `constructor` Property

We saw that when JS creates a function object, it also creates the prototype
object. This object has a "constructor" property, and it is a reference to
the function object. Because it's on the prototype, it will be inherited
by objects created by the function with `new`:

```js
function foo() {}
foo.prototype.constructor === foo // => true
const obj = new foo()
obj.constructor === foo // => true
```

However, it's not safe to assume that if `obj.constructor === foo`, it means
the object was created by `foo`. That's because the "prototype" property of
function objects can be changed, so `prototype.constructor` can point to anything:

```js
function foo() {}
function bar() {}
foo.prototype = {
  constructor: bar
}
const obj = new foo()
obj.constructor === bar // => true - `constructor` is inherited
```

## Setting Properties

In most cases, when setting the value of a property, the result will be the
creation or modification of a property on the target object (`bar` in the
examples below). But there are two situations in which that will not happen,
at least not as expected. Both of them occur when JS finds an object in the
target object's prototype chain which already has a property with the same
name as the one we're trying to set/create on the target object.

### Situation 1 - The Property is a [setter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/set)

In JS, if we want to run a function when setting a property, we can can use a
setter. In this case, **JS will call the setter**. The setter function can
use `this`, which will be a reference to the target object, _not_ the linked
object:

```js
const foo = {
  set myProp(value) {
    this.x = value + 1
  }
}
const bar = Object.create(foo) // 'bar' is the target object
bar.myProp = 10 // the setter will be called; `this` will be `bar`
bar.myProp // => undefined
bar.x // => 11
```

### Situation 2 - The Property is Read-only

We can create read-only properties using [Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty):
```js
const foo = {}
Object.defineProperty(foo, 'myProp', {
  value: 42,
  writable: false,
})
foo.myProp // => 42
foo.myProp = 100 // ignored, or throws in "strict" mode
foo.myProp // => 42
```
If `foo` is part of the prototype chain for `bar`, then we **can't set a
"myProp" property on `bar`**. In other words, a read-only prop somewhere down
the line in the prototype chain will prevent assigning a property with that
name on the target object. In `strict` mode, an exception will be thrown;
otherwise the assignment will just silently fail.

```js
// continuing previous example
const bar = Object.create(foo)
bar.myProp = 100 // nothing happens !
bar.myProp // => 42
```
This is to prevent _shadowing_ - which is what happens when two
prototype-linked objects have properties with the same name; when reading
the value of that prop, the first one will be returned, thus "shadowing" the
second one. I'm not sure why this is done _just_ for read-only properties,
but that's how it is. Essentially, it means that if an object has a 'myProp'
property that is read-only, then it can't _shadowed_ by objects which will
add this object to their prototype chain. But, if the property is writable,
you're free to shadow it !

```js
const foo = {}
Object.defineProperties(foo, {
  'myReadOnlyProp': { value: 42, writable: false },
  'myWritableProp': { value: 10, writable: true }
})
const bar = Object.create(foo)
bar.myReadOnlyProp = 100 // can't shadow read-only props, nothing happens
bar.myReadOnlyProp // => 42
bar.myWritableProp = 100 // can shadow writable props; prop created on `bar`
foo.myWritableProp // => 10; the shadowed property is unaffected
bar.myWritableProp // => 100
```

## Summary

JS automatically adds a "prototype" property to function objects. When we
instantiate objects by calling the function as a constructor (with `new`),
the created objects will have access to properties on the `prototype` object.
One such property is the `constructor` property, added by JS and which
normally points to the function. We can create prototype-linked objects
directly, with `Object.create`.

We can use `__proto__` or `Object.getPrototypeOf` to get a reference to the
object referenced by the internal `[[Prototype]]` slot, and we can use either
`__proto__` or `Object.setPrototypeOf` to set it's value.

Reading the value of a property that is not on the target object will
traverse the prototype chain and return the first one found, or `undefined`,
whereas the algorithm for setting such properties is more complex, calling
the first found setter and preventing the shadowing of read-only properties.

## Comments on the OO Paradigm in JavaScript

Jeremy Kyle argues that the OO paradigm is not well suited for the dynamic
nature of JavaScript, and prefers linking objects directly. I agree, but I
think it is a lost cause - the overwhelming majority of developers use OO,
either with `class` or by adding stuff to function object prototypes.
However, whether you prefer OO or linking objects directly, the prototype
chain is essential, as `class` is mostly just syntactic sugar on top of the
prototype mechanism.