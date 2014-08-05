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

这几乎是用Go获取数据的的唯一方法。例如，你并不能把行作为`map`获取。因为所有的东西都是强类型的。你需要创建正确类型的变量，并且把指针传给它们，如上例所示。


###准备查询

一般情况下，你应该总是准备好要使用多次的查询。准备查询的结果就是一个准备好的语句。语句中可以包含执行语句时提供的参数占位符（即绑定值）。这比串联字符串的拼接好很多，因为可以避免一些常见问题（比如SQL注入）。

在MySQL中，参数占位符是`?`，在PostgreSQL中是`$N`，其中N是一个数字。SQLite可以使用两者的任何一个。在Oracle中占位符以冒号开头，紧接着是名称，比如`:param1`。此处我们使用`?`因为我们在例子中使用的是MySQL。

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

在底层，`db.Query()`实际做了准备、执行并且关闭一个准备好的语句的工作。这对数据库来说其实是3个来回。如果你不仔细，你可能会让应用与数据库交互的次数增至3倍！在Go1.1中，一些驱动在某些具体情况下通过对`database/sql`做一些附加来避免这种情况，但是并不是所有的驱动都这么做了。请注意！

当然，准备的语句和对准备语句的管理都会消耗资源。不再使用它们时，请关闭这些语句。

###单行查询

如果一个查询几乎每次都是只返回一行，那么你可以使用一个快捷方法来避免冗长的样板代码：

	var name string
	err = db.QueryRow("select name from users where id = ?", 1).Scan(&amp;name)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(name)

来自查询的错误被延迟到`Scan()`被调用时，在那之后才会返回。你也可以在准备语句中调用`QueryRow()`：

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

Go定义了一个特殊的错误常量`sql.ErrNoRows`，当结果为空时，`QueryRow()`返回它。在大多数情况下都需要将其作为一个特例进行处理。应用一般不会将空结果作为错误来看待，所以如果你不检查错误是否是这个特例，可能就会造成意想不到的应用代码错误。

有人会问什么将空结果作为一种错误来看待。一个空集合并没有错。之所以这样做的原因是因为`QueryRow()`方法需要用这个特例来告知调用者`QueryRow()`是否实际返回了一行；没有它，`Scan()`将不能够做任何事，你也可能不会意识到你的变量没有从数据库中获得任何值。

你应该不会在其它地方没有使用`QueryRow()`的地方遇到这个错误。如果你在其它地方遇到了这个错误，那说明你一定哪里做错了。

##修改数据和使用事务

现在我们准备好了研究如何修改数据和使用事务了。如果你习惯了以“语句”对象来获取和更新数据的方法来编程，那么在Go中，这种区别有一个重要的原因。

###修改数据的语句

使用`Exec()`（最好与准备语句结合使用），来完成`INSERT`, `UPDATE`, `DELETE` 或者其它不返回行的语句。下面的例子展示了如何插入一行并且检查操作的元数据

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

执行语句会产生一个`sql.Result`，它提供了对语句元数据的访问：最后插入的ID和影响的行数。

What if you don't care about the result? What if you just want to execute a
statement and check if there were any errors, but ignore the result? Wouldn't
the following two statements do the same thing?
如果你不关心返回的结果呢？或者如果你仅仅想执行一个语句，然后检查是否有错误而不管结果呢？那下面的两条语句是否是一样？

	_, err := db.Exec("DELETE FROM users")  // OK
	_, err := db.Query("DELETE FROM users") // BAD

答案是否定的。它们并**不是**一样的，而且**用于不要这样用`Query()`**。`Query()`返回`sql.Rows`，在`sql.Rows`关闭之前，它将一直占用这个数据库连接。

因为也许存在未读的数据（多行），连接不可用。在上例中，连接**永远**都不会被释放。垃圾回收器最终会为你关闭底层的`net.Conn`，但那会消耗很长一段时间。此外，database/sql包保持着对连接池中连接的跟踪，期冀在某一时间点释放它，那样连接就再次可用了。这种反模式是耗尽资源的一种很好方法（比如，过多的连接）。

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

