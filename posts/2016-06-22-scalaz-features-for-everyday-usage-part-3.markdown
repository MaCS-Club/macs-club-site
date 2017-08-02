---
title: Scalaz для ежедневного использования. Часть 3. Монада State, монада Writer и линзы.
author: BeiZero
tags: scala, scalaz для ежедневного использования, scalaz, перевод
description: В этой статье мы посмотрим на пару монад и паттернов из Scalaz. Напомню, что мы будем смотреть на них с практической точки зрения и избегать внутренних деталей Scalaz. В данной статье мы рассмотрим монада Writer(ведёт своего рода логи во время выполнения набора операций), монада State(предоставляет простой способ пробрасывания состояния между функциями), линзы(предоставляет простой доступ к глубоко вложенным атрибутам и делает создание копий иерархии case class'ов более удобным).
---

В этой статье мы посмотрим на пару монад и паттернов из Scalaz. Напомню, что мы будем смотреть на них с практической точки зрения и избегать внутренних деталей Scalaz. В данной статье мы рассмотрим:

* Монада Writer: Ведёт своего рода логи во время выполнения набора операций.
* Монада State: Предоставляет простой способ пробрасывания состояния между функциями.
* Линзы: Предоставляет простой доступ к глубоко вложенным атрибутам и делает создание копий иерархии case class'ов более удобным.

<!--more-->

###Монада Writer

В своей основе каждый Writer имеет журнал и возвращаемое значение. Например, вы можете просто написать чистый код, а потом уже решить что вы хотите делать с логами(проверить их в тесте, вывести в консоль или в файл). Таким образом, мы можем использовать Writer для того, чтобы следить за операциями которые мы выполняем.

_Примечание от переводчика: монаду Writer можно использовать не только для логирования, но я буду придерживаться текста оригинальной статьи, а более подробно расскажу об этом в другой статье._

Итак, давайте посмотрим на следующий код для того, чтобы понять как это работает:

```scala
import scalaz._
import Scalaz._

object WriterSample extends App {

  // левая часть должна быть моноидом. Другими словами это нечто поддерживающее
  // "конкатенацию" и имеющее нейтральный элемент, например, String, List, Set и т.д.
  type Result[T] = Writer[List[String], T]

  def doSomeAction() : Result[Int] = {
    // провести вычисления для получения результата
    val res = 10
    // создать writer используя set
    res.set(List(s"Doing some action and returning res"))
  }

  def doingAnotherAction(b: Int) : Result[Int] = {
    // провести вычисления для получения результата
    val res = b * 2
    // создать writer используя set
    res.set(List(s"Doing another action and multiplying $b with 2"))
  }

  def andTheFinalAction(b: Int) : Result[String] = {
    val res = s"bb:$b:bb"

    // создать writer используя set
    res.set(List(s"Final action is setting $b to a string"))
  }

  // возвращает кортеж: (List, Int)
  println(doSomeAction().run)

  val combined = for {
    a <- doSomeAction()
    b <- doingAnotherAction(a)
    c <- andTheFinalAction(b)
  } yield c

  // возвращает кортеж: (List, String)
  println(combined.run)
}
```

В данном примере у нас есть три операции которые делают что-то. В данном случае, они на самом деле почти ничего не делают, но нам это и не важно. Основая идея в том, что мы вместо результата возвращаем Writer(заметим, что мы так же могли бы создать Writer внутри for-генератора) с помощью функции `set`. Когда мы вызываем `run` у Writer, мы получаем не только результат, но и агрегированные значения которые он собрал. Поэтому, когда мы делаем:

```scala
type Result[T] = Writer[List[String], T]

def doSomeAction() : Result[Int] = {
  // провести вычисления для получения результата
  val res = 10
  // создать writer используя set
  res.set(List(s"Doing some action and returning res"))
}

println(doSomeAction().run)
```

Результат будет выглядеть как `(List(Doing some action and returning res),10)`. Не то чтобы это очень захватывающе, но мы можем делать более интересные вещи используя writer в for-генераторе.

```scala
val combined = for {
  a <- doSomeAction()
  b <- doingAnotherAction(a)
  c <- andTheFinalAction(b)
} yield c

// возвращаем кортеж: (List, String)
println(combined.run)
```

Данный код выведет в консоль:

```
(List(Doing some action and returning res,
     Doing another action and multiplying 10 with 2,
     Final action is setting 20 to a string)
 ,bb:20:bb)
```

Как вы можете видеть, мы собрали все логи в `List[String]` и полученный кортеж так же содержит результат вычислений.

Если вы не хотите добавлять создание writer'ов в ваши функции вы можете создать их в for-генераторе:

```scala
val combined2 = for {
   a <- doSomeAction1()     set(" Executing Action 1 ")   // String тоже моноид
   b <- doSomeAction2(a)    set(" Executing Action 2 ")
   c <- doSomeAction2(b)    set(" Executing Action 3 ")
//  c <- WriterT.writer("bla", doSomeAction2(b))   // альтернативный вариант создания
 } yield c

 println(combined2.run)
```

В результате мы получим:

```
( Executing Action 1  Executing Action 2  Executing Action 3 ,5)
```

###Монада State

Другая интересная монада это монада State, она позволяет работать с состоянием которое должно быть передано через несколько функций. С помощью неё вы можете отслеживать результаты или передавать некоторый контекст между функциями. В предыдущей статье вы уже видели как можно передать некоторый контест в функцию с помощью монады Reader, однако, этот контест нельзя было изменять. Монада State предоставляет удобный способ безопасной и чистой передачи изменяемого контекста.

Давайте посмотрим на следующий пример:

```scala
case class LeftOver(size: Int)

/** State представляет из себя функцию `S => (S, A)`. */
type Result[A] = State[LeftOver, A]

def getFromState(a: Int): Result[Int] = {
  // делаем любые виды вычислений
  State[LeftOver, Int] {
    // просто возвращаем количество денег которое мы взяли
    // и новый State
    case x => (LeftOver(x.size - a), a)
  }
}

def addToState(a: Int): Result[Int] = {
  // делаем любые виды вычислений
  State[LeftOver, Int] {
    // просто возвращаем сумму которую мы добавили
    // и новый State
    case x => (LeftOver(x.size + a), a)
  }
}

val res: Result[Int] = for {
  _ <-  addToState(20)
  _ <- getFromState(5)
  _ <- getFromState(5)
  a <- getFromState(5)
  currentState <- get[LeftOver]                // получаем состояние в данный момент
  manualState <- put[LeftOver](LeftOver(9000)) // задаём новое состояние
  b <- getFromState(10) // и продолжаем работать с новым State
} yield {
  println(s"currenState: $currentState")
  a
}

// мы начинаем с состояния 10, и после всех операций мы остаёмся с 5
// без необходимости пробрасывать состояние с помощью implicits
// или чего-нибудь ещё
println(res(LeftOver(10)))
```

Как вы видите в каждой функции мы получаем текущий контекст, проделываем над ним какие-нибудь операции и возвращаем кортеж содержащий новый State и значение функции. Таким образом, каждая функция имеет доступ к состоянию, может вернуть новое состояние и возвращает его вместе со значением функции как кортеж. Когда мы запускаем данный код мы получаем:

```
currenState: LeftOver(15)
(LeftOver(8990),5)
```

В итоге, каждая функция делает что-то с состоянием. С помощью функции `get` мы можем получить значение в текущий момент времени и в данном примере мы это печатаем в консоль. Кроме того, мы можем непосредственно задать состояние с помощью функции `set`.

Как вы можете видеть это очень хороший и простой для использования паттерн, позволяющий пробрасывать состояние между функциями.

###Линзы

Давайте пока оставим монады и посмотрим на линзы. С помощью линз можно просто(ну проще чем просто копировать case class'ы руками) изменять значения в иерархии вложенных объектов. Линзы позволяют сделать очень многое, но в этой статье мы остановимся на нескольких основных вещах. Давайте посмотрим на код:

```scala
import scalaz._
import Scalaz._

object LensesSample extends App {

  // паршивая case модель с отсутствием креатива
  case class Account(userName: String, person: Person)
  case class Person(firstName: String, lastName: String, address: List[Address], gender: Gender)
  case class Gender(gender: String)
  case class Address(street: String, number: Int, postalCode: PostalCode)
  case class PostalCode(numberPart: Int, textPart: String)

  val acc1 = Account("user123", Person("Jos", "Dirksen",
                List(Address("Street", 1, PostalCode(12,"ABC")),
                     Address("Another", 2, PostalCode(21,"CDE"))),
                Gender("male")))


  val acc2 = Account("user345", Person("Brigitte", "Rampelt",
                List(Address("Blaat", 31, PostalCode(67,"DEF")),
                     Address("Foo", 12, PostalCode(45,"GHI"))),
                Gender("female")))


  // Если теперь вы хотите изменить что-то, скажем, изменить пол (просто потому, что можем) нам нужно начать копировать всё подряд  
  val acc1Copy = acc1.copy(
    person = acc1.person.copy(
      gender = Gender("something")
    )
  )
```

В данном примере мы определили несколько case class'ов и хотим изменить одно значение. Для case class'ов это означает, что мы должны скопировать всю иерархию объектов для изменения вложенных значений. Для простых случаев это выглядит не сложно, но используя такой подход код довольно быстро становится громоздким. С линзами мы можем получить механизм для того чтобы сделать это в более композитном стиле:

```scala
val genderLens = Lens.lensu[Account, Gender](
   (account, gender) => account.copy(person = account.person.copy(gender = gender)),
   (account) => account.person.gender
 )

 // и используя линзы мы можем непосредственно изменить пол
 val updated = genderLens.set(acc1, Gender("Blaat"))
 println(updated)

#Вывод: Account(user123,Person(Jos,Dirksen,List(Address(Street,1,PostalCode(12,ABC)),
         Address(Another,2,PostalCode(21,CDE))),Gender(Blaat)))
```

Таким образом мы определили линзу, которая может изменить конкретное значение в иерархии. С помощью этой линзы мы можем напрямую обратиться и установить новое значение во вложенной иерархии. Мы так же можем создать линзу которая изменяет значение и возвращает модифицированный объект с помощью оператора `=>=`.

```scala
// мы можем использовать базовую линзу для создания модифицирующей линзы
val toBlaBlaLens = genderLens =>= (_ => Gender("blabla"))
println(toBlaBlaLens(acc1))
#Вывод:  Account(user123,Person(Jos,Dirksen,List(Address(Street,1,PostalCode(12,ABC)),
          Address(Another,2,PostalCode(21,CDE))),Gender(blabla)))

val existingGender = genderLens.get(acc1)
println(existingGender)
#Вывод: Gender(male)
```

И мы можем использовать операторы `>=>` и `<=<` для того, чтобы скомбинировать линзы. Например, в следующем примере мы создаём отдельные линзы, которые потом комбинируются и выполняются:

```scala
// для начала создадим линзу возвращающую "человека"
val personLens = Lens.lensu[Account, Person](
  (account, person) => account.copy(person = person),
  (account) => account.person
)

// получаем фамилию
val lastNameLens = Lens.lensu[Person, String](
  (person, lastName) => person.copy(lastName = lastName),
  (person) => person.lastName
)


// получаем человека, потом берём фамилию и задаём новую фамилию
val combined = (personLens >=> lastNameLens) =>= (_ => "New LastName")

println(combined(acc1))

#Вывод: Account(user123,Person(Jos,New LastName,List(Address(Street,1,PostalCode(12,ABC)),
          Address(Another,2,PostalCode(21,CDE))),Gender(male)))
```

###Заключение

Есть ещё две темы о которых я хочу написать, это Validations и свободные монады. В следующей статье я покажу как можно использовать ValidationNEL. Однако, я думаю, что свободные монады не попадают в категорию вещей для ежедневного использовая, поэтому, возможно, я напишу о них в других статьях в будущем.

[Оригинальная статья.](http://www.smartjava.org/content/scalaz-features-everyday-usage-part-3-state-monad-writer-monad-and-lenses)
