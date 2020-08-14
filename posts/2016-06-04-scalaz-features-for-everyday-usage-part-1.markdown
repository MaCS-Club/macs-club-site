---
title: Scalaz для ежедневного использования. Часть 1. Классы типов и расширения Scala.
author: jos.dirksen (перевод BeiZero)
tags: scala, scalaz для ежедневного использования, scalaz, перевод
description: Многие из вас наверно слышали о замечательной JavaScript книге "JavaScript the good parts". В подобном ключе я бы хотел рассказать о некоторых вещах из Scalaz, которые действительно здорово использовать в повседневных проектах без необходимости вникать в то что происходит внутри Scalaz. В первой части мы рассмотрим несколько полезных классов типов. В будущих частях мы рассмотрим такие вещи как монадные трансформеры, свободные монады, Validation и т.д.
---

Многие из вас наверно слышали о замечательной JavaScript книге "JavaScript the good parts". В подобном ключе я бы хотел рассказать о некоторых вещах из Scalaz, которые действительно здорово использовать в повседневных проектах без необходимости вникать в то что происходит внутри Scalaz. В первой части мы рассмотрим несколько полезных классов типов. В будущих частях мы рассмотрим такие вещи как монадные трансформеры, свободные монады, Validation и т.д.

<!--more-->

В наших примерах мы будем использовать Scala REPL. Для этого запустим `scala`, добавим библиотеку scalaz и импортируем нужные нам пакеты:

```scala
scala> :require /Users/jos/.ivy2/cache/org.scalaz/scalaz-core_2.11/bundles/scalaz-core_2.11-7.2.1.jar
Added '/Users/jos/.ivy2/cache/org.scalaz/scalaz-core_2.11/bundles/scalaz-core_2.11-7.2.1.jar' to classpath.

scala> import scalaz._
import scalaz._

scala> import Scalaz._
import Scalaz._
```

В данной статье мы рассмотрим следующие классы типов из библиотеки Scalaz:

1. Equals: для типобезопасной операции сравнения на равенство.
2. Order: для немного более типобезопасного отношения порядка.
3. Enum: для создания богатых типов перечеслений.

Кроме того, мы также рассмотрим несколько простых расширений, которые добавляет Scalaz, для некоторых типов из стандартной библиотеки. Мы не будем рассматривать всё что добавляет Scalaz и остановимся на паре расширений для Option и Boolean.

### Полезные классы типов

С помощью классов типов можно легко добавить функциональность к существующим классам. Scalaz уже содержит в себе несколько полезных классов типов, которые вы можете сразу же использовать.

#### Типобезопасный оператор сравнения

В Scalaz есть типобезопасный оператор сравнения на равенство, который выдаёт ошибку компиляции если мы пытаемся сравнить объекты разных типов. Таким образом, в то время как `==` и `!=` из стандартной библиотеки позволят вам сравнивать, например, объекты классов String и Int, использование операторов `===` и `=/=` из Scalaz приведет к ошибке компиляции:

```scala
scala> 1 == 1
res6: Boolean = true
scala> 1 === 1
res7: Boolean = true
scala> 1 == "1"
res8: Boolean = false
scala> 1 === "1"
<console>:14: error: type mismatch;
 found   : String("1")
 required: Int
              1 === "1"
```

Scalaz предоставляет следующий набор операторов поведение которых легко понять из реализации:

```scala
final def ===(other: F): Boolean = F.equal(self, other)
final def /==(other: F): Boolean = !F.equal(self, other)
final def =/=(other: F): Boolean = /==(other)
final def ≟(other: F): Boolean = F.equal(self, other)
final def ≠(other: F): Boolean = !F.equal(self, other)
```

#### Класс типов Order

Этот очень простой класс типов позволяет более типобезопасно пользоваться отношением порядка. Так же, как и с операторами из Equals теперь мы можем поймать сравнение двух объектов разных типов во время компиляции:

```scala
scala> 1 < 4d
res25: Boolean = true

scala> 1 lte 4d
<console>:14: error: type mismatch;
 found   : Double(4.0)
 required: Int
              1 lte 4d

scala> 1 ?|? 1
res31: scalaz.Ordering = EQ

scala> 1 ?|? 2
res32: scalaz.Ordering = LT

scala> 1 ?|? 2d
<console>:14: error: type mismatch;
 found   : Double(2.0)
 required: Int
              1 ?|? 2d
```

Scalaz предоставляет для этого следующие операторы:

```scala
final def <(other: F): Boolean = F.lessThan(self, other)
final def <=(other: F): Boolean = F.lessThanOrEqual(self, other)
final def >(other: F): Boolean = F.greaterThan(self, other)
final def >=(other: F): Boolean = F.greaterThanOrEqual(self, other)
final def max(other: F): F = F.max(self, other)
final def min(other: F): F = F.min(self, other)
final def cmp(other: F): Ordering = F.order(self, other)
final def ?|?(other: F): Ordering = F.order(self, other)
final def lte(other: F): Boolean = F.lessThanOrEqual(self, other)
final def gte(other: F): Boolean = F.greaterThanOrEqual(self, other)
final def lt(other: F): Boolean = F.lessThan(self, other)
final def gt(other: F): Boolean = F.greaterThan(self, other)
```

#### Класс типов Enum

С Enum из Scalaz очень легко создавать типы перечисления, которые имеют больше функциональных возможностей, чем те, что находятся в стандартных библиотеках Scala и Java. Scalaz предоставляет для этого следующие функции:

```scala
final def succ: F = F succ self
final def -+-(n: Int): F = F.succn(n, self)
final def succx: Option[F] = F.succx.apply(self)
final def pred: F = F pred self
final def ---(n: Int): F = F.predn(n, self)
final def predx: Option[F] = F.predx.apply(self)
final def from: EphemeralStream[F] = F.from(self)
final def fromStep(step: Int): EphemeralStream[F] = F.fromStep(step, self)
final def |=>(to: F): EphemeralStream[F] = F.fromTo(self, to)
final def |->(to: F): List[F] = F.fromToL(self, to)
final def |==>(step: Int, to: F): EphemeralStream[F] = F.fromStepTo(step, self, to)
final def |-->(step: Int, to: F): List[F] = F.fromStepToL(step, self, to)
```
Очень хороший пример можно найти на [StackOverflow](http://stackoverflow.com/questions/28589022/enumeration-concept-in-scala-which-option-to-take), который всё же требует нескольких изменений для того чтобы получить все вкусности Scala. Следующий код показывает как использовать данное перечесление:

```scala
scala> import scalaz.Ordering._
import scalaz.Ordering._

scala> :paste
// Entering paste mode (ctrl-D to finish)

  case class Coloring(val toInt: Int, val name: String)

  object Coloring extends ColoringInstances {

    val RED = Coloring(1, "RED")
    val BLUE = Coloring(1, "BLUE")
    val GREEN = Coloring(1, "GREEN")
  }

  sealed abstract class ColoringInstances {

    import Coloring._

    implicit val coloringInstance: Enum[Coloring] with Show[Coloring] = new Enum[Coloring] with Show[Coloring] {

      def order(a1: Coloring, a2: Coloring): Ordering = (a1, a2) match {
        case (RED, RED) => EQ
        case (RED, BLUE | GREEN) => LT
        case (BLUE, BLUE) => EQ
        case (BLUE, GREEN) => LT
        case (BLUE, RED) => GT
        case (GREEN, RED) => GT
        case (GREEN, BLUE) => GT
        case (GREEN, GREEN) => EQ
      }

      def append(c1: Coloring, c2: => Coloring): Coloring = c1 match {
        case Coloring.RED => c2
        case o => o
      }

      override def shows(c: Coloring) = c.name

      def zero: Coloring = Coloring.RED

      def succ(c: Coloring) = c match {
        case Coloring.RED => Coloring.BLUE
        case Coloring.BLUE => Coloring.GREEN
        case Coloring.GREEN => Coloring.RED
      }

      def pred(c: Coloring) = c match {
        case Coloring.GREEN => Coloring.BLUE
        case Coloring.BLUE => Coloring.RED
        case Coloring.RED => Coloring.GREEN
      }

      override def max = Some(GREEN)

      override def min = Some(RED)

    }
  }

// Exiting paste mode, now interpreting.

defined class Coloring
defined object Coloring
defined class ColoringInstances
```

Теперь мы можем использовать все функции определённые в Scalaz Enum:

```scala
scala> import Coloring._
import Coloring._

scala> RED
res0: Coloring = Coloring(1,RED)

scala> GREEN
res1: Coloring = Coloring(1,GREEN)

scala> RED |-> GREEN
res2: List[Coloring] = List(Coloring(1,RED), Coloring(1,BLUE), Coloring(1,GREEN))

scala> RED succ
warning: there was one feature warning; re-run with -feature for details
res3: Coloring = Coloring(1,BLUE)

scala> RED -+- 1
res4: Coloring = Coloring(1,BLUE)

scala> RED -+- 2
res5: Coloring = Coloring(1,GREEN)
```

Правда впечатляет? Это действительно очень хороший способ создания гибких и богатых перечислений.

### Расширения стандартных классов

Как мы уже говорили в начале статьи, мы рассмотрим как Scalaz делает стандартную библиотеку более богатой и добавляет новую функциональность к стандартным классам.

####Веселимся с Option

С классом типов Optional Scalaz делает работу с Option проще. Например, он предоставляет функции для более простого конструирования:

```scala
scala> Some(10)
res11: Some[Int] = Some(10)

scala> None
res12: None.type = None

scala> some(10)
res13: Option[Int] = Some(10)

scala> none[Int]
res14: Option[Int] = None
```

Мы видим что результирующим типом является Option[T] вместо Some или None. Вам может быть не понятно где это может быть полезно, но давайте рассмотрим следующую ситуацию: скажем, у нас есть список Option к которому мы хотим применить свёртку:

```scala
scala> val l = List(Some(10), Some(20), None, Some(30))
l: List[Option[Int]] = List(Some(10), Some(20), None, Some(30))

scala> l.foldLeft(None) { (el, z) => el.orElse(z)  }
<console>:22: error: type mismatch;
 found   : Option[Int]
 required: None.type
              l.foldLeft(None) { (el, z) => el.orElse(z)  }
```

Данный код упадёт с ошибкой, потому что результат свёртки должен быть None.type, а не Option. Когда мы используем функции из Scalaz это работает так как мы и ожидали:

```scala
scala> l.foldLeft(none[Int]) { (el, z) => el.orElse(z)  }
res19: Option[Int] = Some(10)
```

И конечно же Scalaz добавляет новых операторов:

```scala
// Альтернатива getOrElse
scala> Some(10) | 20
res29: Int = 10

scala> none | 10
res30: Int = 10

// Тернарный оператор
scala> Some(10) ? 5 | 4
res31: Int = 5

// ~ : Возвращает элемент хранимый в Option, если он там есть, а иначе "ноль" типа A
scala> some(List())
res32: Option[List[Nothing]] = Some(List())

scala> ~res32
res33: List[Nothing] = List()

scala> some(List(10))
res34: Option[List[Int]] = Some(List(10))

scala> ~res34
res35: List[Int] = List(10)
```

Ничего слишком сложного, просто несколько вспомогательных функций. Существует множество двугих интересных вещей вокруг Option в Scalaz, но это уже выходит за рамки данной статьи.

#### Более функциональный Boolean

Scalaz так же добавляет больше функциональности к типу Boolean:

```scala
# Тернарный опертор возвращается!
scala> true ? "This is true" | "This is false"
res45: String = This is true

scala> false ? "This is true" | "This is false"
res46: String = This is false

# Возвращает передаваемый аргумент в случае true, иначе возвращает "ноль" типа A
scala> false ?? List(120,20321)
res55: List[Int] = List()

scala> true ?? List(120,20321)
res56: List[Int] = List(120, 20321)
```

И целый список операторов для бинарной арифметики:

```scala
// Conjunction. (AND)
final def ∧(q: => Boolean) = b.conjunction(self, q)
// Conjunction. (AND)
final def /\(q: => Boolean) = ∧(q)
// Disjunction. (OR)
final def ∨(q: => Boolean): Boolean = b.disjunction(self, q)
// Disjunction. (OR)
final def \/(q: => Boolean): Boolean = ∨(q)
// Negation of Disjunction. (NOR)
final def !||(q: => Boolean) = b.nor(self, q)
// Negation of Conjunction. (NAND)
final def !&&(q: => Boolean) = b.nand(self, q)
// Conditional.
final def -->(q: => Boolean) = b.conditional(self, q)
// Inverse Conditional.
final def <--(q: => Boolean) = b.inverseConditional(self, q)
// Bi-Conditional.
final def <-->(q: => Boolean) = b.conditional(self, q) && b.inverseConditional(self, q)
// Inverse Conditional.
final def ⇐(q: => Boolean) = b.inverseConditional(self, q)
// Negation of Conditional.
final def ⇏(q: => Boolean) = b.negConditional(self, q)
// Negation of Conditional.
final def -/>(q: => Boolean) = b.negConditional(self, q)
// Negation of Inverse Conditional.
final def ⇍(q: => Boolean) = b.negInverseConditional(self, q)
// Negation of Inverse Conditional.
final def <\-(q: => Boolean) = b.negInverseConditional(self, q)
```

Например:

```scala
scala> true /\ true
res57: Boolean = true

scala> true /\ false
res58: Boolean = false

scala> true !&& false
res59: Boolean = true
```

#### Больше дополнительных функций

В данной статье мы рассмотрели всего несколько дополнительных функций из Scalaz. Если вас заинтересовала эта тема вам стоит заглянуть в код следующих классов:

- scalaz.syntax.std.BooleanOps
- scalaz.syntax.std.ListOps
- scalaz.syntax.std.MapOps
- scalaz.syntax.std.OptionOps
- scalaz.syntax.std.StringOps

Ещё немного примеров:

#### Балуемся с List

```scala
# взять хвост как Option
scala> List(10,20,30)
res60: List[Int] = List(10, 20, 30)

scala> res60.tailOption
res61: Option[List[Int]] = Some(List(20, 30))

scala> List()
res64: List[Nothing] = List()

scala> res64.tailOption
res65: Option[List[Nothing]] = None

# "усыпать" список дополнительными элементами
scala> List(10,20,30)
res66: List[Int] = List(10, 20, 30)

scala> res66.intersperse(1)
res68: List[Int] = List(10, 1, 20, 1, 30)

# всевозможные перестановки списка
scala> List('a','b','c','d').powerset
res71: List[List[Char]] = List(List(a, b, c, d), List(a, b, c), List(a, b, d), List(a, b), List(a, c, d), List(a, c), List(a, d), List(a), List(b, c, d), List(b, c), List(b, d), List(b), List(c, d), List(c), List(d), List())
```

#### Забавляемся с Map

```scala
# безопасно изменить запись
res77: scala.collection.immutable.Map[Char,Int] = Map(a -> 10, b -> 20)
scala> res77.alter('a')(f => f |+| some(5))
res78: Map[Char,Int] = Map(a -> 15, b -> 20)

# пересечь два Map'а и определить какое значение оставить для каждого из ключей
scala> val m1 =  Map('a' -> 100, 'b' -> 200, 'c' -> 300)
m1: scala.collection.immutable.Map[Char,Int] = Map(a -> 100, b -> 200, c -> 300)

scala> val m2 = Map('b' -> 2000, 'c' -> 3000, 'd' -> 4000)
m2: scala.collection.immutable.Map[Char,Int] = Map(b -> 2000, c -> 3000, d -> 4000)

scala> m1.intersectWith(m2)((m1v,m2v) => m2v)
res23: Map[Char,Int] = Map(b -> 2000, c -> 3000)

scala> m1.intersectWith(m2)((m1v,m2v) => m1v)
res24: Map[Char,Int] = Map(b -> 200, c -> 300)
```

#### Развлекаемся с String

```scala
# сделать строку множественного числа(наивный способ)
scala> "Typeclass".plural(1)
res26: String = Typeclass

scala> "Typeclass".plural(2)
res27: String = Typeclasss

scala> "Day".plural(2)
res28: String = Days

scala> "Weekly".plural(2)
res29: String = Weeklies

# безопасный парсинг значений типов Boolean, Byte, Short, Long, Float, Double и Int
scala> "10".parseDouble
res30: scalaz.Validation[NumberFormatException,Double] = Success(10.0)

scala> "ten".parseDouble
res31: scalaz.Validation[NumberFormatException,Double] = Failure(java.lang.NumberFormatException: For input string: "ten")
```

### Заключение

Я надеюсь, что вам понравилось это краткое введение в Scalaz. И как вы уже видели эти простые функции уже позволяют делать множество интересных вещей, без необходимости разбираться с внутренними сложностями Scalaz. Паттерн рассматриваемый здесь называется TypeClass Pattern и используется для расширения стандартной функциональности типов в Scala.

В следующей статье мы познакомимся с более сложными вещами и поработаем с монадными трансформерами.


[Оригинальная статья.](http://www.smartjava.org/content/scalaz-features-everyday-usage-part-1-typeclasses-and-scala-extensions)
