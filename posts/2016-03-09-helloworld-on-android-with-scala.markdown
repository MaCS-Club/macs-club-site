---
title: Hello, World! на Scala под Android
author: BeiZero
tags: scala, android, scaloid
description: В данной статье я постараюсь рассказать о том как начать разрабатывать приложения под Android используя Scala и SBT. Предполагается что у вас уже установлены Android SDK и SBT.
---
В данной статье я постараюсь рассказать о том как начать разрабатывать приложения под Android используя Scala и SBT. Предполагается что у вас уже установлены Android SDK и SBT.
<!--more-->
Первое что нам нужно сделать это указать переменную окружения ANDROID_HOME. На OS X с Android SDK установленным через Homebrew для этого нужно выполнить команду:

```ShellSession
export ANDROID_HOME=/usr/local/Cellar/android-sdk/{sdk-version}
```

Не забудьте запустить Android SDK Manager и установить все нужные вам пакеты.
Для сборки проекта мы будем использовать SBT(в данный момент у меня стоит версия 0.13.11) с плагином android-sdk-plugin. В директории будущего проекта создадим файл

```
/project/plugins.sbt
```

со следующим содержимым

```scala
addSbtPlugin("com.hanhuy.sbt" % "android-sdk-plugin" % "1.5.19")
```

запустим sbt из директории с проектом и выполним следующую команду

```
gen-android android-23 org.my_company project_name
```

первый аргумент здесь platform-target(23 соответствует версии Android 6.0 Marshmallow), второй package-name, третий project-name, а так же укажем в build.sbt версию Scala(нам понадобится версия 2.11.5 или новее) добавив строку

```scala
scalaVersion := "2.11.8"
```

После этого мы уже можем собрать наш проект и посмотреть что же получилось командой

```
sbt android:package
```

К сожалению иногда proguard не запускается, возможно из-за отсутствия “существенных” изменений в проекте, и когда вы набираете достаточное количество кода и зависимостей в своём проекте компилятор начинает иногда выдавать ошибку вида

```
trouble writing output: Too many method references: 94505; max is 65536.
```

один из вариантов решения проблемы это запускать сборку проекта предварительно его “очистив”

```
sbt clean && sbt android:package
```

Самое время сделать наш код немного лучше. Откроем файл

```
/src/main/scala/org/my_company/sample.scala
```

тем кто уже знаком с разработкой под Android на Java данный код будет более чем знаком и в этом заключается основаная проблема — стандартная библиотека Android не предназначена для написания программ на Scala и код остаётся всё таким же громоздким, но к счастью есть замечательная библиотека scaloid! Добавим эту библиотеку в зависимости к нашему проекту и в исключения к proguard(странно, но без этого компиляция фейлится), для этого в build.sbt добавим следующие строчки

```scala
libraryDependencies += "org.scaloid" %% "scaloid" % "4.2"

proguardOptions in Android ++= Seq("-dontwarn org.scaloid.**")
```

теперь перепишем наш файл sample.scala используя scaloid

```scala
package org.my_company

import org.scaloid.common._

class MainActivity extends SActivity {

   onCreate {

      contentView = new SVerticalLayout {

         SButton("Say Hello", toast("Hello, World!"))

      }

   }

}
```

На этом всё, можем собрать наш проект, запустить, нажать кнопку “Say Hello” и увидим заветные два слова.
