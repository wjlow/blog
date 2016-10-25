How a Monad can be useful
=================

Disclaimer: This is not a comprehensive Monad tutorial. I aim to demonstrate why a concrete example of a Monad is useful in programming.

Consider this problem. 
You have a function that converts a `String` to a `Double`:
```
def stringToDouble(str: String): Double =
  str.toDouble
```

What happens if `stringToDouble` is called with `"800"`? You get a `java.lang.NumberFormatException`. Exceptions are evil! How can a function that returns `Double` do anything but return `Double`? Let's fix up our function to no longer lie to us.

Error handling example
=================

What if we changed the implementation of `stringToDouble` just slightly? 
```
def stringToDouble(str: String): List[Double] = 
  try {
    List(str.toDouble)
  } catch {
    case e: Exception => List()
  }
```
Here, we return either an empty `List` to signify the error case, or a singleton `List` containing the converted `Double`. 

If we had to convert a `List` of `String`s to a `List` of `Double`s, how would it look like?
```
def stringsToDoubles(strs: List[String]): List[List[Double]] =
  strs map stringToDouble
```

e.g. `stringsToDoubles(List("1.0", "2.0", "Bad string")) == List(List(1.0), List(2.0), List())`

This is not good enough! Ideally we want to go from `List[String] => List[Double]`, but now we have `List` of `List`s. 

Flatten
=================
An important property of Monad is its ability to `flatten` from `Monad[Monad[A]]` to `Monad[A]`. How is this useful? 
In our previous example, swap `Monad` for `List` and `A` for `Double` and we will see that we have `List[List[Double]]`.

Luckily, `List` is a Monad ("has a Monad instance")! We can rewrite `stringsToDoubles` now:
```
def stringsToDoubles2(strs: List[String]): List[Double] =
  (strs map stringToDouble).flatten
```

`stringsToDoubles2(List("1.0", "2.0", "Bad string"))` now yields `List(1.0, 2.0)`. Not surprisingly, this is something that we end up having to do fairly often. This is why Monads have a helpful function called `flatMap` (or sometimes `bind`), which is essentially `map` then `flatten`. 

Let's rewrite `stringsToDoubles` using `flatMap`:
```
def stringsToDoubles3(strs: List[String]): List[Double] =
  strs flatMap stringToDouble
```

Option
=================
Surely using `List` to model the possible existence of a single value is not optimal, because a `List` suggests it can contain more than one value. I used `List` as an example because it is the most common Monad that people know. In most functional programming languages, there exists a Monad that is used for this purpose. In Scala, it is called the `Option` Monad.

In a nutshell, a value that is of type `Option[A]` can be one of two possible values: `Some[A]` or `None`. The former represents the existence of a value and the latter represents non-existence.

`stringToDouble` can be written using `Option` very easily:
```
def stringToDouble2(str: String): Option[Double] = 
  try {
    Some(str.toDouble)
  } catch {
    case e: Exception => None
  }
```

Side note: Idiomatic Scala would use the `Try` Monad instead of a `try/catch` block. `Try` is used to wrap expressions that may throw an exception and can be one of two values: `Success` or `Failure`. But I won't go through this here.
```
def stringToDouble3(str: String): Option[Double] = 
  Try(str.toDouble).toOption
```
