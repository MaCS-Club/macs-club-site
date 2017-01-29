---
title: Scalaz для ежедневного использования. Часть 2. Монадные трансформеры и монада Reader.
author: BeiZero
tags: scala, scalaz для ежедневного использования, scalaz, перевод
---

Во второй статье "Scalaz для ежедневного использования" мы рассмотрим монадные трансформеры и монаду Reader. Начнём с монадных трансформеров. Они пригодятся когда вам нужно иметь дело с вложенными монадами, что происходит удивительно часто. Например, когда вам нужно работать с вложенными Future[Option] или Future[Either], ваши for-генераторы довольно быстро могут стать не читаемыми, так как вам нужно обрабатывать None и Some для Option, а так же Success и Failure для Future явно.

<!--more-->

###Работаем без монадных трансформеров

Как мы уже говорили во введении монадные трансформеры полезны при работе с вложенными монадами. Когда вы можете с ними столкнуться? Например, библиотеки для работы с базами данных обычно ассинхронные(используют Future) и в некоторых случаях возвращают Option. Предположим, вы запрашиваете специальную запись которая возвращает Future[Option[T]]:

```scala
# Phantom драйвер для cassandra (без implicits) возвращает Some(Record) если запись найдена
# или None иначе
def one(): Future[Option[Record]]

# или вы можете захотеть получить первый элемент из запроса Slick
val res : Future[Option[Row]] = db.run(filterQuery(id).result.headOption)
```

Или вы можете просто иметь свой сервис, определяющий функции, которые возвращают Either или Option как результат:

```scala
# возвращает аккаунт или None если аккаунт не найден
def getAccount() : Future[Option[Account]]

# снимает сумму со счёта, возвращает новую сумму на счёте
# или сообщение объясняющее что пошло не так
def withdraw(account: Account, amount: Amount) : Future[\/[String, Amount]]
```

Давайте рассмотрим пример некрасивого кода который может получиться если не использовать монадные трансформеры:

```scala
 def withdrawWithoutMonadTransformers(accountNumber: String, amount: Amount) : Future[Option[Statement]] = {
  for {
    // возвращает Future[Option[Account]]
    account <- Accounts.getAccount(accountNumber)
    // мы можем сделать свёртку, используя функцию из scalaz для получения "нетипизированного" None
    balance <- account.fold(Future(none[Amount]))(Accounts.getBalance(_))
    // или иногда нам нужно использовать сопоставление с образцом
    _ <- (account, balance) match {
      case (Some(acc), Some(bal)) => Future(Accounts.withdraw(acc,bal))
      case _ => Future(None)
    }
    // или мы можем сделать вложенный map
    statement <- Future(account.map(Accounts.getStatement(_)))
  } yield statement
}
```

Как мы видим, когда нам нужно работать с вложенными монадами, приходится обрабатывать внутренние монады пошагово в правой части for-генератора. Scala очень богатый язык, поэтому у нас есть много способов сделать это, но код не получается более читаемым. Мы вынуждены прибегать к вложенным map и flatmap, использовать fold(в случае Option) или прибегать к помощи сопоставления с образцом, если нас интересует несколько Option. Есть, вероятно, и другие способы сделать это, но код врядли станет заметно лучше. Так как мы имеем дело с Option в явном виде.

###А теперь с монадными трансформерами

С монадными трансформерами мы можем удалить из кода всё лишнее и получить удобный инструмент для работы с подобными вложенными конструкциями. Scalaz предоставляет монадные трансформеры следующих типов:

```scala
BijectionT
EitherT
IdT
IndexedContsT
LazyEitherT
LazyOptionT
ListT
MaybeT
OptionT
ReaderWriterStateT
ReaderT
StateT
StoreT
StreamT
UnWriterT
WriterT
```

В то время как некоторые из них могут показаться немного экзотичными, ListT, OptionT, EitherT, ReaderT и WriterT имеют очень много вариантов использования. В первом примере мы сосредоточимся на монадном трансформере OptionT. Давайте посмотрим как мы можем создать монаду OptionT. В нашем случае нужно создать OptionT[Future, A], который упаковывает Option[A] внутрь Future. Мы можем создать его из A:

```scala
scala> :require /Users/jos/.ivy2/cache/org.scalaz/scalaz-core_2.11/bundles/scalaz-core_2.11-7.2.1.jar
Added '/Users/jos/.ivy2/cache/org.scalaz/scalaz-core_2.11/bundles/scalaz-core_2.11-7.2.1.jar' to classpath.

scala> import scalaz._
import scalaz._

scala> import Scalaz._
import Scalaz._                                ^

scala> import scala.concurrent.Future
import scala.concurrent.Future

scala> import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.ExecutionContext.Implicits.global

scala> type Result[A] = OptionT[Future, A]
defined type alias Result

scala> 1234.point[Result]
res1: Result[Int] = OptionT(scala.concurrent.impl.Promise$DefaultPromise@1c6ab85)

scala> "hello".point[Result]
res2: Result[String] = OptionT(scala.concurrent.impl.Promise$DefaultPromise@3e17219)

scala> res1.run
res4: scala.concurrent.Future[Option[Int]] = scala.concurrent.impl.Promise$DefaultPromise@1c6ab85
```

Обратите внимание что мы явно определяем тип Result для того, чтобы функция point работала. Если вы этого не сделаете вы получите сообщение об ошибке:

```scala
scala> "why".point[OptionT[Future, String]]
<console>:16: error: scalaz.OptionT[scala.concurrent.Future,String] takes no type parameters, expected: one
              "why".point[OptionT[Future, String]]
```

Вы можете использовать point только тогда, когда вы имеете дело с внутренним значением монадного трансформера. Если у вас уже есть Future или Option, придётся использовать конструктор OptionT.

```scala
scala> val p: Result[Int] = OptionT(Future.successful(some(10)))
p: Result[Int] = OptionT(scala.concurrent.impl.Promise$KeptPromise@40dde94)
```

С монадными трансформерами вы можете автоматически развернуть вложенную монаду. Теперь, когда мы знаем как преобразовать наши значения к OptionT, давайте посмотрим как можно переписать предыдущий пример:

```scala
def withdrawWithMonadTransformers(accountNumber: String, amount: Amount) : Future[Option[Statement]] = {

  type Result[A] = OptionT[Future, A]

  val result = for {
    account <- OptionT(Accounts.getAccount(accountNumber))
    balance <- OptionT(Accounts.getBalance(account))
    _ <- OptionT(Accounts.withdraw(account,balance).map(some(_)))
    statement <- Accounts.getStatement(account).point[Result]
  } yield statement

  result.run
}
```

Неплохо правда? Вместо всей боли с приведением типов к корретным, мы просто создаём OptionT и возвращаем их. Для того чтобы получить сохранённое значение из OptionT мы просто вызываем run.

Несмотря на то что код стал уже гораздо лучше, мы всё ещё имеем много шума связанного с созданием OptionT.

###А теперь сделаем наш код ещё чище

Мы можем очистить наш код ещё немного:

```scala
type Result[A] = OptionT[Future, A]

object ResultLike {
  def applyFO[A](a: Future[Option[A]]) : Result[A] = OptionT(a)
  def applyF[A](a: Future[A]) : Result[A] = OptionT(a.map(some(_)))
  def applyP[A](a: A) : Result[A] = a.point[Result]
}

def withdrawClean(accountNumber: String, amount: Amount) : Future[Option[Statement]] = {

  val result: Result[Statement] = for {
    account <- Accounts.getAccount(accountNumber)         |> ResultLike.applyFO
    balance <- Accounts.getBalance(account)               |> ResultLike.applyFO
    _ <- Accounts.withdraw(account,balance)               |> ResultLike.applyF
    statement <- Accounts.getStatement(account)           |> ResultLike.applyP
  } yield statement

  result.run
}
```

При таком подходе мы просто создаём специальные конверторы, чтобы получить результат в монаде OptionT. В результате текущий for-генератор выглядит очень читаемо без какого-либо хлама. В правой части, не нагромождая "полезный" код, мы делаем преобразование в OptionT. Обратите внимание, что это не самое чистое решение так как нам приходится задавать разные apply функции. Перегрузка функций здесь не работает потому, что после стирания окончаний функции `applyFO` и `applyF` будут иметь одинаковую сигнатуру.

##Монада Reader

Монада Reader одна из стандартных монад предоставляемых Scalaz. Монада Reader может быть использована для того чтобы выносить конфигурацию(или другие значения) и для таких вещей как инъекция зависимостей.

###Решение с монадой Reader

Монада Reader позволяет делать инъекцию зависимостей в Scala. Независимо от того, является ли зависимость объектом конфигурации или ссылкой на какой-либо другой сервис. Мы начнём с примера который лучше всего объясняет как использовать монаду Reader.

Для этого примера предположим что у нас есть сервис, который требует Session, чтобы делать что-либо. Это может быть, например, сессия базы данных или что-то другое. Давайте в качестве примера возьмём код из предыдущего примера и немного его упростим убрав Future:

```scala
trait AccountService {
  def getAccount(accountNumber: String, session: Session) : Option[Account]
  def getBalance(account: Account, session: Session) : Option[Amount]
  def withdraw(account: Account, amount: Amount, session: Session) : Amount
  def getStatement(account: Account, session: Session): Statement
}

object Accounts extends AccountService {
  override def getAccount(accountNumber: String, session: Session): Option[Account] = ???
  override def getBalance(account: Account, session: Session): Option[Amount] = ???
  override def withdraw(account: Account, amount: Amount, session: Session): Amount = ???
  override def getStatement(account: Account, session: Session): Statement = ???
}
```

Это немного раздражает, так как каждый раз, когда мы хотим вызвать один из сервисов, мы должны предоставить реализацию Session. Мы могли бы, конечно, сделать Session implicit'ом, но тогда нам нужно убедиться, что он находится в нашей области видимости, перед тем как вызывать функции этой службы. Было бы круто если бы мы могли внедрить эту сессию каким-нибудь образом. Мы могли бы, конечно, сделать это в конструкторе сервиса, но мы можем так же использовать монаду Reader для этого:

```scala
// введение в тип Action. Он представляет из себя действие которое наш сервис может выполнить.
// Как вы видите Action требует Session.
type Action[A] = Reader[Session, A]

trait AccountService {
  // возвращает аккаунт или None, когда аккаунт не найден
  def getAccount(accountNumber: String) : Action[Option[Account]]
  // возвращает баланс если аккаунт открыт или None иначе
  def getBalance(account: Account) :Action[Option[Amount]]
  // снять сумму со счета, и вернуть новое значение
  def withdraw(account: Account, amount: Amount) : Action[Amount]
  // мы можем также получить обзор выписки со счета
  def getStatement(account: Account): Action[Statement]
}

object Accounts extends AccountService {
  override def getAccount(accountNumber: String): Action[Option[Account]] = Reader((session: Session) => {
    // сделать что-нибудь с сессией здесь и вернуть результат
    session.doSomething
    some(Account())
  })

  override def getBalance(account: Account): Action[Option[Amount]] = Reader((session: Session) => {
    // сделать что-нибудь с сессией здесь и вернуть результат
    session.doSomething
    some(Amount(10,"Dollar"))
  })

  override def withdraw(account: Account, amount: Amount): Action[Amount] = Reader((session: Session) => {
    // сделать что-нибудь с сессией здесь и вернуть результат
    session.doSomething
    Amount(5, "Dollar")
  })

  override def getStatement(account: Account): Action[Statement] = Reader((session: Session) => {
    // сделать что-нибудь с сессией здесь и вернуть результат
    session.doSomething
    Statement(account)
  })
}
```

Как вы можете видеть, мы не возвращаем результат, но оборачиваем его в Reader. Самое замечательное в том, что теперь мы можем начать обрабатывать результат, так как Reader это просто монада.

```scala
def withdrawWithReader(accountNumber: String) = {

  for {
    account <- Accounts.getAccount(accountNumber)
    balance <- account.fold(Reader((session: Session) => none[Amount]))(ac => Accounts.getBalance(ac))
    _ <- (account, balance) match {
      case (Some(acc), Some(bal)) => Accounts.withdraw(acc,bal)
      case _ => Reader((session: Session) => none[Amount])
    }
  statement <- account match { case Some(acc) => Accounts.getStatement(acc)}
  } yield statement
}
```

Данный код не вернёт нам фактическое конечное значение, но вернёт Reader. Теперь мы можем запустить код передав Session:

```scala
// функция возвращает 'шаги' для выполнения, чтобы выполнить эти шаги, нужно запустить run в контексте 'новой сессии'
withdrawWithReader("1234").run(new Session())
```

Когда вы вновь посмотрите на  withdrawWithReader вы увидите, что мы вновь должны работать с монадой Option в явном виде и убедиться, что мы всегда создаём Reader в качестве результата. К счастью, Scalaz предоставляет ReaderT. В следующем коде мы покажем, как это делается на примере:

```scala
// введение в тип Action. Он представляет из себя действие которое наш сервис может выполнить.
// Как вы видите Action требует Session.
type Action[A] = ReaderT[Session, A]

trait AccountService {
  // возвращает аккаунт или None, когда аккаунт не найден
  def getAccount(accountNumber: String) : Action[Option[Account]]
  // возвращает баланс если аккаунт открыт или None иначе
  def getBalance(account: Account) :Action[Option[Amount]]
  // снять сумму со счета, и вернуть новое значение
  def withdraw(account: Account, amount: Amount) : Action[Amount]
  // мы можем также получить обзор выписки со счета
  def getStatement(account: Account): Action[Statement]
}

object Accounts extends AccountService {
  override def getAccount(accountNumber: String): Action[Option[Account]] = Reader((session: Session) => {
    // сделать что-нибудь с сессией здесь и вернуть результат
    session.doSomething
    some(Account())
  })

  override def getBalance(account: Account): Action[Option[Amount]] = Reader((session: Session) => {
    // сделать что-нибудь с сессией здесь и вернуть результат
    session.doSomething
    some(Amount(10,"Dollar"))
  })

  override def withdraw(account: Account, amount: Amount): Action[Amount] = Reader((session: Session) => {
    // сделать что-нибудь с сессией здесь и вернуть результат
    session.doSomething
    Amount(5, "Dollar")
  })

  override def getStatement(account: Account): Action[Statement] = Reader((session: Session) => {
    // сделать что-нибудь с сессией здесь и вернуть результат
    session.doSomething
    Statement(account)
  })
}

def withdrawWithReaderT(accountNumber: String) = {
  for {
    account <- Accounts.getAccount(accountNumber)
    balance <- Accounts.getBalance(account)
    _ <- Accounts.withdraw(account, balance)
    statement <- Accounts.getStatement(account)
  } yield statement
}

withdrawWithReaderT("1234").run(new Session)
```

Как вы видите не многое изменилось. Главное, что мы изменили это определение Action, он теперь использует ReaderT вместо Reader. Мы изменили трейт и его реализацию для того чтобы работать с ним. Теперь, когда вы посмотрите на функцию withdrawWithReaderT вы увидите что нам больше не нужно обрабатывать Option, но он обрабатывается нашим ReaderT(который на самом деле является Kleisli, но это тема для другой статьи). Круто, правда?

Мы видим что это отлично работает для Option, что случится если мы вернёмся назад к оригинальному примеру и захотим использовать Option внутри Future и это всё внутри Reader? Ну этот момент выходит за рамки "Scalaz функций для повседневного использования", но базовый подход такой же:

```scala
// введение в тип Action. Он представляет из себя действие которое наш сервис может выполнить.
// Как вы видите Action требует Session.
type OptionTF[A] = OptionT[Future, A]
type Action[A] = ReaderT[OptionTF, Session, A]

trait AccountService {
  // возвращает аккаунт или None, когда аккаунт не найден
  def getAccount(accountNumber: String) : Action[Account]
  // возвращает баланс если аккаунт открыт или None иначе
  def getBalance(account: Account) :Action[Amount]
  // снять сумму со счета, и вернуть новое значение
  def withdraw(account: Account, amount: Amount) : Action[Amount]
  // мы можем также получить обзор выписки со счета
  def getStatement(account: Account): Action[Statement]
}

object Accounts extends AccountService {
  override def getAccount(accountNumber: String): Action[Account] = ReaderT((session: Session) => {
    // сделать что-нибудь с сессией здесь и вернуть результат
    session.doSomething
    // предположим мы получаем Future[Option[Account]]
    val result = Future(Option(Account()))

    // и нам нужно поднять её в OptionTF и вернуть его
    val asOptionTF: OptionTF[Account] = OptionT(result)
    asOptionTF
  })

  override def getBalance(account: Account): Action[Amount] = ReaderT((session: Session) => {
    // сделать что-нибудь с сессией здесь и вернуть результат
    session.doSomething
    // предположим мы получаем Future[Option[Amount]]
    val result = Future(some(Amount(10,"Dollar")))
    // преобразуем это к типу Action c явным типом чтобы сделать компилятор счастливым
    val asOptionTF: OptionTF[Amount] = OptionT(result)
    asOptionTF
  })

  override def withdraw(account: Account, amount: Amount): Action[Amount] = ReaderT((session: Session) => {
    // сделать что-нибудь с сессией здесь и вернуть результат
    session.doSomething
    // предположим мы получаем Future[Amount]
    val result = Future(Amount(5, "Dollar"))
    // приводим к корректному типу
    val asOptionTF: OptionTF[Amount] = OptionT(result.map(some(_)))
    asOptionTF
  })

  override def getStatement(account: Account): Action[Statement] = ReaderT((session: Session) => {
    // сделать что-нибудь с сессией здесь и вернуть результат
    session.doSomething
    // предположим мы получаем Statement
    val result = Statement(account)
    // приведём к корректному типу
    result.point[OptionTF]
  })
}

def withdrawWithReaderT(accountNumber: String) = {
  for {
    account <- Accounts.getAccount(accountNumber)
    balance <- Accounts.getBalance(account)
    _ <- Accounts.withdraw(account, balance)
    statement <- Accounts.getStatement(account)
  } yield statement
}

// это результат завёрнутый в Option
val finalResult = withdrawWithReaderT("1234").run(new Session)
// получаем Future[Option] и ждём результата
println(Await.result(finalResult.run, 5 seconds))
```

Мы определили другой тип ReaderT, в который положили OptionT вместо просто Option. Этот OptionT будет обрабатывать преобразования Option/Future. Так как мы получили новый ReaderT, нам, конечно, нужно поднять результаты вызовов нашего сервиса к этой монаде, которые нуждаются в несколько насильственном приведении для того чтобы компилятор всё понял. Результат будет очень приятным. Текущий for-генератор остаётся точно таким же, но на этот раз может обрабатывать Option внутри Future внутри Reader!

###Заключение

В этой статье мы рассмотрели две вещи из Scalaz, которые действительно пригодятся при работе с вложенными монадами или если вы хотите лучше управлять зависимостями между компонентами. Особенно крут тот факт, что довольно легко использовать монадные трансформеры вместе с монадой Reader. В результате, сделав пару небольших шагов мы можем полностью скрыть детали реализации(в данном случае) связанные с Future и Option, и получить очень красивые и чистые for-генераторы и другие монадические вкусности.

[Оригинальная статья.](http://www.smartjava.org/content/scalaz-features-everyday-usage-part-2-monad-transformers-and-reader-monad)
