# Eff

Imagine you are working with an application that contains the following effects:
* Future
* Reader
* Either

The following function demonstrates this. Given a `DBConfig`, the function makes an async database call that may fail with a `SQLException` or succeed with a `User`:
```
def getUser(userId: Int): Reader[DBConfig, Future[Either[SQLException, User]] = ...
```

This function is called by the following function:

```
def getUserFirstName(userId: Int): Reader[DBConfig, Future[Either[SQLException, UserFirstName]] = {
  val user = getUser(userId)
  user.map(_.map(_.map(_.firstName)))
}
```

The problem here is that you have nested monads, which have to be mapped over several times. Introducing Eff eliminates this problem.

```
type ReadDBConfig[R] = MemberIn[Reader[DBConfig, ?], R]
type Err[R] = MemberIn[Either[SQLException, ?], R]
// _Async is an out of the box effect
def getUser[R: ReadDBConfig : _Async : Err](userId: Int): Eff[R, User] = ...
```

The code above describes a `getUser` function that has `Reader`, `Future` and `Either` effects. The return type of the function implies that the end result of the effectful stack is a `User`. 
Now `getUserFirstName` is much simpler to implement. `map`ping a function `A => B` over an `Eff[R, A]` gives you a `Eff[R, B]`. 

``
def getUserFirstName(userId: Int): Eff[R, UserFirstName] = {
  val user = getUser(userId)
  user map (_.firstName)
}
```
