#Go database/sql教程

Go中使用SQL或者类似SQL、数据库的常用方法是[database/sql package](http://golang.org/pkg/database/sql/)。它对基于行的数据库提供了一个轻量级的接口。

为什么需要这样一个教程呢？因为包的文档虽然告诉你每一块内容是做什么的，却没有告诉你如何去使用这个包。我们都希望能有一个快速参考或者入门教程告诉我们怎么做而不是要点的罗列。欢迎贡献您的力量，请发送pull requests到[这里](https://github.com/VividCortex/go-database-sql-tutorial)

##概述

为了使用Go来访问数据库，你需要使用`sql.DB`。你使用这个类型来创建语句、事务、执行查询和获得结果。

你应该知道的第一件事就是**`sql.DB`不是数据库连接**。它也不是对某个特定数据库或者schema的映射。它是一个抽象接口和数据库的一种存在形式。这个数据库可以是本地文件，通过网络的连接、或者是存在与内存和进程之中。

`sql.DB`在下列情形下执行非常重要的任务：

*它用驱动打开和关闭对实际底层数据库的连接。
*它管理着一个连接池，连接可以是上面提到的多种形式。

`sql.DB`抽象的作用是让你不用考虑如何去管理并发访问底层数据库。当使用连接来执行任务时，连接被标记为在用状态，如果它不再被使用则由回到了可用连接池。这样造成的一个结果就是**如果释放连接失败，则会导致`db.SQL`打开很多连接**，从而俄仍导致耗尽所有资源(太多的连接，太多打开的文件，缺少可用的网络接口等)。后面会进一步讨论。

当创建过`sql.DB`之后，你就可以在这个基础上去查询、创建语句和事务了。

##导入数据库驱动

为了使用`database/sql`包，你不仅需要这个包本身，还需要你想用的特定数据库的驱动。

通常情况下不需要直接使用驱动包，虽然有些驱动鼓励你这么做（在我看来，这是个不好的想法）。相反的，你的代码应该尽可能的只引用`database/sql`中的定义的类型。这可以避免你的代码依赖于驱动，这样你就可以只需改动少部分代码就更换底层驱动和使用的底层数据。它还尽可能的强制你使用Go的惯用法而不是特定驱动的作者提供的ad-hoc惯用法。

在这个文档中，我们将使用@julienschmidt 和 @arnehormann 带来的优秀的[MySQL驱动](https://github.com/go-sql-driver/mysql)作为例子。

将下面代码添加导Go源码的顶部：

	import (
		"database/sql"
		_ "github.com/go-sql-driver/mysql"
	)

注意，我们使用`_`别名来匿名导入驱动，所以驱动的导出名字不会在我们的代码中可见。在后台，驱动将自己注册给`database/sql`包，但是，一般而言不会有别的事情发生。

现在，你已经准备好了如何访问数据库了

##访问数据

既然你已经加载了驱动包，那你就准备好了创建一个数据库对象，即`sql.DB`。

为了创建`sql.DB`，你需要使用`sql.Open()`。它返回一个`*sql.DB`：

	func main() {
		db, err := sql.Open("mysql",
			"user:password@tcp(127.0.0.1:3306)/hello")
		if err != nil {
			log.Fatal(err)
		}
		defer db.Close()
	}

下面时对这个例子的一些说明：

1. `sql.Open`的第一个参数是驱动名称。这是驱动注册自己到`database/sql`所用到的字符串，为了避免混淆，一般与包名相同。
2. 第二个参数是依赖于特定驱动的语法，用来告诉驱动如何访问底层数据库。在这个例子中，我们将要连接本地MySQL服务器的"hello"数据库。
3. 你应当（绝大部分情况下）总是检查并处理`database/sql`所有操作所返回的错误。我们后面将讨论一些不需要这么做的情况。
4. 一般而言，如果`sql.DB`的生命期不会超过函数的范围，那应该添加`defer db.Close()`

与直觉相反，`sql.Open()`**并不建立起任何到数据库的连接**，也不验证驱动连接的参数。相反，它仅仅准备好数据库抽象以备后用。第一个实际到底层数据库的连接会被延迟到第一次需要时才建立。如果你想立刻检查数据库是否可用(比如检查网络连接的建立与登录)，就应当使用`db.Ping()`，并且记住检查错误。

	err = db.Ping()
	if err != nil {
		// do something here
	}

虽然习惯上是在使用完数据库之后就`Close()`掉数据库。**但是，`sql.DB`对象是为了长连接而设计的**，不要过于频繁的`Open()`和`Close()`数据库。相反的，为每个需要访问的数据库创建**一个**`sql.DB`，并且，在程序访问完数据库之前一直保留它。在需要时传递它或者让它变为全局可用，而且是打开的。不要在短暂的函数中使用`Open()`和`Close()`，而是将`sql.DB`作为一个参数传递给短暂的函数。

如果你不把`sql.DB`作为一个长存的对象，那么你会遇到各种各样的错误，比如无法复用和共享连接，耗尽网络资源，由于TCP连接保持在`TIME_WAIT`状态而间断性的失败。诸如此类的问题表示着你并没有按照`database\sql`设计的意图来使用它。

Now it's time to use your `sql.DB` object.
现在是时候使用`sql.DB`对象了。

##获取结果集

从数据库中获取结果有几种常用的操作：

1. 执行一个返回行的查询
2. 准备一个供多次使用的语句，多次执行它，然后再销毁它。
3. 执行一个一次性使用的语句
4. 执行一个返回仅返回一行的查询。这是某些情况的快捷方式

Go的`database/sql`函数名字非常重要。**如果函数名字包含`Query`，则它被设计用来向数据库提出一个问题，并且返回一个行的集合**，即使它是空的。哪些不需要返回行的语句不应该使用`Query`函数，而应该使用`Exec()`。

###从数据库获取数据

让我们看一个如何查询数据库并且处理结果的例子。我们将查询用户表中`id`为1的用户，并且打印出用户的`id`和`name`。并且将结果赋值给变量，使用`rows.Scan()`，一次一行。

	var (
		id int
		name string
	)
	rows, err := db.Query("select id, name from users where id = ?", 1)
	if err != nil {
		log.Fatal(err)
	}
	defer rows.Close()
	for rows.Next() {
		err := rows.Scan(&id, &name)
		if err != nil {
			log.Fatal(err)
		}
		log.Println(id, name)
	}
	err = rows.Err()
	if err != nil {
		log.Fatal(err)
	}

Here's what's happening in the above code:
下面是关于代码的一些说明

1. 我们使用`db.Query()`来发送查询到数据库。像往常一样，检查错误。
2. 使用defer `rows.Close()`.这一步很重要。
3. 我们使用`rows.Next()`来遍历行。
4. 我们使用`rows.Scan()`来读取每行中的列到变量中。
5. 在遍历过所有行之后要检查错误。

这里有部分很容易出错，并且会导致不好的结果。

首先，只要有一个打卡的结果集(`rows`)，底层的connection就为忙，并且不能被其他查询所用。这意味着在连接池中不能使用它。如果你用`rows.Next()`遍历所有的行，最后你会遍历到最后一行，然后就会在内部遇到的EOF错误，然后为你调用`rows.Close()`。但是如果出于任何原因你退出了循环-错误，过早的返回等等，那么`rows`就不会被关闭，并且connection保持着为打开状态。这很容易耗尽资源。这就是为什么**你应该总是`defer rows.Close()`**，及时你也在循环的后面显示调用它，这么做也不是一个坏主意。`rows.Close()`调用不会有坏的影响，即使它已经被关闭过了，所以你可以多次调用它。注意，为了避免`runtime panic`，我们要先检查错误，然后在没有错误时再做`rows.Close()`。

其次，你应该在`for rows.Next()`循环结束后检查错误。如果循环中有错误，你需要知道到底出了什么错。不要假设循环会一直遍历完所有的行。

在数据库的所有操作中，一般都是捕获并检查错误。但`rows.Close()`返回的错误是唯一的一个例外。如果`rows.Close()`跑出错误，并不能明确说明正确的做法。记录错误消息或者引发panic或许是唯一有意义的方法，如果这也没有意义，你可以选择忽略这个错误。

This is pretty much the only way to do it in Go. You can't
get a row as a map, for example. That's because everything is strongly typed.
You need to create variables of the correct type and pass pointers to them, as
shown.


Preparing Queries
=================

You should, in general, always prepare queries to be used multiple times. The
result of preparing the query is a prepared statement, which can have
placeholders (a.k.a. bind values) for parameters that you'll provide when you
execute the statement.  This is much better than concatenating strings, for all
the usual reasons (avoiding SQL injection attacks, for example).

In MySQL, the parameter placeholder is `?`, and in PostgreSQL it is `$N`, where
N is a number. SQLite accepts either of these.  In Oracle placeholders begin with
a colon and are named, like `:param1`. We'll use `?` because we're using MySQL
as our example.

<pre class="prettyprint lang-go">
stmt, err := db.Prepare("select id, name from users where id = ?")
if err != nil {
	log.Fatal(err)
}
defer stmt.Close()
rows, err := stmt.Query(1)
if err != nil {
	log.Fatal(err)
}
defer rows.Close()
for rows.Next() {
	// ...
}
</pre>

Under the hood, `db.Query()` actually prepares, executes, and closes a prepared
statement. That's three round-trips to the database. If you're not careful, you
can triple the number of database interactions your application makes! Some
drivers can avoid this in specific cases with an addition to `database/sql` in
Go 1.1, but not all drivers are smart enough to do that. Caveat Emptor.

Naturally prepared statements and the managment of prepared statements cost
resources. You should take care to close statements when they are not used again.

Single-Row Queries
==================

If a query returns at most one row, you can use a shortcut around some of the
lengthy boilerplate code:

<pre class="prettyprint lang-go">
var name string
err = db.QueryRow("select name from users where id = ?", 1).Scan(&amp;name)
if err != nil {
	log.Fatal(err)
}
fmt.Println(name)
</pre>

Errors from the query are deferred until `Scan()` is called, and then are
returned from that. You can also call `QueryRow()` on a prepared statement:

<pre class="prettyprint lang-go">
stmt, err := db.Prepare("select id, name from users where id = ?")
if err != nil {
	log.Fatal(err)
}
var name string
err = stmt.QueryRow(1).Scan(&amp;name)
if err != nil {
	log.Fatal(err)
}
fmt.Println(name)
</pre>

Go defines a special error constant, called `sql.ErrNoRows`, which is returned
from `QueryRow()` when the result is empty. This needs to be handled as a
special case in most circumstances. An empty result is often not considered an
error by application code, and if you don't check whether an error is this
special constant, you'll cause application-code errors you didn't expect.

One might ask why an empty result set is considered an error. There's nothing
erroneous about an empty set. The reason is that the `QueryRow()` method needs
to use this special-case in order to let the caller distinguish whether
`QueryRow()` in fact found a row; without it, `Scan()` wouldn't do anything and
you might not realize that your variable didn't get any value from the database
after all.

You should not run into this error when you're not using `QueryRow()`. If you
encounter this error elsewhere, you're doing something wrong.

**Previous: [Accessing the Database](accessing.html)**
**Next: [Modifying Data and Using Transactions](modifying.html)**

---
layout: article
title: Modifying Data and Using Transactions
---

Now we're ready to see how to modify data and work with transactions. The
distinction might seem artificial if you're used to programming languages that
use a "statement" object for fetching rows as well as updating data, but in Go,
there's an important reason for the difference.

Statements that Modify Data
===========================

Use `Exec()`, preferably with a prepared statement, to accomplish an `INSERT`,
`UPDATE`, `DELETE`, or other statement that doesn't return rows. The following
example shows how to insert a row and inspect metadata about the operation:

<pre class="prettyprint lang-go">
stmt, err := db.Prepare("INSERT INTO users(name) VALUES(?)")
if err != nil {
	log.Fatal(err)
}
res, err := stmt.Exec("Dolly")
if err != nil {
	log.Fatal(err)
}
lastId, err := res.LastInsertId()
if err != nil {
	log.Fatal(err)
}
rowCnt, err := res.RowsAffected()
if err != nil {
	log.Fatal(err)
}
log.Printf("ID = %d, affected = %d\n", lastId, rowCnt)
</pre>

Executing the statement produces a `sql.Result` that gives access to statement
metadata: the last inserted ID and the number of rows affected.

What if you don't care about the result? What if you just want to execute a
statement and check if there were any errors, but ignore the result? Wouldn't
the following two statements do the same thing?

<pre class="prettyprint lang-go">
_, err := db.Exec("DELETE FROM users")  // OK
_, err := db.Query("DELETE FROM users") // BAD
</pre>

The answer is no. They do **not** do the same thing, and **you should never use
`Query()` like this.** The `Query()` will return a `sql.Rows`, which reserves a
database connection until the `sql.Rows` is closed.
Since there might be unread data (e.g. more data rows), the connection can not
be used. In the example above, the connection will *never* be released again.
The garbage collector will eventually close the underlying `net.Conn` for you,
but this might take a long time. Moreover the database/sql package keeps
tracking the connection in its pool, hoping that you release it at some point,
so that the connection can be used again.
This anti-pattern is therefore a good way to run out of resources (too many
connections, for example).

Working with Transactions
=========================

In Go, a transaction is essentially an object that reserves a connection to the
datastore. It lets you do all of the operations we've seen thus far, but
guarantees that they'll be executed on the same connection.

You begin a transaction with a call to `db.Begin()`, and close it with a
`Commit()` or `Rollback()` method on the resulting `Tx` variable. Under the
covers, the `Tx` gets a connection from the pool, and reserves it for use only
with that transaction. The methods on the `Tx` map one-for-one to methods you
can call on the database itself, such as `Query()` and so forth.

Prepared statements that are created in a transaction are bound exclusively to
that transaction, and can't be used outside of it. Likewise, prepared statements
created only on a database handle can't be used within a transaction.

You should not mingle the use of transaction-related functions such as `Begin()`
and `Commit()` with SQL statements such as `BEGIN` and `COMMIT` in your SQL
code. Bad things might result:

* The `Tx` objects could remain open, reserving a connection from the pool and not returning it.
* The state of the database could get out of sync with the state of the Go variables representing it.
* You could believe you're executing queries on a single connection, inside of a transaction, when in reality Go has created several connections for you invisibly and some statements aren't part of the transaction.

**Previous: [Retrieving Result Sets](retrieving.html)**
**Next: [Working with NULLs](nulls.html)**

---
layout: article
title: Working with NULLs
---

Nullable columns are annoying and lead to a lot of ugly code. If you can, avoid
them. If not, then you'll need to use special types from the `database/sql`
package to handle them, or define your own.

There are types for nullable booleans, strings, integers, and floats. Here's how
you use them:

<pre class="prettyprint lang-go">
for rows.Next() {
	var s sql.NullString
	err := rows.Scan(&amp;s)
	// check err
	if s.Valid {
	   // use s.String
	} else {
	   // NULL value
	}
}
</pre>

Limitations of the nullable types, and reasons to avoid nullable columns in case
you need more convincing:

1. There's no `sql.NullUint64` or `sql.NullYourFavoriteType`. You'd need to
	define your own for this.
1. Nullability can be tricky, and not future-proof. If you think something won't
	be null, but you're wrong, your program will crash, perhaps rarely enough
	that you won't catch errors before you ship them.
1. One of the nice things about Go is having a useful default zero-value for
	every variable. This isn't the way nullable things work.

If you need to define your own types to handle NULLs, you can copy the design of
`sql.NullString` to achieve that.

**Previous: [Modifying Data and Using Transactions](modifying.html)**
**Next: [Working with Unknown Columns](varcols.html)**

---
layout: article
title: Working with Unknown Columns
---

The `Scan()` function requires you to pass exactly the right number of
destination variables. What if you don't know what the query will return?

If you don't know how many columns the query will return, you can use
`Columns()` to find a list of column names. You can examine the length of this
list to see how many columns there are, and you can pass a slice into `Scan()`
with the correct number of values. For example, some forks of MySQL return
different columns for the `SHOW PROCESSLIST` command, so you have to be prepared
for that or you'll cause an error. Here's one way to do it; there are others:

<pre class="prettyprint lang-go">
cols, err := rows.Columns()
if err != nil {
	// handle the error
} else {
	dest := []interface{}{ // Standard MySQL columns
		new(uint64), // id
		new(string), // host
		new(string), // user
		new(string), // db
		new(string), // command
		new(uint32), // time
		new(string), // state
		new(string), // info
	}
	if len(cols) == 11 {
		// Percona Server
	} else if len(cols) &gt; 8 {
		// Handle this case
	}
	err = rows.Scan(dest...)
	// Work with the values in dest
}
</pre>

If you don't know the columns or their types, you should use `sql.RawBytes`.

<pre class="prettyprint lang-go">
cols, err := rows.Columns() // Remember to check err afterwards
vals := make([]interface{}, len(cols))
for i, _ := range cols {
	vals[i] = new(sql.RawBytes)
}
for rows.Next() {
	err = rows.Scan(vals...)
	// Now you can check each element of vals for nil-ness,
	// and you can use type introspection and type assertions
	// to fetch the column into a typed variable.
}
</pre>

**Previous: [Working with NULLs](nulls.html)**
**Next: [The Connection Pool](connection-pool.html)**

---
layout: article
title: The Connection Pool
---

There is a basic connection pool in the `database/sql` package. There isn't a
lot of ability to control or inspect it, but here are some things you might find
useful to know:

* Connections are created when needed and there isn't a free connection in the pool.
* By default, there's no limit on the number of connections. If you try to do a lot of things at once, you can create an arbitrary number of connections. This can cause the database to return an error such as "too many connections."
* In Go 1.1 or newer, you can use `db.SetMaxIdleConns(N)` to limit the number of *idle* connections in the pool. This doesn't limit the pool size, though.
* In Go 1.2.1 or newer, you can use `db.SetMaxOpenConns(N)` to limit the number of *total* open connections to the database. Unfortunately, a [deadlock bug](https://groups.google.com/d/msg/golang-dev/jOTqHxI09ns/x79ajll-ab4J) ([fix](https://code.google.com/p/go/source/detail?r=8a7ac002f840)) prevents `db.SetMaxOpenConns(N)` from safely being used in 1.2.
* Connections are recycled rather fast. Setting a high number of idle connections with `db.SetMaxIdleConns(N)` can reduce this churn, and help keep connections around for reuse.
* Keeping a connection idle for a long time can cause problems (like in [this issue](https://github.com/go-sql-driver/mysql/issues/257) with MySQL on Microsoft Azure). Try `db.SetMaxIdleConns(0)` if you get connection timeouts because a connection is idle for too long.

**Previous: [Working with Unknown Columns](varcols.html)**
**Next: [Surprises, Antipatterns and Limitations](surprises.html)**

---
layout: article
title: Surprises, Antipatterns and Limitations
---

Although `database/sql` is simple once you're accustomed to it, you might be
surprised by the subtlety of use cases it supports. This is common to Go's core
libraries.

Resource Exhaustion
===================

As mentioned throughout this site, if you don't use `database/sql` as intended,
you can certainly cause trouble for yourself, usually by consuming some
resources or preventing them from being reused effectively:

* Opening and closing databases can cause exhaustion of resources.
* Failing to read all rows or use `rows.Close()` reserves connections from the pool.
* Using `Query()` for a statement that doesn't return rows will reserve a connection from the pool.
* Failing to use prepared statements can lead to a lot of extra database activity.

Large uint64 Values
===================

Here's a surprising error. You can't pass big unsigned integers as parameters to
statements if their high bit is set:

<pre class="prettyprint lang-go">
_, err := db.Exec("INSERT INTO users(id) VALUES", math.MaxUint64)
</pre>

This will throw an error. Be careful if you use `uint64` values, as they may
start out small and work without error, but increment over time and start
throwing errors.

Connection State Mismatch
=========================

Some things can change connection state, and that can cause problems for two
reasons:

1. Some connection state, such as whether you're in a transaction, should be
	handled through the Go types instead.
2. You might be assuming that your queries run on a single connection when they
	don't.

For example, setting the current database with a `USE` statement is a typical
thing for many people to do. But in Go, it will affect only the connection that
you run it in. Unless you are in a transaction, other statements that you think
are executed on that connection may actually run on different connections gotten
from the pool, so they won't see the effects of such changes.

Additionally, after you've changed the connection, it'll return to the pool and
potentially pollute the state for some other code. This is one of the reasons
why you should never issue BEGIN or COMMIT statements as SQL commands directly,
too.

Database-Specific Syntax
========================

The `database/sql` API provides an abstraction of a row-oriented database, but
specific databases and drivers can differ in behavior and/or syntax.  One
example is the syntax for placeholder parameters in prepared statements. For
example, comparing MySQL, PostgreSQL, and Oracle:

	MySQL               PostgreSQL            Oracle
	=====               ==========            ======
	WHERE col = ?       WHERE col = $1        WHERE col = :col
	VALUES(?, ?, ?)     VALUES($1, $2, $3)    VALUES(:val1, :val2, :val3)

Multiple Result Sets
====================

The Go driver doesn't support multiple result sets from a single query in any
way, and there doesn't seem to be any plan to do that, although there is [a
feature request](https://code.google.com/p/go/issues/detail?id=5171) for
supporting bulk operations such as bulk copy.

This means, among other things, that a stored procedure that returns multiple
result sets will not work correctly.

Invoking Stored Procedures
==========================

Invoking stored procedures is driver-specific, but in the MySQL driver it can't
be done at present. It might seem that you'd be able to call a simple
procedure that returns a single result set, by executing something like this:

<pre class="prettyprint lang-go">
err := db.QueryRow("CALL mydb.myprocedure").Scan(&amp;result)
</pre>

In fact, this won't work. You'll get the following error: _Error
1312: PROCEDURE mydb.myprocedure can't return a result set in the given
context_. This is because MySQL expects the connection to be set into
multi-statement mode, even for a single result, and the driver doesn't currently
do that (though see [this
issue](https://github.com/go-sql-driver/mysql/issues/66)).

Multiple Statement Support
==========================

The `database/sql` doesn't explicitly have multiple statement support, which means
that the behavior of this is backend dependent:

<pre class="prettyprint lang-go">
_, err := db.Exec("DELETE FROM tbl1; DELETE FROM tbl2")
</pre>

The server is allowed to interpret this however it wants, which can include
returning an error, executing only the first statement, or executing both.

Similarly, there is no way to batch statements in a transaction. Each statement
in a transaction must be executed serially, and the resources in the results,
such as a Row or Rows, must be scanned or closed so the underlying connection is free
for the next statement to use.  Since transactions are connection state and bind
to a single database connection, you may wind up with a corrupted connection if
you attempt to perform another statement before the first has released that resource
and cleaned up after itself.  This also means that each statement in a transaction
results in a separate set of network round-trips to the database.

**Previous: [The Connection Pool](connection-pool.html)**
**Next: [Related Reading and Resources](references.html)**

---
layout: article
title: Related Reading and Resources
---

Here are some external sources of information we've found to be helpful.

* [http://golang.org/pkg/database/sql/](http://golang.org/pkg/database/sql/)
* [http://jmoiron.net/blog/gos-database-sql/](http://jmoiron.net/blog/gos-database-sql/)
* [http://jmoiron.net/blog/built-in-interfaces/](http://jmoiron.net/blog/built-in-interfaces/)

We hope you've found this website to be helpful. If you have any suggestions for
improvements, please send pull requests or open an issue report at
[https://github.com/VividCortex/go-database-sql-tutorial](https://github.com/VividCortex/go-database-sql-tutorial).

**Previous: [Surprises, Antipatterns and Limitations](surprises.html)**

