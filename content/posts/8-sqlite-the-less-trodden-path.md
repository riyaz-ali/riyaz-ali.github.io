---
title: "The less trodden path with sqlite"
date: 2021-03-27T02:00:00Z
author: riyaz-ali
background: https://source.unsplash.com/4m7gmLNr3M0
description: |
    In this article, I highlight exciting set of features and interfaces 
    that allows you to unlock the full potential of sqlite3
aliases:
- /sqlite-the-less-trodden-path.html
tags:
- Software Engineering 
- SQLite
keywords:
- Software Engineering 
- SQLite
---

Lately, I've been tinkering around a bit with `sqlite3`, and this post contains a brief of some exciting set of features and interfaces that I've re-discovered during the process.

---

`sqlite3` isn't exactly new. It's been around for almost two decades now (much longer than I've known coding myself 😛). There's plethora of articles on the internet describing [what it is](https://www.sqlite.org/about.html), what makes it special, [when you should ideally use it](https://www.sqlite.org/whentouse.html) (and when not) and why _["it's the only database you will ever need"](https://unixsheikh.com/articles/sqlite-the-only-database-you-will-ever-need-in-most-cases.html)._

So, I'm not gonna repeat all those advices here. Instead I'll try to cover some of other exciting, specialised interfaces that `sqlite3` offers and how we can profit from it!

---

### #1 Custom functions in SQL

`sqlite3` provides support for registering [custom, application-defined functions](https://www.sqlite.org/appfunc.html) in C that can be exposed to queries in SQL.

Custom functions can be used to integrate your queries with the rest of the system, or provide access to external functionality. They can be used to perform complex calculation or do statistical operations on data.

Custom functions can be used to marshal / un-marshal and / or manipulate data in alternative formats (such as `json`). They can be used to compute hashes and do much more!

`sqlite3` comes with a bunch of built-in [scalar](https://www.sqlite.org/lang_corefunc.html) and [aggregate](https://www.sqlite.org/lang_aggfunc.html) functions, and with [routines to manipulate date / time information](https://www.sqlite.org/lang_datefunc.html). There is also [nalgeon/sqlean](https://github.com/nalgeon/sqlean) on Github that aims to pack a lot more set of functions into convenient extensions that can be loaded on-demand.

### #2 Virtual Tables

[Virtual Tables](http://sqlite.org/vtab.html) are one of the most exciting features of `sqlite3` that I found! Essentially, virtual table allows us to register objects with `sqlite3` that, from the perspective of an SQL statement, looks like any other table / view.

This allows the user to be able to query them just as they would do with any other table / view. When a user executes a query against a virtual table `sqlite3` invokes callback methods on the virtual table object.

This has the potential to unlock a ton of different use-cases. For eg., there's an [official](https://www.sqlite.org/csv.html) [`csv`](https://www.sqlite.org/csv.html) [virtual table implementation](https://www.sqlite.org/csv.html) that allow users to query data from a `csv` file just as they would with a normal table. There are other implementations that make it possible to interface with other systems, such as [augmentable-dev/askgit](https://github.com/augmentable-dev/askgit) uses `sqlite3` virtual tables to allow users to query data from `git` repositories, [osquery.io](https://osquery.io) that allow users to run queries against the operating systems!

### #3 Authorizer to check / limit what a user can query

`sqlite3` supports registering [a custom authorizer function](https://www.sqlite.org/c3ref/set_authorizer.html) that can limit what data a query can access and what action can it perform. This, together with [other defensive settings](https://www.sqlite.org/security.html), can be used to turn `sqlite3` into a sandbox where we can execute raw, user-supplied, untrusted statements _with confidence_ that it won't cause any un-intended _side-effects._

### #4 Capture a change-set of differences in a session

[`sqlite3` session extension](https://www.sqlite.org/sessionintro.html) provides a mechanism for recording changes to some or all of the _rowid tables_ in an SQLite database, and packaging those changes into a "changeset" file that can later be used to apply the same set of changes to another database. This can be used to capture _diffs_ generated by multiple users, working asynchronously, capturing and merging it together into a unified database file.

Sessions can be useful when `sqlite3` is used as an _application data format_ __ and multiple users work on the same file at the same time. Each user can generate a patch for their side of work (much like how one would do it with a vcs like `git`) and the system could combine those into a unified database file.

### #5 Online Backup API

[`sqlite3`Online Backup API](https://www.sqlite.org/backup.html) allows the contents of one database to be copied into another database, in an incremental fashion where the database is only locked for the brief periods of time when it is actually being read from. This allows other database users to continue uninterrupted while a backup of an online database is made.

It provides a basic building block onto which other, more complex replication solutions can be built.

### #6 Dynamically loaded extensions

`sqlite3` has [support for building extensions](https://www.sqlite.org/loadext.html) that can either be loaded dynamically (as a shared object file) or linked statically into the final build.

The [`sqlite3ext.h`](https://sqlite.org/src/file?name=src/sqlite3ext.h&ci=trunk) defines the Extension API which the module can use to interact with the database connection (that loaded the extension). The extension can do pretty much anything that you could normally do with the `sqlite3` core, like registering custom functions, defining virtual table modules and more!

One should consider building re-usable modules as loadable extensions to allow for easier distribution and better re-use.

---

In the follow-up articles, I would try to cover some of the features I've discussed here and how one can leverage those while coding with Golang using [http://go.riyazali.net/sqlite](http://go.riyazali.net/sqlite), a library that I recently open-sourced 🤗
