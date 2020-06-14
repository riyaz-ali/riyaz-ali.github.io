---
title: Why documentation matters
date: 2019-03-14
author: riyaz-ali
background: https://source.unsplash.com/3ym6i13Y9LU
description: |
    In this article, I've tried to cover an often ignored aspect of 
    software engineering, writing good documentation.
aliases:
- /why-documentation-matters.html
tags:
- SoftwareEngineering
- Documentation
keywords:
- SoftwareEngineering
- Documentation
---

### tl;dr

> Good documentation gives the developers the necessary control to maintain a system - [nochance](https://softwareengineering.stackexchange.com/users/34148/nochance) @ stackexchange

## But good code doesn't need documentation 🤔

Remember, documentation go way beyond the comments that are inserted in code. Often times, they're a combination of [well describing code comments](http://antirez.com/news/124), [design documents](/design-docs.html), business requirement documents and other relevant pieces.

## Ain’t nobody got time for that ⏰

The main reason code goes undocumented is because of _time_!

Developers often argue that they could rather be _more productive_ and be writing code than spending time documenting the software.

## The case for documentation

No matter what you're building, chances are that some day you (or somebody else) would need to re-visit it, to fix some bug or extend it to add new feature or refactor it or any other reason.. and when that day comes, you will not remember so vividly what you wrote and why.

And if you do remember, there may be some edge cases or specific uses which may not be clearly apparent. The obvious solution is **documentation**.

And after all, when we, as developers, need to understand something about certain aspect of coding / library / framework, what do we do?

We go look at the documentation!

## But [aren't comments a code smell](https://softwareengineering.stackexchange.com/questions/1) 🤷‍♂️

Umm... <span style="text-decoration: underline">only if the comment describes _what the code is doing_</span> <sup>[[1]](https://softwareengineering.stackexchange.com/a/13)</sup>

```java
static int add(int a, int b) {
  // add b to a and return the result
  return a + b;
}
```

👆 that's  🤷‍♂️

But have a look at following snippet from [Spring Data Commons' `CrudRepository.java`](https://git.io/JeZrK)

```java
/**
 * Interface for generic CRUD operations on a repository for a specific type.
 *
 * @author Oliver Gierke
 * @author Eberhard Wolff
 */
@NoRepositoryBean
public interface CrudRepository<T, ID> extends Repository<T, ID> {
  
  /**
   * Saves a given entity. Use the returned instance for further operations as the save operation might have changed the
   * entity instance completely.
   *
   * @param entity must not be {@literal null}.
   * @return the saved entity; will never be {@literal null}.
   */
  <S extends T> S save(S entity);

  // other definitions omitted..
}
```

The comments here describe _why_ the interface is there and _what_ the function does (not how it does it!)

This makes it very clear for any future reader to know about the purpose of the interface and what the function is supposed to do.

This instance is just one example! There are several other places where comments can be very useful.

[@antirez](https://twitter.com/antirez), creator of Redis, had put together an [excellent blog about comments](http://antirez.com/news/124)! Make sure you check it out.

----------

👆only covers code comments / documentation.. what about other forms of documentation?

They are all important and are there to serve different use cases!

- [Design docs](/design-docs.html) are there to describe and postulate the approach a project took to solve a given problem
- System documentation represents documents that describe the system itself and its parts.
- User documentation covers manuals that are mainly prepared for end-users of the product and system administrators. User documentation includes tutorials, user guides, troubleshooting manuals, installation, and reference manuals.

[MDN](https://developer.mozilla.org/en-US/docs/Web/Guide), [Spring](https://docs.spring.io/spring/docs/current/spring-framework-reference), [Django](https://docs.djangoproject.com/en/2.2/), [Stripe](https://stripe.com/docs) are all good examples of products / tools / services with an excellent documentation.

## References

- [Why should you document code?](https://softwareengineering.stackexchange.com/questions/121775)
- [This Answer](https://softwareengineering.stackexchange.com/a/13) to ["comments are code smell"](https://softwareengineering.stackexchange.com/q/1)
- [Writing system software: code comments](http://antirez.com/news/124) by [@antirez](https://twitter.com/antirez)
- [Why documentation matters, and why you should include it in your code](https://freecodecamp.org/news/41ef62dd5c2f) by [Tomer Ben Rachel](https://github.com/TomerPacific)
