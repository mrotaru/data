# JavaScript Prototypes

There is hardyly anything more important in JavaScript than understanding how
the prototype chain works. Even if we can use the `class` syntax nowadays, it
still uses prototypes under the hood. Understanding prototypes is, perhaps,
even _more_ important when using `class` because if you're coming from a more
traditional object-oriented language, such as Java or C++, you might make some
assumptions that are not true.

> It ain't what you don't know that gets you into trouble. It's what you know for sure that just ain't so.

It is intuitivelly simple - you just add properties to a function's
`prototype` and they become available to objects created when that function
is called with `new`. But how does that actually work ? What's the
difference between `prototype`, `__proto__`, and how do they relate to
functions like `getPrototypeOf` and `setPrototypeOf` ? How does the
`constructor` property work ? The answers to these questions might not be as
simple as one expects.

## The Prototype Chain

First thing first; how can we create a "prototype chain" ? There are multiple
ways; one would be the familiar pattern of adding properties to the
`prototype` of a `function` object. In JavaScript, functions are also
objects, so you can add properties to them. We don't have to add `prototype`
explicitly though, JavaScript does it for us when we declare a function.
Later, if we call the function with `new`, the created object will be
prototype-linked to `foo`'s prototype:

```js
function foo {}
typeof foo.prototype // => "object" (implicitly created by JavaScript)
foo.prototype.x = 42 // linked objects can access these properties 
foo.y = 100 // foo is an object, so we can set any props we want on it
const bar = new foo() // bar is prototype-lined to foo's prototype
bar.x // => 42 ("inherited" from foo's prototype)
bar.y // => undefined
```

So in the code above we have two prototpye-linked objects. However, we didn't
link `foo` and `bar` directly. If we add a `y` property to `foo`, `bar` won't
"inherit" it. What we're actually linking is _another_ object, the one
created implicitly by JavaScript and accessible at `foo.prototype`.

## Accessing linked prototype objects

Here's where things can get confusing: non-function objects don't
normally have a "prototype" property, even though they _do_ have a prototype.
The JavaScript spec calls it the "`[[Prototype]]` internal slot" [(1)](http://www.ecma-international.org/ecma-262/6.0/index.html#sec-ordinary-object-internal-methods-and-internal-slots), and that's
where it stores a reference to the object that will form the first link in
the prototype chain for that particular object. When we say that "`foo` is
prototype-linked to `bar`" it means that `foo`'s internal `[[Prototype]]` is
a reference to `bar`.

So if we want to grab a reference to this object, in other words, get the
value of this `[[Prototype]]` thing, how do we do that ? The old and
deprecated way is with `__proto__`; the new and recommended way is by using
`Object.getPrototypeOf`:

```js
function foo {}
const bar = new foo()
foo.prototype // => {} - an empty object
bar.prototype // => undefined
bar.__proto__ === foo.prototype // => true
bar.__proto__ === Object.getPrototypeOf(bar) // => true
foo.prototype !== foo.__proto__ // => true - two completely different objects
```

## Linking objects explicitly

Note how we had to create a function object, but the properties which we want
to be "inherited" are not set on it directly, but on another object
JavaScript creates automatically and attaches it to the function object as
the value of the "prototype" property. But can't we just link two objects
directly ? Yes we can, using `Object.create`:

```js
const foo = { x: 42 }
const bar = Object.create(foo)
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

## Implementing prototypal inheritance

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
called with `new`; even a function like `function foo = {}` will still return
a new object, prototype-linked to `foo.prototype`:

```js
function foo {}
const obj = new foo()
Object.getPrototypeOf(obj) === foo.prototype // => true
```

When a function is caled as a constructor, JavaScript will create a brand new
object (the one that will be returned) and it will set `this` to point to it
for the duration of the function call. It will also set the object's internal
`[[Prototype]]` property to point to the function's `prototype`.

When you instantiate a sub-class, in most OO languages the super-class
constructor is automatically called - unless you define a constructor for the
subclass. Therefore, need to make sure to call the parent function:

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

## The `constructor` property

We saw that when JS creates a function object, it also creates the prototype
object. This object has a "constructor" property, and it is a reference to
the function object. Because it's on the prototype, it will be accessible
to objects created by the function with `new`:

```js
function foo() {}
foo.prototype.constructor === foo // => true
const obj = new foo()
obj.constructor === foo // => true
```

However, it's not safe to assume that if `obj.constructor === foo`, it means
the object was created by `foo`. That's because the "prototype" property of
function objects can be changed, so `prototype.construcor` can point to anything:

```js
function foo() {}
function bar() {}
foo.prototype = {
  constructor: bar
}
const obj = new foo()
obj.constructor === bar // => true
```

## Setting properties

We saw that the prototype chain is actually pretty simple to understand, at
least when it comes to _getting_ properties - just traverse the prototype
chain, and return the first found value.

When we're setting the value of a property, it's a bit more complicated. In
most situations, the property will be created on the target object. But
before doing that, JS will traverse the prototype chain, and check if any of
the linked objects has a property with the same name. And if it finds such an
object, there are two situations in which it might not do what you expect.

### Situation 1 - the property is a [setter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/set)

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
const bar = Object.create(foo)
bar.myProp = 10 // the setter will be called; `this` will be `bar`
bar.myProp // => undefined
bar.x // => 11
```

### Situation 2 - the property is read-only

We can create read-only properties like using [Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty):
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
otherwise the assinment will just silently fail.

```js
// continuing previous example
const bar = Object.create(foo)
bar.myProp = 42 // nothing happens !
bar.myProp // => undefined
```
This is to prevent _shadowing_ - which is what happens when two
prototype-linked objects have properties with the same name; when reading
the value of that prop, the first one will be returned, thus "shadowing" the
second one. I'm not sure why this is done _just_ for read-only properties,
but that's how it is. Essentially, it means that if an object 'myProp'
property that is read-only, then it can't _shadowed_ by objects which will
add this object to their prototype chain. But, if the property is readable,
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
