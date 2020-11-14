---
layout: post
title:  "Better error handling with monads"
permalink: /2020/11/14/better-error-handling-with-monads
tags: functional-programming typescript
---

Functional programming seems to be gaining quite a bit of tractions in the
javascript community in recent years, but I haven't been using many of the
techniques in everyday programming except the well-known notions of pure
functions and immutability.

## Functional error handling with Either

Recently I came across [this post][khalilstemmler] from Khalil Stemmler's blog
while looking for some real life [Domain Driven Design][ddd] examples. It
makes use of the `Either` type to validate inputs like this:

{% highlight typescript %}
function createUser1(name: string, email: string, password: string): Either<E, User> {
  const userName: Either<E, UserName> = UserName.parse(name);
  if (userName.isLeft()) {
    return left(userName.value);
  }

  const userEmail: Either<E, UserEmail> = UserEmail.parse(email);
  if (userEmail.isLeft()) {
    return left(userEmail.value);
  }

  const userPassword: Either<E, UserPassword> = UserPassword.parse(password);
  if (userPassword.isLeft()) {
    return left(userPassword.value);
  }

  return Right.of(new User(userName.value, userEmail.value, userPassword.value));
}
{% endhighlight %}

If you are unfamiliar with the `Either` type, `Either<E, A>` is simply a
disjoint union of `Left<E, A>` and `Right<E, A>`. In typescript:

{% highlight typescript %}
type Either<E, A> = Left<E, A> | Right<E, A>;
{% endhighlight %}

Conventionally, `Left<E>` is used for errors and `Right<A>` contains
successful computation results. The freely available ebook [Professor
Frisby's Mostly Adequate Guide to Functional
Programming][mostly-adequate-guide] provides an excellent introduction to
`Either` and many more functional programming concepts.

The usage of `Either` here is interesting in that it forces the client to
handle the error through typescript's type system. It is certainly an
improvement to the more traditional `try {...} catch (e) {...}` way which is
easily forgotten and clutters the program text with syntax noise.

## Either as functor

However, I believe the above `createUser` snippet can be further improved by
taking advantage of more interesting properties of `Either`. First, let's
take a look of how `Either` behaves as a functor.

{% highlight typescript %}
let two: Either<string, number> = new Right(1).map(double);      // Right(2)
let err: Either<string, number> = new Left("BOOM!").map(double); // Left("BOOM!")
{% endhighlight %}

In other words, the `Left` type actually shortcircuits all further
computations once an error like "BOOM!" occurs. This is a very powerful
property that can help us remove all those `isLeft` checking in the previous
code, making the core program logic much more prominent.

There is one issue here that prevents us to just drop `map` into our
`createUser` function: our `parse` functions are returning `Either`s instead
of plain value objects. This means that we will get a `Either<E, Either<E,
Either<E, User>>>` at the end, not the desired `Either<E, User>` as before:

{% highlight typescript %}
function createUser2(name: string, email: string, password: string): Either<E, Either<E, Either<E, User>>> {
  const userName: Either<E, UserName> = UserName.parse(name);

  const userEmail: Either<E, Either<E, UserEmail>> = 
    userName.map(() => UserEmail.parse(email));

  const userPassword: Either<E, Either<E, Either<E, UserPassword>>> =
    userEmail.map((inner) => inner.map(() => UserPassword.parse(password)));

  return userPassword.map(x => x.map(
    y => y.map(
      () => new User(
        userName.value as UserName,
        (userEmail.value as Either<E, UserEmail>).value as UserEmail,
        ((userPassword.value as Either<E, Either<E, UserPassword>>).value as Either<E, UserPassword>).value as UserPassword
      )
    )
  ));
}
{% endhighlight %}

What a hassle! In addition to the nested Either, we also need to add lots of
type casting as typescript doesn't know that only `Right<E, A>` would run the
map function. We can do better.

## Monad to the rescue

`Monad` provides a `chain` function that can deal with this exact issue:

> chain :: Monad m => (a => m b) -> m a -> m b

Using `chain`, we can flatten all the nested `Either` as follow:

{% highlight typescript %}
function createUser(name: string, email: string, password: string): Either<E, User> {
  const userName: Either<E, UserName> = UserName.parse(name);
  const userEmail: Either<E, UserEmail> = userName.chain(() => UserEmail.parse(email));
  const userPassword: Either<E, UserPassword> = userEmail.chain(() => UserPassword.parse(password));
  return userPassword.map(() => new User(userName.value, userEmail.value, userPassword.value));
}
{% endhighlight %}

This looks much better!

## Replacing variable assignments with closure

The monad version should be good enough in many practical situations, and it
fits very nicely with any surrounding object-oriented code. But if we want to
be even more pure in the functional world, we can replace all the variable
assignments with closure like this:

{% highlight typescript %}
function createUser(name: string, email: string, password: string): Either<E, User> {
  return UserName.parse(name).chain((userName: UserName) =>
      UserEmail.parse(email).chain((userEmail: UserEmail) =>
        UserPassword.parse(password).map((userPassword: UserPassword) =>
          new User(userName, userEmail, userPassword))));
}
{% endhighlight %}

This final version is very satisfying to read. Not only does it eliminates
all the explicit type casting, but it also makes the sequential parsing logic
explicit while hiding away all the error handling code in a type safe manner.

The full code samples can be downloaded at [here][code].

[khalilstemmler]: https://khalilstemmler.com/articles/enterprise-typescript-nodejs/functional-error-handling/
[ddd]: https://www.amazon.co.uk/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215
[mostly-adequate-guide]: https://mostly-adequate.gitbooks.io/mostly-adequate-guide/content/ch08.html
[code]: https://github.com/ksmai/functional-error-handling-demo/blob/master/index.ts
