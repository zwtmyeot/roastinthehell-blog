---
title: "Go操作数据库：sqrl搭配sqlx使用"
date: 2020-04-08T02:50:05+08:00
draft: false
tags: [
    "go"
]
---

介绍了在Go中通过sqrl生成SQL语句，再用sqlx执行来操作数据库的方法。<!--more-->

ORM功能很强大，使用也很方便。但是前提是我们熟悉摸透了怎么使用，很多时候如果我们只是想简单地执行SQL语句，使用SQL语句生成器，再执行就可以了。没必要再学习使用ORM，毕竟有时候ORM的学习成本并不低。

## sqrl使用

> sqrl是一个非线程安全的SQL生成器

[sqrl](https://github.com/elgris/sqrl) fork 自 [squirrel](http://github.com/lann/squirrel)，去除了线程安全的特性。作为SQL语句的生成器，我们大多数的使用都不需要这个特性。

## 安装

```sh
go get -u github.com/elgris/sqrl
```

## 使用

以下就是最典型的使用，调用 sqrl的 `Select()`等函数传入生成SQL语句的列名、表名和条件等，然后通过 `ToSql()` 生成SQL语句

第二个返回值是一个切片，集合了传入的所有SQL语句中的参数

```go
query, args, err := sqrl.Select("u.id", "p.name").From("user as u", "profile as p").Where(sqrl.Eq{"id": 1}).ToSql()
fmt.Println(query) // SELECT u.id, p.name FROM user as u, profile as p WHERE id = ?
fmt.Println(args)  // [1]
```

### In 查询

在 `sqrl.Eq()` 的值字段传入一个切片

```go
query, args, err := sqrl.Select("u.id", "p.name").From("user as u", "profile as p").Where(sqrl.Eq{"id": []int{1, 2, 3}}).ToSql()
fmt.Println(query) // SELECT u.id, p.name FROM user as u, profile as p WHERE id IN (?,?,?)
fmt.Println(args)  // [1 2 3]
```

## 占位符

如果是使用 PostgreSQL ， 那么需要指定 dollor 占位符

```go
psql := sqrl.StatementBuilder.PlaceholderFormat(sqrl.Dollar)
// 使用 ？ 符号，会自动转成 $ 符号
query, _, _:= psql.Select("*").From("profile").Where("id = ?").ToSql()
fmt.Println(query) // 输出：SELECT * FROM profile WHERE id = $1

// 也可以在链式调用后面指定占位符
query, _, _:=sqrl.Select("*").From("profile").Where("id = ?").PlaceholderFormat(sqrl.Dollar).ToSql()
fmt.Println(query) // SELECT * FROM profile WHERE id = $1
```

## 搭配sqlx使用

先使用 sqrl 构造好 SQL 语句，再使用 sqlx 查询

```go
// sqrl搭配sqlx使用
query, _, _ := sqrl.Select("u.id", "p.name").From("\"user\" as u", "profile as p").Where("p.id = u.id").PlaceholderFormat(sqrl.Dollar).ToSql()
up := []userProfile{}
db.Select(&up, query)
fmt.Println(up)
```

## 插入、更新和删除

插入一条新的行：

```go
query, args, err := sqrl.Insert("profile").Columns("name", "created_at", "updated_at").Values("2543配置", time.Now(), time.Now()).PlaceholderFormat(sqrl.Dollar).ToSql()
fmt.Println(query, args)

// 注意执行的时候，要把参数切片展开，使用 ... 符号
result, err := db.Exec(query, args...)
if err != nil {
    fmt.Println(err)
    panic(err)
}
fmt.Println(result.RowsAffected()) // 1
```

修改某一行

```go
query, args, err := sqrl.Update("profile").Set("name", "配置3444").Where("id = ?", 24).PlaceholderFormat(sqrl.Dollar).ToSql()

result, err := db.Exec(query, args...)
```

删除一行：

```go
query, args, err := sqrl.Delete("profile").Where("id = ?", 1).PlaceholderFormat(sqrl.Dollar).ToSql()

result, err := db.Exec(query, args...)
```

## 针对PostgreSQL的某些特性

插入 json类型的值

```go
sql, args, err := sqrl.Insert("posts").
    Columns("content", "tags").
    Values("Lorem Ipsum", pg.JSONB([]string{"foo", "bar"})).
    ToSql()
```

## sqlx的使用

> `sqlx` 是对Go内置的 `database/sql` 包的拓展

知道怎么通过sqlx连接数据库，看一下执行相关的就行了。对于 sqlx 的 in查询、命名查询这些，是你没有使用sqrl生成语句时才需要用到的。

## 安装sqlx

```go
go get -u github.com/jmoiron/sqlx

// 安装postgresql驱动
go get github.com/lib/pq
```

## 连接数据库

```go
package main

import (
    "fmt"
    "github.com/jmoiron/sqlx"
    // 导入相对于的数据库驱动
    _ "github.com/lib/pq"
)

func main() {
    db, err := sqlx.Open("postgres", "user=postgres password=your_password dbname=your_db sslmode=disable")
    checkErr(err)
    // 强制连接一下数据库，测试是否有问题
    err = db.Ping()
    checkErr(err)
    defer db.Close()
}

// 检查错误
func checkErr(err error) {
    if err != nil {
        fmt.Println("has error: ", err)
    }
}
```

## 占位符

在SQL语句字符串中使用 `?` 表示占位符，然后在占位符的位置插入参数。但是要注意这个占位符不能用于表名，列名这些操作，仅限于传入数值参数。

MySQL 使用的占位符是 `?` ，直接执行即可。PostgreSQL 使用的是 `$1, $2...` 这些占位符，当你手动填入占位符时请到 [官方文档](http://jmoiron.github.io/sqlx/) 内查询自己使用的数据库对应的占位符。也可以使用 `rebind()` 直接转换成你所使用的的数据库对应的占位符。

> 如果是使用sqrl，那么在sqrl中指定了占位符就行了

```go
query := `delete from profile where name=?`
fmt.Println(query) // delete from profile where name=?
query = db.Rebind(query)
fmt.Println(query) // delete from profile where name=$1
```

## 执行SQL语句

`sqlx` 只是对go内置的 `database/sql` 包进行了拓展，有以下几个方法用于执行SQL语句：

- `Exec(...) (sql.Result, error)` - 和 database/sql 的一样
- `Query(...) (*sql.Rows, error)` - 和 database/sql 的一样
- `QueryRow(...) *sql.Row` - 和 database/sql 的一样

- `MustExec() sql.Result` -- 执行过程中有error的话会panic

创建一个表：

```go
schema := `CREATE TABLE place (
    country text,
    city text NULL,
    telcode integer);`

// 执行SQL语句
result, err := db.Exec(schema)
checkErr(err)
fmt.Println(result)
```

## 查询

sqlx提供了扫描结构体的查询方法，我们可以先定义好一个结构体，对应数据库里table的列。在查询的时候，传入结构体变量的指针即可在对应的字段中填入数据。有以下这些方法：

- `Queryx(...) (*sqlx.Rows, error)`
- `QueryRowx(...) *sqlx.Row`
- `Get(dest interface{}, ...) error`
- `Select(dest interface{}, ...) error`

定义结构体：

```go
type Profile struct {
    ID        int       `db:"id"`
    CreatedAt time.Time `db:"created_at"`
    UpdatedAt time.Time `db:"updated_at"`
    // 如果表中这一列的值可能是NULL，那么需要定义类型为 NULL*的类型
    // 不然从数据库中查询值为null，在以 nil->time.Time 转换的时候会报错
    DeletedAt sql.NullTime `db:"deleted_at"`
    Name      string       `db:"name"`
}
```

首先要注意的是字段名要大写开头，如果是小写的字段名，那么将不会被导出，即sqlx无法访问，不能为这个字段赋值

我们可以在字段后面使用Tag语法，为字段指定数据表中对应的名字，如果不指定，sqlx会将字段名转换成全小写，以此作为数据表中的列名来关联。

使用 `Queryx()` 获得一行结果，使用 `QueryRowx()` 获得多行结果。需要自己手动取出结果中的数据。注意


使用 `Get()` 获取一行查询结果，自动赋值到传入的结构体变量，使用 `Select()` 获取多行查询结果，自动赋值到传入的结构体变量数组

```go
// 测试get
var profile Profile
err = db.Get(&profile, "select * from profile")
fmt.Println(profile)

// 测试select
var profiles []Profile
err = db.Select(&profiles, "select * from profile")
fmt.Println(profiles)
```

## 插入

## `exec()` 和 `query()`的区别

如果只是为了执行SQL语句，那么两个方法都是可以用来执行。他们最主要的区别就是返回类型不一样，`exec()` 返回 `sql.Result`， `query()` 会返回 `*sql.Rows`对象的指针。

`sql.Result` 含有两个接口方法：

- `LastInsertId() (int64, error)`：最后一个行的ID。注意这个方法在PostgreSQL中无效，不能获取到ID。

- `RowsAffected() (int64, error)`：影响的行数。

如果你想要知道影响的行数，那么使用 `exec()`，如果你不需要知道行数，使用 `query()` 即可。

参考：[Why even use *DB.exec() or prepared statements in Golang?
](https://stackoverflow.com/questions/50664648/why-even-use-db-exec-or-prepared-statements-in-golang/50666083)

## 在PostgreSQL中获得最后行的ID

PostgreSQL驱动不支持 `LastInsertId()` 方法，所以不能使用 `exec()` 后获取到最后插入（更新、删除）行的ID。需要使用 `RETURNING` 从句，从查询中获得ID：

```go
lastInsertId := 0
// 使用了 RETURNING从句返回了ID列的值
err = db.QueryRow("INSERT INTO brands (name) VALUES($1) RETURNING id", name).Scan(&lastInsertId)
```

## `Exec`

`Exec(...) (sql.Result, error)` 是用于直接执行SQL语句，返回一个 `sql.Result` 接口变量，里面有两个函数：

- `LastInsertId() (int64, error)` 返回最后的行，通常是插入的时候自增ID。PostgreSQL的驱动并不支持这个函数，获取不到ID。

- `RowsAffected() (int64, error)` 返回影响的行数

一个执使用的例子：

```go
query := `insert into profile(created_at, updated_at, name) values(?,?,?)`
query = db.Rebind(query)
result, err :=db.Exec(query, time.Now(), time.Now(), "配置234")
checkErr(err)
fmt.Println(result.RowsAffected())
```

## `Query`

`Query()` 也是用于执行SQL语句，区别是会返回一个SQL执行的结果集。例如查询一行数据，查询多行数据。应该把这个返回的结果集当成SQL游标，而不是一个object对待

一个使用的例子：

```go
rows, err := db.Query(`select id, created_at, "name" from profile`)
checkErr(err)
// 依次调用 Next() 函数最后会自动调用 Close()
// 如果没有使用 Next() 请手动调用Close()
// defer rows.close()
for rows.Next() {
    var ID int
    var created_at time.Time
    var name string
    // 按照查询的顺序填入字段
    rows.Scan(&ID, &created_at, &name)
    fmt.Println(ID, created_at, name)
}
```

### `Queryx()`

`sqlx` 拓展了 `Query()` 的使用，提供了 `Queryx()`，可以传入一个结构体变量指针，自动给对应字段复制。sqlx会小写化字段名称，与列名对应。可以使用 tag 标签，对结构体字段提供另外的命名。

```go
rows, err := db.Queryx(`select id, created_at, "name" from profile`)
checkErr(err)
// 依次调用 Next() 函数最后会自动调用 Close()
// 如果没有使用 Next() 请手动调用Close()
// defer rows.close()
for rows.Next() {
    p:=Profile{}
    // 按照查询的顺序填入字段
    err := rows.StructScan(&p)
    checkErr(err)
    fmt.Println(p)
}
```

### `QueryRow()` 和 `QueryRowx()`

`QueryRow()` 返回一行，`QueryRowx()` 的返回一行可以扫描结构体。与数据库的连接会在scan后关闭，所以要么scan，要么手动调用close。

```go
// QueryRow
row:= db.QueryRow(`select id, created_at, "name" from profile`)
var id int
var created_at time.Time
var name string
// 连接会在scan以后关闭
err =row.Scan(&id, &created_at,&name)
checkErr(err)
fmt.Println(id, created_at, name)

// QueryRowx
row:= db.QueryRowx(`select id, created_at, "name" from profile`)
p := Profile{}
err =row.StructScan(&p)
checkErr(err)
fmt.Println(p)
```

### `Get()` 和 `Select()`

`Get()` 和 `Select()` 是在执行SQL语句的时候，直接传入结构体变量或者结构体变量数组，然后赋值结果。这对查询提供了遍历。

`Get()` 获得一行结果， `Select()` 获得多行结果。

```go
profile := Profile{}
query = `select * from profile limit 1`
// 获取一行
err = db.Get(&profile, query)
checkErr(err)
fmt.Println(profile)

query := `select * from profile`
profiles := []Profile{}
// 获取所有结果
err = db.Select(&profiles, query)
checkErr(err)
for _, profile :=range profiles {
    fmt.Println(profile)
}
```

虽然 `Select()` 提供了便利，但是它会一次性载入所有查询结果到内存，如果你的查询结果集非常大，请使用 `QueryX()` 来逐行处理结果。

## 事务

要使用事务，必须要使用 `db.Begin()`获得一个事务对象，然后使用这个事务对象逐一执行同一个事务中的语句。

```go
tx, err := db.Begin()
err = tx.Exec(...)
err = tx.Commit()
```

可以调用 `db.Beginx()` 获得返回类型为 `*sqlx.Tx` 的对象，这样就可以使用 `Queryx()` 和 `Get()` 这些具有扫描功能的查询函数。

sqlx的事务仅仅是绑定一个数据库连接，每次执行一条语句，具体的事务实现依然依赖于数据库内部实现。在sqlx的事务中，Row 和 Rows 类型返回值，必须使用 scan 或者手动调用 close 来关闭自己的连接，不然无法执行下一条语句。

## 预处理语句

我们可以先处理好查询语句，然后等需要的时候再执行。

```go
// Prepare就是普通查询
stmt, err:=db.Prepare(`select * from profile where name = $1`)
checkErr(err)
rows, err:= stmt.Query(&p, "测试33")
for rows.Next(){
}

// Preparex返回的对象，查询后可以对传入的结构体变量自动赋值
stmt, err:=db.Preparex(`select * from profile where name = $1`)
checkErr(err)
p:=Profile{}
err = stmt.Get(&p, "测试33")
fmt.Println(p)
```

## "In" 查询

`sqlx.In()` 的第二个参数是一个切片， `sqlx.In()` 按照切片的大小，替换查询语句（第一个参数）中的 `?` 标记符为对应的个数。在执行的时候，再把想填入 In 子句中的参数打散（ `...` 语法糖）传入到query语句，即可完成 In子句查询。

```go
IDs := []int{1, 3, 23}
query := `select * from profile where id in (?);`
query, args, err := sqlx.In(query, IDs)
fmt.Printf("%v\n", query) // select * from profile where id in (?, ?, ?);

// 把 ? 占位符换成PostgreSQL中的$1, $2
query = db.Rebind(query)
profiles := []Profile{}
err = db.Select(&profiles, query, args...)
fmt.Println(profiles)
```

## 命名查询

在查询的时候，可以使用 `NameQuery()` 这些方法，不需要按照位置传入参数，传入结构体变量即可按照字段名对应插入查询语句中

```go
profile := Profile{Name:"测试33"}
query := `select * from profile where name = :name`
rows, err := db.NamedQuery(query, &profile)
for rows.Next(){
}
```

也可以预处理好语句

```go
profile := Profile{Name:"测试33"}
query := `select * from profile where name = :name`
stmt ,err := db.PrepareNamed(query)
checkErr(err)
profiles := []Profile{}
err = stmt.Select(&profiles, &profile)
fmt.Println(profiles)
```

如果想使用 In子句查询，那么需要使用 `sqlx.Named()` 来处理命名，在用 `sqlx.In()` 处理 in子句里面的参数

```go
query := `select * from profile where id in (:ids)`
arg := map[string]interface{}{
    "ids": []int{1, 3, 23},
}
query, args, err := sqlx.Named(query, arg)
query, args, err = sqlx.In(query, args...)
query = db.Rebind(query)

profiles := []Profile{}
db.Select(&profiles, query, args...)
fmt.Println(profiles)
```

## 参考

- [sqrl](https://github.com/elgris/sqrl)

- [Go 优雅的 SQL 语句拼接库](https://www.5-wow.com/article/detail/77)

- [Illustrated guide to SQLX](http://jmoiron.github.io/sqlx/)