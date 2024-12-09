# Main Goal
1. Allow using `by` delegation with instance properties
2. Allow using `by` delegation on a method-by-method basis

## Instance Properties
```kotlin
interface IFoo {
	fun foo()
} // etc

class Example : IFoo by myFoo, IBar by myBar, IBaz by myBaz {
	val myFoo: IFoo = IFoo.Impl()
	var myBar = IBar.Impl()
	lateinit var myBaz: IBaz
}
// compiles to:
class Example : IFoo, IBar, IBaz {
	val myFoo = IFoo.Impl()
	var myBar = IBar.Impl()
	lateinit var myBaz: IBaz
	
	override fun foo() = myFoo.foo()
	override fun bar() = myBar.bar()
	override fun baz() = myBaz.baz() // throws normally if not initialized before being called, just like manual delegation
}
```

The only point is to have it be what it is: shorthand for manual redirection boilerplate, not any kind of guarded language construct. 

This is literally what Kotlin does already, just solely with hidden `val`s named stuff like `$$delegate1`.

This also allows us to finally resolve conflicting methods without having to manually implement redirection for the whole thing ourselves:

```kotlin
interface Deserializable {
	fun restoreState(state: StateHolder)
}
class ArmMover : ArmMovable, Deserializable {
	override fun restoreState(state: StateHolder) {
		this.armPosition = state.getTag("armPosition")
	}
}
class LegMover : LegMovable, Deserializable {
	override fun restoreState(state: StateHolder) {
		this.legPosition = state.getTag("legPosition")
	}
}

class MainBody : Deserializable, ArmMovable by ArmMover(), LegMovable by LegMover() {
	override fun restoreState(state: StateHolder) {
		this.heartBpm = state.getTag("heartBpm")
		// can't reference either anonymous property despite the fact that they exist
	}
}

// modified syntax, still essentially compiles to the same thing but exposed

class MainBody : Deserializable, ArmMovable by armMover, LegMovable by legMover {
	val armMover = ArmMover()
	val legMover = LegMover()
	
	override fun restoreState(state: StateHolder) {
		this.heartBpm = state.getTag("heartBpm")
		armMover.restoreState(state)
		legMover.restoreState(state)
	}
}
```

AND YES I KNOW YOU CAN DO THIS: 
```kotlin
class MainBody(val armMover: ArmMover, val legMover: legMover) : Deserializable, ArmMovable by armMover, LegMovable by legMover {
	override fun restoreState(state: StateHolder) {
		this.heartBpm = state.getTag("heartBpm")
		armMover.restoreState(state)
		legMover.restoreState(state)
	}
}
```
but sometimes you're forced to have a billion constructor parameters already and there's never any instance where you'd want to pass a different component in, so adding constructor parameters for _every single interface_ is stupid

## 2. `by` with Instance Methods

Solving the diamond problem:

```kotlin
interface Foo1 : Foo {
	override fun foo() {
		println("one")
	}
}
interface Foo2 : Foo1 {
	override fun foo() {
		println("two")
	}
}
interface Bar : Foo1 {
	fun bar()
	
	/* inherited from Foo1:
	override fun foo() = println("one")
	 */
}
interface Qux : Foo2 {
	fun qux()
	
	/* inherited from Foo2: 
	override fun foo() = println("two")
	 */
}

class MyThing : Bar by Bar.Impl(), Qux by Qux.Impl() {
	// Ok so which `foo` do we use? Cause both distinct interface trees declare Foo#foo, but they default it differently.
}

// to resolve this vanilla, we have to manually redirect for the ENTIRE TREE just to resolve ONE single method conflict:
class MyThing : Bar, Qux {
	val myBar = Bar.Impl()
	val myQux = Qux.Impl()
	
	override fun foo() = myBar.foo() // specifically use Bar's version of Foo#foo
	
	override fun bar() = myBar.bar()
	override fun qux() = myQux.qux()
}

// With the altered syntax, though:
class MyThing : Bar by myBar, Qux by myQux {
	val myBar = Bar.Impl()
	val myQux = Qux.Impl()
	
	// exactly the same thing, but with only two words instead of explicitly (since we already know what we're overriding), 
	// and only having to resolve the conflicting method(s) instead of the entire redirection tree
	override fun foo() by myBar
}
```

hmm, actually! we don't even need to make them instance variables!
```kotlin
class MyThing : Bar by Bar.Impl(), Qux by Qux.Impl() {
	// Specify that we're using Bar#foo, AKA Foo1#foo
	override fun foo() by Bar 
	// compiles to:
	override fun foo() = `$$delegate$2$bar$$`.foo() // or whatever kotlin's hidden property name is
	
	// Alt: could we even do this, since that's up there in the tree? 
	override fun foo() by Foo1
	// It would need to find the first redirected property that has that in its tree. 
	// Not sure if this is too brittle... make this a stretch goal
	// also this would need a warning for conflicting methods and an "unused" warning if redundant
	
}
```

# Side Goal: `as`, and TRUE delegation
What is the point of this? 

Well, current delegation is actually fully inherited, with the source object taking on all of the delegated interface's members.

```kotlin
interface Part1 {
	fun partMethod()
}

class Foo : Part1 by Part1.Impl() {}

// what's actually happening:
class Foo : Part1 {
	private val `$$part1delegate`: Part1 = Part1.Impl()
	
	override fun partMethod() {
		return `$$part1delegate`.part1Method()
	}
}
```

This is fine, _except_ when there are method name conflicts:
```kotlin
interface Part2 {
	fun partMethod()
}

class Bar : Part1 by Part1.Impl(), Part2 by Part2.Impl() // CAN'T DO THIS BECAUSE OF NAMING CONFLICT
```

One way to resolve this is to use the more-_verbose_ (and arguably _real_ version of the) delegation pattern:

```kotlin
interface IHavePart1 {
	fun asPart1(): Part1
}

interface IHavePart2 {
	fun asPart2(): Part2
}


class Baz : IHavePart1, IHavePart2 {
	val ourPart1 = Part1.Impl()
	val ourPart2 = Part2.Impl()
	
	override fun asPart1(): Part1 = ourPart1
	override fun asPart2(): Part2 = ourPart2
}
```

This makes a whole lot more sense when using the _true_ delegation pattern, _which includes a reference to the original object_:
```kotlin
class Block : IAmRotatable {
	var facing: Direction = NORTH
	
	inner class RotatableView {
		override fun rotate() {
			facing = SOUTH
		}
	}
	
	val rotatable: Rotatable = RotatableView()
	
	override fun asRotatable() = rotatable
}
```
When we do this, what is this new object? It doesn't have any state of its own, it's barely a real object, it's just a...

It's just a _delegate_, for that behavior.  You tell it to do something, and it passes that on to the original source.

But more generally, and on a higher level, it is a _**VIEW** OF THE ORIGINAL SOURCE OBJECT_
Just like upcasting to Rotatable, you are stripping down the unique properties of the object and **viewing it** as a Rotatable.  It's just like getting a Set from a Map: despite how stupid that pattern is, it's an example of this same behavior - exposing the exact same source object in a _different way_!

This touches on something in higher-level programming known as "Optics": rather than outright transforming one object into a different type just to have a new set of methods to interact with it, we _view it in a different way_.  Just like `Set.add(x)` being turned into `Map.put(x, true)` on the backend, interacting with the perfectly normal set in front of you interacts with the original map!

My mental image has always been like, this fuzzy thing of having a guy and a tiny man coming out of his chest that's still connected to the larger guy, and messing with the little man does something to the bigger dude.  My mental image is also very influenced by that one doctor who episode with the shapeshifting robot man piloted by tiny people - it's a whole thing and really abstract, don't read into it - but you get the general idea.

Or like a _remote control_ for a larger machine: you could spin the dials and everything on the main control panel, _or_ you could take the remote and bring it with you. You're taking a "remote" object with you, but the remote doesn't actually have anything in itself to worry about, it's just a _view_ of the original machine's state! And can be modified as such to change the original machine, without having any state of its own related to the machine

With that idea in mind, we don't actually need inner classes to stay "inner", do we!  All we want is something that can be interacted with in a certain way - for instance, being "rotatable" - and for anything done to that "thing" to modify the original object's state.

So, the _true_ delegation pattern isn't this:

```kotlin
class Machine : Wrenchable, Rotatable {
	val wrenchable = WrenchableStateHolder()
	val rotatable = RotatableStateHolder()
	
	override fun useWrench() = wrenchable.useWrench()
	override fun rotate() = rotatable.rotate()
}
```

It's _THIS_!

```kotlin
class Machine {
	inner class WrenchableView : Wrenchable {
		override fun useWrench() {
			this@Machine.fixGears()
			this@Machine.tightenBolts()
		}
	}
	
	inner class RotatableView : Rotatable {
		override fun rotate() {
			this@Machine.facing = Direction.clockwise(this@Machine.facing)
		}
	}
	
	fun asWrenchable() {
		return WrenchableView()
	}
	
	fun asRotatable() {
		return RotatableView()
	}
}
```
Obviously since we want to save on object creation, we'd probably just hold onto the views instead of constantly disposing of them, but honestly that might take up more memory than just recreating them each time so what do I know? It totally depends on the circumstance

With this in mind, there's still one major problem with this pattern:

## How do we know it has a Wrenchable inside it?

Obviously, interfaces aren't _just_ views on their own. They also tell other people _that_ they're viewable that way.

One way would be another interface - called a "bridge" - to tell others that they _can_ be viewed as a wrenchable using a given method:

```kotlin
interface ViewableAsWrenchable {
	fun asWrenchable(): Wrenchable
}
```

but that's kinda sucky for syntax, cause now you have to check if something is _either_ a Wrenchable _or_ one of the seven other ways to GET a wrenchable from something, and then do the manual conversion with a specific process unique to that interface.

Well, rather than implementing a billion delegate interfaces, we could just have one function for conversion to its different views, like this:

```kotlin
class Foo : Viewable {
	override fun getView(type: Type) {
		return when (type) {
			Wrenchable -> ourWrenchable
			Rotatable -> ourRotatable
			else -> throw ClassCastException()
		}
	}
}
```
This is actually used in the Capabilities system in Minecraft Forge to great effect, as well as _Rust's "from" and "to" system_.

Now, of course, doing this dynamically doesn't really make sense when you could just be doing it at compiletime.  That's where this compiler plugin comes in!

## What does this do?

```kotlin
interface btpos.tools.Wrenchable {
	fun onWrench()
}

class Foo : Wrenchable through WrenchableView() 

class Bar : Wrenchable through wrn {
	val wrn = WrenchableView()
}

// GENERATES:
interface bridges.btpos.tools.WrenchableProvider { // shared package to not make duplicates
	fun asWrenchable(): Wrenchable
}

// COMPILES TO:
class Foo : WrenchableProvider {
	private val `$$delegate$0$Wrenchable`: Wrenchable = WrenchableView() // this is how Kotlin natively handles "by" declarations
	
	override fun asWrenchable(): Wrenchable {
		return `$$delegate$0$Wrenchable`
	}
}

class Bar : WrenchableProvider {
	val wrn = WrenchableView()
	
	override fun asWrenchable(): Wrenchable {
		return wrn
	}
}
```

## What does this allow?

```kotlin
Foo().to<Wrenchable>().onWrench() // brackets not required if unambiguous
Foo().onWrench() // if `onWrench()` isn't ambiguous, otherwise the cast must be explicitly done
(Foo() as Wrenchable).onWrench()

// both directly compile to:
foo.asWrenchable().useWrench()
```

```kotlin
fun acceptsWrenchable(x: Wrenchable) { 
	// ...
}

acceptsWrenchable(Foo())
// compiles to:
acceptsWrenchable(Foo().asWrenchable())
```

```kotlin
Foo() is btpos.tools.Wrenchable
// compiles to:
val foo = Foo()
(foo is btpos.tools.Wrenchable || foo is bridges.btpos.tools.WrenchableProvider)
```

Since the only way to have other APIs get a Wrenchable instance is through method calls...... i think (spring might make things difficult), this _shouldn't_ be a problem as long as we do the conversions right?

We're essentially enabling polymorphism with _component views_ instead of views of the _original object_.

Problem: what if we want to allow people to declare that their subclasses will have wrenchable values? How do they reference that generated interface?
```kotlin
interface IComposite : btpos.tools.Wrenchable from wrenchable, btpos.tools.Rotatable from rotatable {
	val wrenchable: Wrenchable
	val rotatable: Rotatable
}
// compiles to:
interface IComposite : bridges.btpos.tools.Wrenchable, bridges.btpos.tools.Rotatable {
	val wrenchable: Wrenchable
	val rotatable: Rotatable
	
	override fun asWrenchable(): Wrenchable = wrenchable
	override fun asRotatable(): Rotatable = rotatable
}
```
Yeah, this works.



Question: Should the keyword be `through` or `from`? 
I think it's better to be `through` since calls are going THROUGH the delegate FIRST and THEN to the source object