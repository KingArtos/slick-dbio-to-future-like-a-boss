# slick-dbio-to-future-like-a-boss
If you have problems to work with transaction and repositories

### Slick and transactions
Considering that we work with slick and [As soon as you convert DBIO to Future you loose capability of bundling actions into single transaction](https://stackoverflow.com/questions/41219200/how-to-use-transaction-in-slick/41219465#41219465)

We need a DBIO repository to allow composition and run actions in single transaction when necessary.

#### Currently

We have one approach where we create two repositories, one to dbio and other to future.
```scala
trait RepoAlgebra[F[_]] {
  def find(id: Int): F[Option[Record]]
}

class DBIORepo extends RepoAlgebra[DBIO] {
  def find(id: Int): DBIO[Option[Record]] = ???
}

class FutureRepo(dbioRepo: RepoAlgebra[DBIO]) extends RepoAlgebra[Future] {
  def find(id: Int): Future[Option[Record]] = db.run(dbioRepo.find(id))
}
```
So we have to change 3 files when add a new function.

#### Possible Solution

After some research and a lot of tests, i found finally a possible solution to avoid this boilerplate with minimal changes in current code: [Implicit Conversion](https://docs.scala-lang.org/tour/implicit-conversions.html) + [Natural transformation](https://typelevel.org/cats/datatypes/functionk.html) +
[tagless final](https://jproyo.github.io/posts/2019-02-07-practical-tagless-final-in-scala.html)

This code bellow implements a `FunctionK`, it's where we say how a `DBIO` will convert to a `Future`
```scala
package object dbioOps extends DatabaseComponent {
  object Converters {
    implicit val future: FunctionK[DBIO, Future] = new FunctionK[DBIO, Future]{
      override def apply[A](fa: DBIO[A]): Future[A] = db.run(fa)
    }
  }
}
```
This code allow functions to transform G in F
>  G = Your repository monad
>  F = Your service monad
```scala
package object serviceHelper {
  implicit def asF[G[_], F[_], T](query: G[T])(implicit f: FunctionK[G, F]): F[T] = f(query)

  implicit class AsFOps [G[_], F[_], T](query: G[T])(implicit f: FunctionK[G, F]) {
    def asF: F[T] = f(query)
  }
}
```
And in our services:
- Concrete services(fixed monad):
```scala
import com.gympass.booking.utils.dbioOps.Converters.future

FutureService(repo: RepoAlgebra[DBIO])(implicit toFuture: FunctionK[DBIO, Future]) {
  def find(id: Int): Future[Option[Record]] = toFuture(repo.find(id))

  def otherFind(id: Int): Future[ErrorOr[Record]] = toFuture(repo.find(id)).flatMap(...)
}
```
- tagless final explicit mode(without serviceHelper):
```scala
object Service {
  import com.gympass.booking.utils.dbioOps.Converters.future
  def withFutureEffect: ServiceAlgebra[Future] =
    new MyService[Future, DBIO](Repo.withDBIOEffect())
}

trait ServiceAlgebra[F[_]] {
  def find(id: Int): F[Option[Record]]
  def otherFind(id: Int): F[ErrorOr[Record]]
}

class MyService[F[_] : Monad, G[_]](repo: RepoAlgebra[G])(implicit toF: FunctionK[G, F])
  extends ServiceAlgebra[F] {
  def find(id: Int): F[Option[Record]] = toF(repo.find(id))

  def otherFind(id: Int): F[ErrorOr[Record]] = toF(repo.find(id)).flatMap(...)
}
```
- tagless final smooth mode(using ServiceHelper):
```scala
object Service {
  import com.gympass.booking.utils.dbioOps.Converters.future
  def withFutureEffect: ServiceAlgebra[Future] =
    new MyService[Future, DBIO](Repo.withDBIOEffect())
}

trait ServiceAlgebra[F[_]] {
  def find(id: Int): F[Option[Record]]
  def otherFind(id: Int): F[ErrorOr[Record]]
}

class MyService[F[_] : Monad, G[_]](repo: RepoAlgebra[G])(implicit toF: FunctionK[G, F])
  extends ServiceAlgebra[F] {
  import com.gympass.booking.utils.serviceHelper._
  def find(id: Int): F[Option[Record]] = repo.find(id)

  def otherFind(id: Int): F[ErrorOr[Record]] = repo.find(id).asF.flatMap(...)
}
```
- tagless final explicit mode(using ServiceHelper):
```scala
object Service {
  import com.gympass.booking.utils.dbioOps.Converters.future
  def withFutureEffect: ServiceAlgebra[Future] =
    new MyService[Future, DBIO](Repo.withDBIOEffect())
}

trait ServiceAlgebra[F[_]] {
  def find(id: Int): F[Option[Record]]
  def otherFind(id: Int): F[ErrorOr[Record]]
}

class MyService[F[_] : Monad, G[_]](repo: RepoAlgebra[G])(implicit toF: FunctionK[G, F])
  extends ServiceAlgebra[F] {
  import com.gympass.booking.utils.serviceHelper.AsFOps
  def find(id: Int): F[Option[Record]] = repo.find(id).asF

  def otherFind(id: Int): F[ErrorOr[Record]] = repo.find(id).asF.flatMap(...)
}
```

And transactions continue as is today
```scala
FutureService(repo: RepoAlgebra[DBIO]) extends TransactionSupport {
  def increment(id: Int): Future[Option[Record]] =
    transactionally(
      for {
        ...
      } yield x
    )
}
```
#### Other links
- [Presentation about slick](http://virtuslab-team.slides.com/pdolega/slick-101#/85)
- [comparing-scala-relational-database-access-libraries](https://softwaremill.com/comparing-scala-relational-database-access-libraries/)
- [context bound and implicit](https://gvolpe.github.io/blog/context-bound-vs-implicit-evidence/)
- [context bound](https://docs.scala-lang.org/tutorials/FAQ/context-bounds.html)
