# Scala implicits

---

## Introduction

Scala implicits are scary:

- they are unique to this language
- they are not super intuitive

Goal of this presentation:

- usual patterns solved by implicits
- how to implement them in Scala with implicits

---

## Beware

> Explicit is better than implicit.

(Zen of Python, Dalai Lama)

---

## Extension functions

Class provided by a library:

```scala
case class Point(x: Double, y: Double) {
  def norm(): Double =
    Math.sqrt(x * x + y * y)
}
```

We want a `l1Norm` method that would do this:

```scala
case class Point(x: Double, y: Double) {
  ...

  def l1Norm(): Double =
    Math.abs(x) + Math.abs(y)
}
```

---

## Extension functions

But we can't because we can't change the `Point` class.

- we could use a function instead
- but sometimes we really really want a method

---

## Extenstion functions

Implicits to the rescue !

```scala
implicit class PointOps(point: Point) {
  def l1Norm(): Double =
    Math.abs(point.x) + Math.abs(point.y)
}
```

We can then use:

```scala
val n = Point(3, 4).l1Norm()
```
---

## Extenstion functions

WTF happened here ?

==> when we call a method that does not exist, Scala will look for an implicit class that has this method

Where does it look ?

- in the lexical scope
- in the componion object of the involved class
- in [a lot of other places](https://docs.scala-lang.org/tutorials/FAQ/finding-implicits.html) :(

---

## Extenstion functions

Lexical scope

```scala
object Toto {
  def printNorm() = {
    implicit class PointOps(point: Point) {
      def l1Norm(): Double =
        Math.abs(point.x) + Math.abs(point.y)
    }
    println(Point(3, 4).l1Norm()) // OK
  }

  def printNorm2() =
    println(Point(3, 4).l1Norm()) // Not OK
}
```

---

## Extenstion functions

Lexical scope

```scala
object Tata {
  implicit class PointOps(point: Point) {
    def l1Norm(): Double =
      Math.abs(point.x) + Math.abs(point.y)
  }

  def printNorm3() =
    println(point(3, 4).l1norm()) // ok

}

object Titi {
  import Tata.PointOps
  def printNorm4() =
    println(Point(3, 4).l1Norm()) // ok
}
```

---

## Extenstion functions

On the companion object

```scala
case class Point(x: Double, y: Double) {}


object Point {
  implicit class PointOps(point: Point) {
    def l1Norm(): Double =
      Math.abs(point.x) + Math.abs(point.y)
  }
}

object Toto {
    def printNorm5() =
      println(point(3, 4).l1norm()) // ok
}
```

(not super useful for this use case)

---

## Implicit conversions

```scala
case class Fractional(numerator: Double, denominator: Double) {
   ...
}

def doSomething(x: Double) = { ... }

doSomething(Fractional(3, 4)) // Error !
```

We can solve this with implicit conversions


...but it's a terrible idea :)

---

## Implicit conversions

```scala
implicit def toDouble(f: Fractional): Double =
  f.numerator / f.denominator

doSomething(Fractional(3, 4)) // Works !
```

**I repeat**: do NOT do this

---


## Implicit conversions

Remember implicit classes:

```scala
implicit class PointOps(point: Point) {
  def l1Norm(): Double =
    Math.abs(point.x) + Math.abs(point.y)
}
```

The horrible truth:

```scala
class PointOps(point: Point) {
  def l1Norm(): Double =
    Math.abs(point.x) + Math.abs(point.y)
}
implicit def convertToPointOps(point: Point) : PointOps = PointOps(point)
```

---

# Stop here for now ?


---

## Abstraction over "static" methods

```scala
case class Point(x: Double, y: Double) {
  ...

  def toJson(): String = ???
}


def printAsJson(point: Point) = {
    println(point.toJson())
}
```

---

## Abstraction over "static" methods

`printAsJson` does not need to be restricted to `Point`:

```scala
trait ToJson {
  def toJson(): String
}

def printAsJson[T <: ToJson](point: T) =
  println(point.toJson())

case class Point(x: Double, y: Double) extends ToJson {
  def toJson(): String = ???
}
```

---

## Abstraction over "static" methods

What about a `fromJson` method ?


```python
@dataclass
class Point:
    x: float
    y: float

    @staticmethod
    def from_json(value: str) -> Point:
        ...
```

---

## Abstraction over "static" methods

- no static methods in scala
- we use companion objects

```scala
case class Point(x: Double, y: Double) {}

object Point {
    def fromJson(value: String) : Point = ???
}
```

---

## Abstraction over "static" methods

How to abstract over this ?

```scala
def fromJsonTuple(values: (String, String)) : (Point, Point) = {
    (Point.fromJson(values._1), Point.fromJson(values._2))
}
```

---

## Abstraction over "static" methods

We would need something like this:

```scala
trait FromJson {
  static fromJson(value: String) : <Something>
}

def fromJsonTuple[T <: FromJson](values: (String, String)) : (T, T) = {
    (T.fromJson(values._1), T.fromJson(values._2))
}
```

But it does not exist :(

---

## Abstraction over "static" methods

A first solution:

```scala
trait FromJson[T] {
  def fromJson(value: String): T
}

def fromJsonTupleBad[T](values: (String, String), fromJson: FromJson[T]): (T, T) =
  (fromJson.fromJson(values._1), fromJson.fromJson(values._2))


class PointFromJson extends FromJson[Point] {
  def fromJson(value: String): Point = ???
}
```

---

## Abstraction over "static" methods

Then you can do:


```scala
fromJsonTupleBad(myStringTuple, new PointFromJson())
```

But you have to explicitly pass the `PointFromJson` instance everywhere.

---

## Abstraction over "static" methods

```scala
def fromJsonTuple[T](v: (String, String))(implicit f: FromJson[T]): (T, T) =
  (f.fromJson(v._1), f.fromJson(v._2))
```

And you can do:

```scala
implicit val fromJson: FromJson[Point] = new PointFromJson()
fromJsonTuple[Point](myStringTuple)
```


---

## Abstraction over "static" methods

Bonus: you can swap implementations:

```scala
// Reads '{"x": 1, "y": 3.4}'
class PointFromJsonObject extends FromJson[Point] {
  def fromJson(value: String): Point = ???
}

// Reads '[1, 3.4]'
class PointFromJsonArray extends FromJson[Point] {
  def fromJson(value: String): Point = ???
}

implicit val fromJson: FromJson[Point] = new PointFromJsonObject()
fromJsonTuple(myStringTuple)
fromJsonTuple(myStringTuple)(new PointFromJsonArray())
```

---

## Abstraction over "static" methods

In this case, defining a default implementation in the companion object makes a lot of sense

```scala
object Point {
    implicit val fromJson = new PointFromJsonObject()
}
```

---

## Abstraction over "static" methods

The combination of:

- the `FromJson[T]` trait
- the implicit instance

Is sometimes called a typeclass









