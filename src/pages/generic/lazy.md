## Deriving instances for recursive types

```tut:book:invisible
// ----------------------------------------------
// Forward definitions

trait CsvEncoder[A] {
  def encode(value: A): List[String]
}

def writeCsv[A](values: List[A])(implicit encoder: CsvEncoder[A]): String =
  values.map(encoder.encode).map(_.mkString(",")).mkString("\n")

def createEncoder[A](func: A => List[String]): CsvEncoder[A] =
  new CsvEncoder[A] {
    def encode(value: A): List[String] =
      func(value)
  }

implicit val stringEncoder: CsvEncoder[String] =
  createEncoder(str => List(str))

implicit val intEncoder: CsvEncoder[Int] =
  createEncoder(num => List(num.toString))

implicit val booleanEncoder: CsvEncoder[Boolean] =
  createEncoder(bool => List(if(bool) "cone" else "glass"))

import shapeless.{HList, HNil, ::}

implicit val hnilEncoder: CsvEncoder[HNil] =
  createEncoder(hnil => Nil)

implicit def hlistEncoder[H, T <: HList](
  implicit
  hEncoder: CsvEncoder[H],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :: T] = createEncoder {
  case h :: t =>
    hEncoder.encode(h) ++ tEncoder.encode(t)
}

import shapeless.Generic

implicit def genericEncoder[A, R](
  implicit
  gen: Generic.Aux[A, R],
  rEncoder: CsvEncoder[R]
): CsvEncoder[A] =
  createEncoder(value => rEncoder.encode(gen.to(value)))

// ----------------------------------------------
```

Let's try something more ambitious---a binary tree:

```tut:book:silent
sealed trait Tree[A]
final case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
final case class Leaf[A](value: A) extends Tree[A]
```

Theoretically we should already have all of the definitions in place
to summon a CSV writer for this definition.
However, calls to `writeCsv` fail to compile:

```tut:book:fail
implicitly[CsvEncoder[Tree[Int]]]
````

The problem here is that the compiler is preventing itself
going into an infinite loop searching for implicits.
If it sees the same type constructor twice in any branch of search,
it assumes implicit resolution is diverging and gives up.
`Branch` is recursive so
the rule for `CsvEncoder[Branch]` is triggering this behaviour.

In fact, the situation is worse than this.
If the compiler sees the same type constructor twice
and the complexity of the type parameters is *increasing*,
it also gives up.
This is a problem for shapeless
because types like `::[H, T]` and `:+:[H, T]`
come up in different generic representations
and cause the compiler to give up prematurely.

### *Lazy*

Fortunately, shapeless provides a type called `Lazy` as a workaround.
`Lazy` does two things:

 1. it delays resolution of an implicit parameter
    until it is strictly needed,
    permitting the derivation of self-referential implicits.

 2. it guards against some of the aforementioned
    over-defensive heuristics.

As a rule of thumb,
it is always a good idea to wrap the "head" parameter
of any `HList` or `Coproduct` rule
and the "repr" parameter of any `Generic` rule in `Lazy`:

```tut:book:invisible:reset
import shapeless._

trait CsvEncoder[A] {
  def encode(value: A): List[String]
}

def createEncoder[A](func: A => List[String]): CsvEncoder[A] =
  new CsvEncoder[A] {
    def encode(value: A): List[String] =
      func(value)
  }

implicit val intEncoder: CsvEncoder[Int] =
  createEncoder(num => List(num.toString))

implicit val hnilEncoder: CsvEncoder[HNil] =
  createEncoder(hnil => Nil)
```

```tut:book:silent
implicit def hlistEncoder[H, T <: HList](
  implicit
  hEncoder: Lazy[CsvEncoder[H]],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :: T] = createEncoder {
  case h :: t =>
    hEncoder.value.encode(h) ++ tEncoder.encode(t)
}
```

```tut:book:invisible
implicit val cnilEncoder: CsvEncoder[CNil] =
  createEncoder(cnil => throw new Exception("The impossible has happened!"))
```

```tut:book:silent
implicit def coproductEncoder[H, T <: Coproduct](
  implicit
  hEncoder: Lazy[CsvEncoder[H]], // wrapped in Lazy
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :+: T] = createEncoder {
  case Inl(h) => hEncoder.value.encode(h)
  case Inr(t) => tEncoder.encode(t)
}
```

```tut:book:silent
implicit def genericEncoder[A, R](
  implicit
  gen: Generic.Aux[A, R],
  rEncoder: Lazy[CsvEncoder[R]] // wrapped in Lazy
): CsvEncoder[A] = createEncoder { value =>
  rEncoder.value.encode(gen.to(value))
}
```

```tut:book:invisible
sealed trait Tree[A]
final case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
final case class Leaf[A](value: A) extends Tree[A]
```

This prevents the compiler giving up prematurely,
and enables the solution to work on complex/recursive types like `Tree`:

```tut:book
implicitly[CsvEncoder[Tree[Int]]]
```
