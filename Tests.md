# Tests - can we have too many?

I find myself having this discussion with people a lot. We all love tests and we all know that tests are super useful. But have we ever thought about the possibility that writing tests may mislead us into thinking that our code is good? Perhaps we should aim to have _less_ tests. Maybe tests are a sign that our code could be improved?

Note that a lot of the things I'm going to discuss are worth an entire blog post each. I'm going to scratch the surface and hopefully it will motivate you to read up on this further.

Let's explore this topic further with a couple of examples. We'll start off with a simple one.

## Parametricity

### Example 1:

Given the following function:

```
def doNothing(x: Int): Int = x
```

This function takes an `Int` and returns it without doing anything to it. It is the simplest function you can have. Do we need to test it? Surely! What if someone comes along and changes it to `x + 1`? You need a test to capture that possibility.

```
assert(doNothing(1) == 1)
```

You might use a property based testing library to check _for all_ `Int`s:

```
forAll(x => assert(doNothing(x) == x))
```

What if you were challenged to not write a test for this but still ensure that the function cannot be implemented incorrectly?

Parametric types is precisely what you can use in this case. Consider this implementation:

```
def doNothing[A](x: A): A = x
```

All we have done is introduce a parametric type `A`. You can read the function's type signature as _"for all `A`s, given an `A`, the function returns an `A`"_. `A` is parametric, meaning: it _is not_ a `Int` or a `String` or a `Double` but it _can be_ any of those. You cannot add `1` to `A` because it's not an `Int`, you cannot concatenate `"abc"` to `A` because it's not a `String`. Returning `x` is the _only_ reasonable way to get the function to compile!

These are not valid implementations:

```
def doNothing[A](x: A): A = 500                            // A is not an Int so you can't just return 500
def doNothing[A](x: A): A = x + 1                          // x is not an Int/Double
def doNothing[A](x: A): A = x ++ "abc"                     // x is not a String
def doNothing[A](x: A): A = x ++ List("abc", "def", "ghi") // x is not a List[String]
```

This small change to the type signature means we have written a `doNothing` function that works on _all types_! It will pass all these assertions:

```
assert(doNothing(1) == 1)
assert(doNothing("abc") == "abc")
assert(doNothing(List(1, 2, 3)) == List(1, 2, 3))
```

In addition to not needing to test this, we're now keeping it DRY!

### Example 2:

You can have parametric types at a higher level too, consider this:

```
def transform(list: List[Int], func: Int => String): List[String] = ???
```

What can this function do? Many things. These are all valid implementations:

```
= list.map(elem => func(elem))
= List("abc")
= List()
= List("xyz", "def")
```

The first implementation is the only one that we consider correct. We would probably need the two tests below:

```
assert(transform(List(1, 2, 3), num => s"$num") == List("1", "2", "3"))
assert(transform(List(), num => s"$num") == List())
```

We can write this function as this:

```
def transform[F: Functor](container: F[Int], func: Int => String): F[String] =
  container.map(elem => func(elem))
```

We have parameterised the type of `container`. Instead of saying it's a `List`, we say it's a `Functor`, which is a container that has the function `.map` and nothing else! You can't just spawn an `F[String]` out of nowhere so what I wrote above is literally the only implementation that satisfies the type signature!

The beauty of this? It works for not just `List`, but anything that has a `Functor` instance. This includes `Option`, `Vector`, etc. 

## Stronger data types

Powerful languages like Haskell and Scala would have more powerful data types that can help you make illegal state irrepresentable. Let's look at some examples.

### Example 1: NonEmptyList

Consider this function:

```
def mean(numbers: List[Int]): Double = numbers.sum / numbers.length.toDouble
```

If you're obsessed with testing edge cases, you'd pick up the fact that `numbers.length` can be `0`, which would result in some kind of runtime error. This means you'd have to handle that case differently, potentially returning an `Option[Double]` as such:

```
def mean(numbers: List[Int]): Option[Double] = numbers match {
  case Nil => None                    // if numbers is an empty List aka. Nil
  case ns => Some(ns.sum / ns.length.toDouble) // otherwise
}
```

...and you'd need a test for this:

```
assert(mean(List()) == None)
```

Seems like something that can be avoided using stronger data types.

Luckily, Scala has [`NonEmptyList`](https://typelevel.org/cats/datatypes/nel.html)! Again, this isn't restricted to Scala.

```
def mean(numbers: NonEmptyList[Int]): Double = numbers.sum / numbers.length.toDouble
```

It is impossible to call this function with an empty List. We have made this illegal state impossible to reach!

### Example 2: Algebraic data types and safe constructors

Consider this function:

```
def showTrafficLight(trafficLight: String) = trafficLight match {
  case "red"    => "I am a red light"
  case "green"  => "I am a green light"
  case "yellow" => "I am a yellow light"
  case _        => "Oops I am an invalid light"
}
```

Again, we'd have to write a test for the invalid case:

```
assert(showTrafficLight("blah") == "Oops I am an invalid light")
```

Why oh why is it possible to get into this state in the first place? Imagine if we had other functions that work on traffic lights, we'd need to handle invalid traffic lights in every single one of them to ensure they can be re-used safely.

We can improve this by defining our own algebraic data type. Here we create a new type called `TrafficLight` and that it can be one of `Red`, `Green` and `Yellow`.

```
sealed trait TrafficLight
case object Red extends TrafficLight
case object Green extends TrafficLight
case object Yellow extends TrafficLight
```

Since we might be getting a traffic light as a `String` from a file (for instance), we would need to write a safe constructor to convert a `String` into our `TrafficLight`.

```
def mkTrafficLight(str: String): Option[TrafficLight] = str match {
  case "red"    => Some(Red)
  case "green"  => Some(Green)
  case "yellow" => Some(Yellow)
  case _        => None
}
```

This function needs to be tested:

```
assert(mkTrafficLight("red")    == Some(Red))
assert(mkTrafficLight("green")  == Some(Green))
assert(mkTrafficLight("yellow") == Some(Yellow))
assert(mkTrafficLight("blah")   == None)
```

Now, we can rewrite `showTrafficLight`:

```
def showTrafficLight(trafficLight: TrafficLight): String = trafficLight match {
  case Red    => "I am a red light"
  case Green  => "I am a green light"
  case Yellow => "I am a yellow light"
}
```

Notice we don't need to handle the case where `trafficLight` is neither `Red`, `Green` nor `Yellow` because it's impossible for it to get into this state! We handle the invalid case once in the safe constructor earlier on in the program (when parsing from a file, reading from HTTP, etc.) and then the rest of the program can _safely_ assume that it is working with a valid `TrafficLight`.
