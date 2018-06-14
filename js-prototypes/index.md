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

Why not just `Bar.prototype = Foo.prototype` ? Because then both functions
would reference the same object as their prototype. If you add something to
`Bar`'s prototype, it would also be inherited by all `Foo` instances. This is
not desirable because super classes should not inherit from their
sub-classes.

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
function objects can be changed:

```js
function foo() {}
foo.prototype = {
  constructor: "i'm not even a function!"
}
const obj = new foo()
obj.constructor // => "I'm not even a function!"
```

## Getting and setting prototype properties

But how does it work when trying to _set_ it ? You might think that it works
the same as when trying to _get_ the value of a property which the object
does not have, but one of the prototype-linked objects does. So if you try to
set the value of such a property, you end up modifying the value found while
traversing the prototype chain. So, in the example below, the statement
`foo.x = 43` would modify `bar.x`, because `foo` does not have an `x`
property, but `bar` does and it's linked to `foo`:

```js
const bar = { x: 42 }
const foo = Object.create(bar)
foo.x = 43
```
### Situation 1 - one of the prototypes has a setter

### Situation 2 - one of the prototypes 

So what does JS do for `get` and `set` operations, when the targeted property already
exists on one of the objects in the prototype chain:

- `get`: return the first found property value
- `set`: **don't set the value of the existing property** - set it on the target object

## Introspection

## Comparing JavaScript `class`es with classes in other languages

With a good understanding of how inheritance happens in JavaScript, we can
now compare it to other object-oriented languages. We will see that they are
fundamentally different approaches that really don't have much in common,
even thought the syntax is almost identical.
