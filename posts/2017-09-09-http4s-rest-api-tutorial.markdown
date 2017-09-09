---
title: Создание простого REST сервера на http4s и doobie
author: BeiZero
tags: scala, scalaz, http4s
description: В этой статье я расскажу о базовых принципах работы с библиотекой http4s и о том как с помощью неё создать простой REST сервер.
---

В этой статье я расскажу о базовых принципах работы с библиотекой http4s и о том как с помощью неё создать простой REST сервер.


### "Hello, world!" на http4s

Для создания каркаса приложения мы можем воспользоваться уже готовым шаблоном используя [Giter8](http://www.foundweekends.org/giter8/). Выполним в консоле команду и заполним основные переменные _name_, _organization_, _package_, _scala_version_, _http4s_version_:

```bash  
sbt -sbt-version 0.13.15 new http4s/http4s.g8

name [http4s quickstart]: http4s-rest-api-example
organization [com.example]: io.github.macs_club
package [io.github.macs-club.http4srestapiexample]:
scala_version [2.12.2]: 2.12.3
http4s_version [0.15.11a]: 0.16.0a

Template applied in ./http4s-rest-api-example
```

Это команда создаст в директории `http4s-rest-api-example` следующие файлы:

```bash
build.sbt
project/build.properties
src/main/resources/logback.xml
src/main/scala/io/github/macs-club/http4srestapiexample/HelloWorld.scala
src/main/scala/io/github/macs-club/http4srestapiexample/Server.scala
```

Ну и давайте сразу запустим то, что у нас получилось командой `sbt run`. После разрешения зависимостей и компиляции мы увидим:

```scala
[run-main-0] INFO  o.h.b.c.n.NIO1SocketServerGroup - Service bound to address /0:0:0:0:0:0:0:0:8080
[run-main-0] INFO  o.h.s.ServerApp - Started server on /0:0:0:0:0:0:0:0:8080
```
Это говорит о том, что blaze, собственный серверный бэкенд http4s, запустил наш сервис на порту 8080. Можем отправить ему "Hello, world!" запрос curl'ом:

```bash
bash-3.2$ curl localhost:8080/hello/world!
{"message":"Hello, world!"}
```

Теперь рассмотрим содержимое файла `HelloWorld.scala` и разберём как он работает. Основная логика здесь скрывается в переменной `helloWorldService` типа `HttpService`. `HttpService` это просто алиас к `Kleisli[Task, Request, Response]`. Если вы не понимаете, что это значит, то ничего страшного. В данном случае `Kleisli` это обёртка над `Request => Task[Response]` и `Task` это асинхронная операция.

Для работы с `HttpService` импортируются зависимости:

```scala
import org.http4s._, org.http4s.dsl._
```

Используя http4s-dsl можно создать `HttpService` используя сопоставление с образцом для запроса. Давайте посмотрим как устроен сервис, к которому мы обращались в предудещей части статьи, который обрабатывает запрос `GET /hello/:name`, где `:name` параметр:

```scala
val helloWorldService = HttpService {
  case GET -> Root / "hello" / name =>
     Ok(Json.obj("message" -> Json.fromString(s"Hello, ${name}")))
}
```

Теперь перейдём к `Server.scala` и разберём как запускается сервер. http4s поддерживает несколько серверных бэкнендов. В данном случае используется blaze. Внутри файла находится объект `BlazeExample` который реализует функцию `server` специального трейта `ServerApp` предназначенного для работы с сервером. Используя `BlazeBuilder` в функции `server` монтируется `helloWorldService` по корневому пути `/`, далее передаётся `ExecutorService` и команда `start` для запуска сервера. Сервисы можно монтировать в любом порядке, а запрос будет сопоставлять сначала с самым длинным из путей. `BlazeBuilder` неизменяемый с цепными методами, каждый из которых возвращает новый `BlazeBuilder`.

```scala
override def server(args: List[String]): Task[Server] =
  BlazeBuilder
    .bindHttp(port, ip)
    .mountService(HelloWorld.helloWorldService)
    .withServiceExecutor(pool)
    .start
```

`bindHttp` не является необходимым, по умолчанию сервер будет настроен на запуск по `localhost:8080`. `mountService` связывает путь с `HttpService`.

### Приложение заметок

Давайте используя http4s создадим простенький REST API для работы со списком заметок. Создадим класс представляющий наши записки:
```scala
package io.github.macs_club.http4srestapiexample.domain

case class Note(title: String, body: String = "")
```

Для работы с базой данных будем использовать библиотеку _doobie_ и встроенную базу данных _h2_. Для этого добавим в `build.sbt` необходимые зависимости:

```scala
val DoobieVersion = "0.4.4"

libraryDependencies ++= Seq(
  "org.tpolecat"  %% "doobie-core"        % DoobieVersion,
  "org.tpolecat"  %% "doobie-h2"          % DoobieVersion,
  "org.tpolecat"  %% "doobie-specs2"      % DoobieVersion
)
```

А так же добавим дополнительную зависимость библиотеки _circe_ для автоматического вывода кодеков для _json_.

```scala
val CirceVersion = "0.8.0"

libraryDependencies ++= Seq(
  "io.circe"      %% "circe-generic"      % CirceVersion
)
```

Для доступа к базе данных будем использовать паттерн репозиторий. Создадим абстрактный класс `Repository`:
```scala
package io.github.macs_club.http4srestapiexample.repository

import scalaz.Monad
import scala.language.higherKinds

abstract class Repository[A, M[_]: Monad] {
  def +=(entity: A): M[A]
  def update(entity: A): M[A]
  def -=(entity: A): M[Unit]
  def list: M[List[A]]
}
```

Здесь используется обобщённый тип `A`, который будет соответствовать сущности с которой мы будем работать, и обобщённый тип `M` являющийся монадой для оборачивания результата работы с базой данных.

Напишем реализацию для нашего типа `Note`. В качестве монады в которую будем упаковывать результат будем использовать `Task`:

```scala
package io.github.macs_club.http4srestapiexample.repository.impl

import scalaz._, Scalaz._
import doobie.imports._
import doobie.h2.imports._
import scalaz.concurrent.Task

import io.github.macs_club.http4srestapiexample.domain.Note
import io.github.macs_club.http4srestapiexample.repository.Repository

object H2NoteRepository {
  def apply() = Task {
      val h2nr = new H2NoteRepository()
      h2nr.init.unsafePerformSync
      h2nr
  }
}

class H2NoteRepository extends Repository[Note, Task]{

  val xa = H2Transactor[Task]("jdbc:h2:mem:http4srestapiexample;DB_CLOSE_DELAY=-1", "h2username", "h2password")

  private def init = {
      val query = sql"""
        CREATE TABLE IF NOT EXISTS note (
          title VARCHAR(20) NOT NULL UNIQUE,
             body  VARCHAR(20)
        )
      """.update.run
      xa >>= (query.transact(_))
  }

  override def +=(note: Note) = {
      val insertQ = sql"""
            INSERT INTO note (title, body)
            VALUES (${note.title},${note.body})
        """.update.run
    val selectQ = sql"""
            SELECT title, body
            FROM note
        WHERE title = ${note.title}
        """.query[Note].unique
      val query = insertQ *> selectQ
      xa >>= (query.transact(_))
  }

  override def update(note: Note) = {
      val updateQ = sql"""
            UPDATE note
            SET body = ${note.body}
            WHERE title = ${note.title}
        """.update.run
      val selectQ = sql"""
            SELECT title, body
            FROM note
        WHERE title = ${note.title}
        """.query[Note].unique
      val query = updateQ *> selectQ
      xa >>= (query.transact(_))
  }

  override def -=(note: Note) = {
      val query = sql"""
            DELETE FROM note
            WHERE title = ${note.title}
        """.update.run *> FC.unit
      xa >>= (query.transact(_))
  }

  override def list = {
      val query = sql"""
            SELECT title, body
            FROM note
        """.query[Note].list
      xa >>= (query.transact(_))
  }
}
```

Теперь рассмотрим подробнее что же здесь происходит.

`Transactor` это обёртка над пуллом соединений. Т.к. у транзактора есть внутренние состояние, то его создание это сайд эффект и его необходимо обернуть, в данном случае используется `Task`. Т.к. при создании репозитория мы хотим ещё и создать таблицу в которой у нас будут храниться наши сущности, то похожую вещь провернём и с созданием репозитория, т.е. в функции _apply_ объекта компаньона создадим репозиторий, потом создадим таблицу, вернём наш репозиторий и завернём всё это в `Task`.

Каждому запросу для запуска необходимо передать транзактор. Т.к. транзактор обёрнут в монаду `Task` то доступ к нему происходит через оператор `>>=`(алиас для функции `flatMap`), в который мы передаём вызов функции `transact` нашего запроса, которая так же возвращает `Task`.

Запросы `+=, update, -=` используют комбинирование нескольких запросов с помощью `*>` и они выполняются их в рамках одной транзакции.

Теперь необходимо создать сервис с помощью которого мы будем с работать с заметками:
```scala
package io.github.macs_club.http4srestapiexample.service

import scalaz._, Scalaz._
import io.circe._
import org.http4s._
import org.http4s.circe._
import io.circe.generic.auto._
import org.http4s.dsl._
import scalaz.concurrent.Task

import io.github.macs_club.http4srestapiexample.domain.Note
import io.github.macs_club.http4srestapiexample.repository.Repository
import io.github.macs_club.http4srestapiexample.repository.impl.H2NoteRepository

object NoteService {
  implicit def circeJsonDecoder[A](implicit decoder: Decoder[A]): EntityDecoder[A] = org.http4s.circe.jsonOf[A]
  implicit def circeJsonEncoder[A](implicit encoder: Encoder[A]): EntityEncoder[A] = org.http4s.circe.jsonEncoderOf[A]

  val repository: Task[Repository[Note, Task]] = H2NoteRepository()

  val service = HttpService {
      case GET -> Root => repository >>= (_.list) >>= (Ok(_))
      case req @ POST -> Root => req.decode[Note]{ data =>
         Ok(repository >>= (_ += data))
      }
      case req @ PUT -> Root => req.decode[Note]{ data =>
         Ok(repository >>= (_ update data))
      }
      case DELETE -> Root / name => Ok(repository >>= (_ -= Note(name)))
  }
}
```

Сервис создаётся на основе репозитория и обрабатывает сообщения на основе сопоставления с образцом. Здесь так же объявлены специальные неявные преобразования предоставляющие `EntityDecoder` и `EntityEncoder` для работы с json.

Теперь остаётся только примонтировать новый сервис в `BlazeBuilder`. Теперь можно запустить сервер и создавать/получать/удалять/изменять записки.

```shell
$ curl -X POST localhost:8080 -d '{"title":"Test Title","body":"Test Body"}'
{"title":"Test Title","body":"Test Body"}
$ curl  localhost:8080
[{"title":"Test Title","body":"Test Body"}]
$ curl -X DELETE localhost:8080/Test%20Title
$ curl  localhost:8080
[]
```

[Исходники проекта](https://github.com/MaCS-Club/http4s-rest-api-example)
