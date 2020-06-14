---
title: Using interfaces in Go
date: 2020-03-21
author: riyaz-ali
background: https://source.unsplash.com/4m7gmLNr3M0
description: |
    This article aims to cover interfaces and their usage in Golang. 
    It tries to explain where and how you should interfaces and how
    to avoid common _pitfalls_ while using them in Golang.
tags:
- Golang
- Interfaces
- HowTo
keywords:
- Golang
- Interfaces
---

A lot have changed in the past few months!

I've started a new job @ [CloudCover](https://cldcvr.com/) and have also started [contributing](https://github.com/cavaliercoder/grab/pull/67) [to](https://github.com/riyaz-ali/shhh) golang projects!

The inspiration for this post comes from a (slightly) confusing topic in golang which I initially struggled to understand, and that's **interfaces**!

This post **doesn't** cover what interfaces are! There are lots of [really](https://www.callicoder.com/golang-interfaces/) [good](https://medium.com/rungo/interfaces-in-go-ab1601159b3a) [articles](https://www.alexedwards.net/blog/interfaces-explained) on the internet for that.

What I really want to focus on is "how" you can use them and what common _pitfalls_ to avoid (especially if you come from Java / C#)

-------------

## <small>#1</small> Define interfaces close to site of use

This allows the code (that makes use of the interface) to depend on abstractions rather than on concrete implementations.

And because [interfaces in go are implemented implicitly](https://tour.golang.org/methods/10), your actual implementation would just work auto-magically!

```golang
// UserService defines a service that's responsible to manage user profiles
type UserService interface {
    // FetchAll returns a cursor that lists all users on the platform
    FetchAll() UserIterator
    
    // ......
    // this interface could contains more (but related) methods
    // just make sure to not bloat it (see Rule #3)
}

// Now we could utilize UserService abstraction in our code.
// Here we pass it to a function that returns an http.HandlerFunc
// When that handler is executed, the code could utilize the supplied user service
// without having to worry about where the actual implementation came from

func ListAllUser(svc UserService) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var cursor = svc.FetchAll()
        //...
    }
}
```

This allows you to decouple your code and it's dependencies enabling you to write modular code.

**DO NOT** place the interface in the package that provides the dependency and than make the provided service _implement_ that interface. This seems like a common pattern used in other languages (namely, Java) where you'd declare an interface `com.example.UserService` and than provide a package-private implementation of it `com.example.UserServiceImpl` (along with a factory perhaps)

The _real power_ of Go interfaces comes from the fact that they can be implemented implicitly! This reduces the overhead of _arranging_ for an implementation before-hand and also allows a single type to implement multiple (but un-related) interfaces or implement interfaces that doesn't even exist yet (crazy right!)

A common example from the standard library to support this would be the `fmt.Stringer` interface which is defined close to it's site of use (the methods in `fmt` package) yet anyone can provide an implementation by adding a method with compatible signature `String() string`

## <small>#2</small> If it's gonna be used at multiple places, declare it only once

If you feel like your interface definition is going to be used in multiple places, it's better to declare it once (and maybe in a shared package)

Wait a minute.. doesn't that contradict #1 ?

Umm no, not really. You need to declare them in a single place and make the symbol (the interface) available to your _site of use_. This doesn't mean you need to couple it with your actual implementation.

A notable project which makes use of this pattern is [drone.io](https://github.com/drone/drone)

All the _core_ interfaces used by drone are declared under the [`core`](https://github.com/drone/drone/tree/d1a2f2174db51f4011ec4ab212ad4704c71c751e/core) package which is than used by various other packages and satisfied by another set of packages.

## <small>#3</small> Keep it small silly 😛

Try and stick with [SOLID](https://en.wikipedia.org/wiki/SOLID)! 

More specifically one should try to follow [Interface Segregation](https://en.wikipedia.org/wiki/Interface_segregation_principle) principle.

Quoting from Wikipedia,

> The **Interface Segregation** principle states that no client should be forced to depend on methods it does not use.

One way to achieve this would be to keep your interfaces _lean_ and focused. Model your interfaces around _behaviour_, and not data, and prefer [embedding](https://golang.org/doc/effective_go.html#embedding) over large interfaces.

A very popular example from the standard library would be the interfaces in [`io`](https://github.com/golang/go/tree/release-branch.go1.13/src/io) package

```golang
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

type ReadCloser interface {
    Reader
    Closer
}
```

## <small>#4</small> Accept interfaces return structs

Lol no, I didn't come up with it 😅 in fact, [no one's actually sure who did](https://redd.it/dfe1qr) 🙄 but people often cite [Postel's Law](https://en.wikipedia.org/wiki/Robustness_principle)

The basic idea is that your method should accept (as arguments) values of interface types while _should_ return values which are concrete types.

A few examples from the standard library would be

-  [`io.Pipe`](https://pkg.go.dev/io?tab=doc#Pipe) method whose signature declares the concrete underlying type of the return values
    ```golang
    func Pipe() (*PipeReader, *PipeWriter)
    ```

- [`io.Copy`](https://pkg.go.dev/io?tab=doc#Copy) method whose signature declares accepted argument types to be interfaces
    ```golang
    func Copy(dst Writer, src Reader) (written int64, err error)
    ```

The primary advantage here is that your function is now decoupled and depends on _behaviour_ rather than concrete types of the arguments. 

Yet consumers of your function have the flexibility to choose the way they want to use the return type (by casting it appropriately) based on their use case.

This doesn't mean that you should (or can) always declare methods like that. There are exceptions. For example, [`io.TeeReader`](https://pkg.go.dev/io?tab=doc#TeeReader) method

```golang
func TeeReader(r Reader, w Writer) Reader
```

Here, it actually makes sense to wrap the underlying implementation and return an `io.Reader`

Another _common_ use case for returning interface types from method is to _ensure_ that the value being returned _actually_ satisfies the interface. I think this is rather a hack around Go's philosophy that "interfaces are implemented implicitly". 

If what you actually need is a way to ensure that your custom implementation satisfies an interface you can use _interface guards_.

----------

## Bonus

### Interface guards

An interface guard (not an official term) is a simple no-cost hack to add checks in your code to ensure that your implementation satisfies some interface. Use this if you are building a custom implementation for an interface that is exported by a third-party library.

Following snippet is taken from [Caddy's documentation on interface guards](https://caddyserver.com/docs/extending-caddy#interface-guards)

```golang
var _ InterfaceName = (*YourType)(nil)
```

This is simple yet no-cost way of ensuring `*YourType` _implemements_ `InterfaceName` at compile-time rather than running into errors at runtime.

----------

Got doubts / queries / suggestions, feel free to [open an issue on Github](https://github.com/riyaz-ali/riyaz-ali.github.io/issues/new)!