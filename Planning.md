# Main Goal
- Allow using `by` delegation with instance properties

# Side Goal: `as`
What is the point of this? 

Well, current delegation is actually fully inherited, with the source object taking on all of the delegated interface's members.

```kotlin
interface Part1 {
	fun partMethod()
}

class Foo : Part1 by Part1.Impl()

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

One way to resolve this is to use the more-_verbose_ (and arguably the _real_ version of the) delegation pattern:

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
Just like upcasting to Rotatable, you are stripping down the unique properties of the object and **viewing it** as a Rotatable.  It's just like getting a Set from a Map: despite how stupid that pattern is, it's an example of this same behavior - exposing the exact same source object in a different way!

This touches on something in higher-level programming known as "Optics": rather than outright transforming one object into a different type just to have a new set of methods to interact with it, we _view it in a different way_.  Just like `Set.add(x)` being turned into `Map.put(x, true)` on the backend, interacting with the perfectly normal set in front of you interacts with the original map!

My mental image has been like, this fuzzy thing of having a guy and a tiny man coming out of his chest that's still connected to the larger guy, and messing with the little man does something to the bigger dude.  My mental image is also very influenced by that one doctor who episode with the shapeshifting robot man piloted by tiny people - it's a whole thing and really abstract, don't read into it - but you get the general idea.

Or like a _remote control_ for a larger machine: you could spin the dials and everything on the main control panel, _or_ you could take the remote and bring it with you. You're taking a "remote" object with you, but the remote doesn't actually have anything in itself to worry about, it's just a _modifiable view_ of the original machine's state!

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
Obviously since we want to save on object creation, we'd probably just hold onto the views instead of constantly disposing of them, but honestly that might take up more memory than just recreating them each time so what do I know?

With this in mind, there's still one major problem with this pattern:

## How do we know it has a Wrenchable inside it?

Obviously, interfaces aren't _just_ views on their own. They also tell other people _that_ they're viewable that way.

One way would be another interface - called a "bridge" - to tell others that they can be viewed as a wrenchable using a given method:

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
@View
interface Wrenchable {
	fun onWrench()
}

// generates:

interface Wrenchable {
	fun onWrench()
	
	interface WrenchableProvider {
		fun asWrenchable(): Wrenchable
	}
}

class Foo : Wrenchable through wrn {
	val wrn = WrenchableView()
}

// allows the use of:
Foo().to<Wrenchable>().onWrench() // brackets not required

// generates:
class Foo : WrenchableProvider {
	val wrn = WrenchableView()
	
	override fun asWrenchable(): Wrenchable {
		return wrn
	}
}


val foo = Foo()
foo.to<Wrenchable>().useWrench()
(foo as Wrenchable).useWrench()
//foo.useWrench()
// directly compiles to:
foo.asWrenchable().useWrench()
```

ok problem tho:
```kotlin
foo is Wrenchable
// compiles to:
(foo is Wrenchable || foo is ViewProvider && foo.`as`<Wrenchable>() != null)
```

...hmmmmmmmm, no
best way to do this is to just define a bridge interface INSIDE the other interface, so there's not only a standardized way of using the interface but also of accessing it as that interface







The current Kotlin delegation pattern is basically just modularity with a bit less boilerplate.

Change it from this dynamic approach:
```kotlin
inline fun <reified T> `as`() : T {
	return when (T) {
		Part1 -> ourPart1
		Part2 -> ourPart2
		else -> throw ClassCastException("${this::class} cannot be cast to $T")
	}
}
```
To a true _part_-based delegation with inner classes (or classes with `this` references) and optics
Be able to view it AS an interface - through a DELEGATE - _instead of directly CASTING_ the original object
Be able to define transformations on an object and directly convert between them implicitly using the standard syntax.