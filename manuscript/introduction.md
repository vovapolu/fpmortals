
# Вступление

Быть скептическим насчет новой парадигмы естественно для человека. 
Чтобы обозначить некоторую перспективу насколько далеко мы зашли и 
какие изменения мы уже приняли для JVM, давайте начнем с быстрого 
повторения последних 20 лет. 

Java 1.2 добавила Collections API, что позволило нам создавать методы, 
абстрагированные над изменяемыми (((В противовес неизменяемым (immutable)))) коллекциями. Это было удобно для
создания алгоритмов общего назначения и, по сути, было основой нашего 
кода. 

Но была одна проблема, нам приходилось делать приведение типов во 
время выполнения программы:

{lang="text"}
~~~~~~~~
  public String first(Collection collection) {
    return (String)(collection.get(0));
  }
~~~~~~~~

В ответ разработчики определяли объекты предметной области 
прямо в их бизнес-логике, которые фактически были `CollectionOfThings` 
(((Тут имеется в виду, что такие объекты представляли собой 
специфичную коллекцию, набор каких-то определенных значений, 
связанных с предметной областью))),
а Collection API становился деталью реализации. 

В 2005 году Java 5 добавила *generics*
(((Для этого слова нет нормального перевода.
Жаргонно говорят "дженерики", но официального слова имеено 
для generics нет. 
Есть понятие "обобщенное программирование", 
но оно не способно отражать смысл сущности как "generics")))
, что позволило нам определять `Collection<Thing>`, 
абстрагируясь над контейнером **и** его элементами. 
Generics изменили то, как мы пишем на Java. 

Автор компилятора для generics в Java, Мартин Одерски, затем создал
Scala с более сильной системой типов, неизменяемыми данными и 
множественным наследованием. Это повлекло слияние объекто-ориентированного
программирования (ООП) и функционального программирования (ФП). 

Для многих разработчиков ФП означает использование неизменяемых данных 
где только возможно, но изменяемое состояние программы до сих пор является 
неизбежным злом, которое должно быть изолировано и контролироваться, например 
с помощью акторов в Akka или `synchronized` классов. 
Такой стиль ФП порождает более простые программы, которые проще 
запускать параллельно и в распределнных системах, это является 
преимуществом над Java. Но это лишь малая часть всех преимуществ ФП, 
как мы узнаем по ходу этой книги.  

В Scala также появилась `Future`, что позволило упростить написание
асинхронных приложений. Но когда `Future` просачивается в возвращаемый тип, 
необходимо переписывать *все*, чтобы поддержать это, включая тесты, 
которые теперь подвержены произвольным задержкам.

У нас возникает проблема, схожая с Java 1.0: у нас нет способа абстрагироваться 
над выполнением программы, так же как у нас не было способа абсрагироваться над 
коллекциями. 

## Абстрагирование над выполнением программы

Скажем, мы хотим взаимодействовать с пользователем через интерфейс командной 
строки. Мы можем _читать_(`read`), что набирает пользователь, и мы можем
_писать_(`write`) сообщения ему. 

{lang="text"}
~~~~~~~~
  trait TerminalSync {
    def read(): String
    def write(t: String): Unit
  }
  
  trait TerminalAsync {
    def read(): Future[String]
    def write(t: String): Future[Unit]
  }
~~~~~~~~

Как нам написать такой обощенный код, который бы делал такую простую вещь 
как повторение введенных пользователем данных синхронно или асинхронно в 
зависимости от нашей реализации во время выполнения программы?

Мы могли бы написать синхронную версию и обернуть ее во `Future`, но
тогда бы нам пришлось заботиться о том, какой пул потоков мы должны 
использовать для работы, или мы могли бы использовать `Await.result` 
на этой `Future` и блокировать поток. В любом случае, получается очень 
шаблонный код, где мы работаем с фундаметально разными API, 
которые не унифицированы. 

Мы можем решить эту проблему, как в Java 1.2, при помощи общего предка, 
используя *типы высшего порядка* (higher kinded type, HKT) в языке Scala. 

A> **Типы Высшего Порядка** позволяют нам использовать *констуктор типа* 
A> в параметрах типа, что выглядит как `C[_]`. Это способ сказать, что
A> независимо от того, что такое `C`, оно должно принимать параметр типа. 
A> Например:
A>
A> {lang="text"}
A> ~~~~~~~~
A>   trait Foo[C[_]] {
A>     def create(i: Int): C[Int]
A>   }
A> ~~~~~~~~
A>
A> `List` является конструктором типа, потому что он принимает тип (например `Int`)  
A> и создает новый тип (`List[Int]`). Мы можем реализовать `Foo` использую `List`. 
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   object FooList extends Foo[List] {
A>     def create(i: Int): List[Int] = List(i)
A>   }
A> ~~~~~~~~
A> 
A> We can implement `Foo` for anything with a type parameter hole, e.g.
A> `Either[String, _]`. Unfortunately it is a bit clunky and we have to
A> create a type alias to trick the compiler into accepting it:
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   type EitherString[T] = Either[String, T]
A> ~~~~~~~~
A> 
A> Type aliases don't define new types, they just use substitution and
A> don't provide extra type safety. The compiler substitutes
A> `EitherString[T]` with `Either[String, T]` everywhere. This technique
A> can be used to trick the compiler into accepting types with one hole
A> when it would otherwise think there are two, like when we implement
A> `Foo` with `EitherString`:
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   object FooEitherString extends Foo[EitherString] {
A>     def create(i: Int): Either[String, Int] = Right(i)
A>   }
A> ~~~~~~~~
A> 
A> Alternatively, the [kind projector](https://github.com/non/kind-projector/) plugin allows us to avoid the `type`
A> alias and use `?` syntax to tell the compiler where the type hole is:
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   object FooEitherString extends Foo[Either[String, ?]] {
A>     def create(i: Int): Either[String, Int] = Right(i)
A>   }
A> ~~~~~~~~
A> 
A> Finally, there is this one weird trick we can use when we want to ignore the
A> type constructor. Define a type alias to be equal to its parameter:
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   type Id[T] = T
A> ~~~~~~~~
A> 
A> Before proceeding, understand that `Id[Int]` is the same thing as `Int`, by
A> substituting `Int` into `T`. Because `Id` is a valid type constructor we can use
A> `Id` in an implementation of `Foo`
A> 
A> {lang="text"}
A> ~~~~~~~~
A>   object FooId extends Foo[Id] {
A>     def create(i: Int): Int = i
A>   }
A> ~~~~~~~~

We want to define `Terminal` for a type constructor `C[_]`. By
defining `Now` to construct to its type parameter (like `Id`), we can
implement a common interface for synchronous and asynchronous
terminals:

{lang="text"}
~~~~~~~~
  trait Terminal[C[_]] {
    def read: C[String]
    def write(t: String): C[Unit]
  }
  
  type Now[X] = X
  
  object TerminalSync extends Terminal[Now] {
    def read: String = ???
    def write(t: String): Unit = ???
  }
  
  object TerminalAsync extends Terminal[Future] {
    def read: Future[String] = ???
    def write(t: String): Future[Unit] = ???
  }
~~~~~~~~

We can think of `C` as a *Context* because we say "in the context of
executing `Now`" or "in the `Future`".

But we know nothing about `C` and we cannot do anything with a
`C[String]`. What we need is a kind of execution environment that lets
us call a method returning `C[T]` and then be able to do something
with the `T`, including calling another method on `Terminal`. We also
need a way of wrapping a value as a `C[_]`. This signature works well:

{lang="text"}
~~~~~~~~
  trait Execution[C[_]] {
    def chain[A, B](c: C[A])(f: A => C[B]): C[B]
    def create[B](b: B): C[B]
  }
~~~~~~~~

letting us write:

{lang="text"}
~~~~~~~~
  def echo[C[_]](t: Terminal[C], e: Execution[C]): C[String] =
    e.chain(t.read) { in: String =>
      e.chain(t.write(in)) { _: Unit =>
        e.create(in)
      }
    }
~~~~~~~~

We can now share the `echo` implementation between synchronous and
asynchronous codepaths. We can write a mock implementation of
`Terminal[Now]` and use it in our tests without any timeouts.

Implementations of `Execution[Now]` and `Execution[Future]` are
reusable by generic methods like `echo`.

But the code for `echo` is horrible!

The `implicit class` Scala language feature gives `C` some methods.
We will call these methods `flatMap` and `map` for reasons that will
become clearer in a moment. Each method takes an `implicit
Execution[C]`, but this is nothing more than the `flatMap` and `map`
that we're used to on `Seq`, `Option` and `Future`

{lang="text"}
~~~~~~~~
  object Execution {
    implicit class Ops[A, C[_]](c: C[A]) {
      def flatMap[B](f: A => C[B])(implicit e: Execution[C]): C[B] =
            e.chain(c)(f)
      def map[B](f: A => B)(implicit e: Execution[C]): C[B] =
            e.chain(c)(f andThen e.create)
    }
  }
  
  def echo[C[_]](implicit t: Terminal[C], e: Execution[C]): C[String] =
    t.read.flatMap { in: String =>
      t.write(in).map { _: Unit =>
        in
      }
    }
~~~~~~~~

We can now reveal why we used `flatMap` as the method name: it lets us
use a *for comprehension*, which is just syntax sugar over nested
`flatMap` and `map`.

{lang="text"}
~~~~~~~~
  def echo[C[_]](implicit t: Terminal[C], e: Execution[C]): C[String] =
    for {
      in <- t.read
       _ <- t.write(in)
    } yield in
~~~~~~~~

Our `Execution` has the same signature as a trait in Scalaz called `Monad`,
except `chain` is `bind` and `create` is `pure`. We say that `C` is *monadic*
when there is an implicit `Monad[C]` available. In addition, Scalaz has the `Id`
type alias.

The takeaway is: if we write methods that operate on monadic types,
then we can write sequential code that abstracts over its execution
context. Here, we have shown an abstraction over synchronous and
asynchronous execution but it can also be for the purpose of more
rigorous error handling (where `C[_]` is `Either[Error, _]`), managing
access to volatile state, performing I/O, or auditing of the session.


## Pure Functional Programming

Functional Programming is the act of writing programs with *pure functions*.
Pure functions have three properties:

-   **Total**: return a value for every possible input
-   **Deterministic**: return the same value for the same input
-   **Inculpable**: no (direct) interaction with the world or program state.

Together, these properties give us an unprecedented ability to reason about our
code. For example, input validation is easier to isolate with totality, caching
is possible when functions are deterministic, and interacting with the world is
easier to control, and test, when functions are inculpable.

The kinds of things that break these properties are *side effects*: directly
accessing or changing mutable state (e.g. maintaining a `var` in a class or
using a legacy API that is impure), communicating with external resources (e.g.
files or network lookup), or throwing and catching exceptions.

We write pure functions by avoiding exceptions, and interacting with the world
only through a safe `F[_]` execution context.

In the previous section, we abstracted over execution and defined `echo[Id]` and
`echo[Future]`. We might reasonably expect that calling any `echo` will not
perform any side effects, because it is pure. However, if we use `Future` or
`Id` as the execution context, our application will start listening to stdin:

{lang="text"}
~~~~~~~~
  val futureEcho: Future[String] = echo[Future]
~~~~~~~~

We have broken purity and are no longer writing FP code: `futureEcho` is the
result of running `echo` once. `Future` conflates the definition of a program
with *interpreting* it (running it). As a result, applications built with
`Future` are difficult to reason about.

A> An expression is *referentially transparent* if it can be replaced with its
A> corresponding value without changing the program's behaviour.
A> 
A> Pure functions are referentially transparent, allowing for a great deal of code
A> reuse, performance optimisation, understanding, and control of a program.
A> 
A> Impure functions are not referentially transparent. We cannot replace
A> `echo[Future]` with a value, such as `val futureEcho`, since the pesky user can
A> type something different the second time.

We can define a simple safe `F[_]` execution context

{lang="text"}
~~~~~~~~
  final class IO[A](val interpret: () => A) {
    def map[B](f: A => B): IO[B] = IO(f(interpret()))
    def flatMap[B](f: A => IO[B]): IO[B] = IO(f(interpret()).interpret())
  }
  object IO {
    def apply[A](a: =>A): IO[A] = new IO(() => a)
  }
~~~~~~~~

which lazily evaluates a thunk. `IO` is just a data structure that references
(potentially) impure code, it isn't actually running anything. We can implement
`Terminal[IO]`

{lang="text"}
~~~~~~~~
  object TerminalIO extends Terminal[IO] {
    def read: IO[String]           = IO { io.StdIn.readLine }
    def write(t: String): IO[Unit] = IO { println(t) }
  }
~~~~~~~~

and call `echo[IO]` to get back a value

{lang="text"}
~~~~~~~~
  val delayed: IO[String] = echo[IO]
~~~~~~~~

This `val delayed` can be reused, it is just the definition of the work to be
done. We can map the `String` and compose additional programs, much as we would
map over a `Future`. `IO` keeps us honest that we are depending on some
interaction with the world, but does not prevent us from accessing the output of
that interaction.

The impure code inside the `IO` is only evaluated when we `.interpret()` the
value, which is an impure action

{lang="text"}
~~~~~~~~
  delayed.interpret()
~~~~~~~~

An application composed of `IO` programs is only interpreted once, in the `main`
method, which is also called *the end of the world*.

In this book, we expand on the concepts introduced in this chapter and show how
to write maintainable, pure functions, that achieve our business's objectives.


