# Documentation

## Opening Database

To open a new Database connection, construct the `Database` class with the path
to the database file. If the file does not exist, it will be created unless you
pass `create: false` in the options.

### Options

- `create: boolean` - Whether to create the database file if it does not exist.
  Defaults to `true`.
- `readonly: boolean` - Whether to open the database in read-only mode. Defaults
  to `false`.
- `memory: boolean` - Whether to open the database in memory. Defaults to
  `false`.
- `int64: boolean` - Whether to support BigInt columns. False by default, which
  means integers larger than 32 bit will be inaccurate.
- `flags: number` - Raw flags to pass to the C API. Normally you don't need
  this. Passing this ignores all other options.

### Usage

```ts
// Open using default options
const db = new Database("test.db");

// Open using URL path (relative to current file/module, not CWD)
const db = new Database(new URL("./test.db", import.meta.url));

// Open in memory
const db = new Database(":memory:");

// Open in read-only mode
const db = new Database("test.db", { readonly: true });

// Open existing database, error if it doesn't exist
const db = new Database("test.db", { create: false });
```

## Properties of `Database`

- `inTransaction: boolean` - Whether the database is currently in a transaction.
- `open: boolean` - Whether the database connection is open.
- `path: string` - The path to the database file (not full path, just the once
  passed to the constructor).
- `totalChanges: number` - The total number of changes made to the database
  since it was opened.
- `changes: number` - The number of changes made to the database by last
  executed statement.
- `lastInsertRowId: number` - The rowid of the last inserted row.
- `autocommit: boolean` - Whether the database is in autocommit mode. This is
  `true` when not in a transaction, and `false` when in a transaction.

## Closing Database

To close the database connection, call the `close()` method. This will close the
database connection and free all resources associated with it. Calling it more
than once will be a no-op.

```ts
db.close();
```

## Executing SQL

To execute SQL statements, use the `exec()` method. This method will execute all
statements in the SQL string, and return the number of changes made by the last
statement. This method is useful for executing DDL statements, such as `CREATE`,
`DROP`, `ALTER`, and even pragma statements that do not return any data.

```ts
const changes = db.exec(
  "CREATE TABLE foo (bar TEXT); INSERT INTO foo VALUES ('baz');",
);

console.log(changes); // 1

// Executing pragma statements
db.exec("pragma journal_mode = WAL");
db.exec("pragma synchronous = normal");
db.exec("pragma temp_store = memory");
```

Any parameters past the first argument will be bound to the statement. When you
pass parameters, the function under the hood instead uses the prepared statement
API.

See [Binding Parameters](#binding-parameters) for more details.

## Creating Prepared Statements

To prepare a statement, use the `prepare()` method. This method will return a
`Statement` object, which can be used to execute it, bind the parameters,
retrieve the results, and more.

```ts
const stmt = db.prepare("SELECT * FROM foo WHERE bar = ? AND baz = ?");
```

For more details on binding parameters, see
[Binding Parameters](#binding-parameters).

## Properties of `Statement`

- `db: Database` - The database the statement belongs to.
- `expandedSql: string` - The SQL string with all bound parameters expanded.
- `sql: string` - The SQL string used to prepare the statement.
- `readonly: boolean` - Whether the statement is read-only.
- `bindParameterCount: number` - The number of parameters the statement expects.

## Executing Statement

To execute a statement, use the `run()` method. This method will execute the
statement, and return the number of changes made by the statement.

```ts
const changes = stmt.run(...params);
```

## Retrieving Rows

To retrieve rows from a statement, use the `all()` method. This method will
execute the statement, and return an array of rows as objects.

```ts
const rows = stmt.all(...params);
```

To get rows in array form, use `values()` method.

```ts
const rows = stmt.values(...params);
```

To get only the first row as object, use the `get()` method.

```ts
const row = stmt.get(...params);
```

To get only the first row as array, use the `value()` method.

```ts
const row = stmt.value(...params);
```

`all`/`values`/`get`/`value` methods also support a generic type parameter to
specify the type of the returned object.

```ts
interface Foo {
  bar: string;
  baz: number;
}

const rows = stmt.all<Foo>(...params);
// rows is Foo[]

const row = stmt.get<Foo>(...params);
// row is Foo | undefined

const values = stmt.values<[string, number]>(...params);
// values is [string, number][]

const row = stmt.value<[string, number]>(...params);
// row is [string, number] | undefined
```

## Freeing Prepared Statements

Though the `Statement` object is automatically freed once it is no longer used,
that is it's caught by the garbage collector, you can also free it manually by
calling the `finalize()` method. Do not use the `Statement` object after calling
this method.

```ts
stmt.finalize();
```

## Setting fixed parameters

To set fixed parameters for a statement, use the `bind()` method. This method
will set the parameters for the statement, and return the statement itself.

It can only be called once and once it is called, changing the parameters is not
possible. It's merely an optimization to avoid having to bind the parameters
every time the statement is executed.

```ts
const stmt = db.prepare("SELECT * FROM foo WHERE bar = ? AND baz = ?");
stmt.bind("bar", "baz");
```

## Iterating over Statement

If you iterate over the statement object itself, it will iterate over the rows
step by step. This is useful when you don't want to load all the rows into
memory at once. Since it does not accept any parameters, you must bind the
parameters before iterating using `bind` method.

```ts
for (const row of stmt) {
  console.log(row);
}
```

## Transactions

To start a transaction, use the `transaction()` method. This method takes a
JavaScript function that will be called when the transaction is run. This method
itself returns a function that can be called to run the transaction.

When the transaction function is called, `BEGIN` is automatically called. When
the transaction function returns, `COMMIT` is automatically called. If the
transaction function throws an error, `ROLLBACK` is called.

If the transaction is called within another transaction, it will use
`SAVEPOINT`/`RELEASE`/`ROLLBACK TO` instead of `BEGIN`/`COMMIT`/`ROLLBACK` to
create a nested transaction.

The returned function also contains `deferred`/`immediate`/`exclusive`
properties (functions) which can be used to change `BEGIN` to
`BEGIN DEFERRED`/`BEGIN IMMEDIATE`/`BEGIN EXCLUSIVE`.

```ts
const stmt = db.prepare("INSERT INTO foo VALUES (?)");
const runTransaction = db.transaction((data: SomeData[]) => {
  for (const item of data) {
    stmt.run(item.value);
  }
});

runTransaction([
  { value: "bar" },
  { value: "baz" },
]);

// Run with BEGIN DEFERRED

runTransaction.deferred([
  { value: "bar" },
  { value: "baz" },
]);
```

## Binding Parameters

Parameters can be bound both by name and positiion. To bind by name, just pass
an object mapping the parameter name to the value. To bind by position, pass the
values as rest parameters.

SQLite supports `:`, `@` and `$` as prefix for named bind parameters. If you
don't have any in the Object's keys, the `:` prefix will be used by default.

Bind parameters can be passed to `Database#exec` after SQL parameter, or to
`Statement`'s `bind`/`all`/`values`/`run` function.

```ts
// Bind by name
db.exec("INSERT INTO foo VALUES (:bar)", { bar: "baz" });

// In prepared statements
const stmt = db.prepare("INSERT INTO foo VALUES (:bar)");
stmt.run({ bar: "baz" });

// Bind by position
db.exec("INSERT INTO foo VALUES (?)", "baz");

// In prepared statements
const stmt = db.prepare("INSERT INTO foo VALUES (?, ?)");
stmt.run("baz", "foo");
```

JavaScript to SQLite type mapping:

| JavaScript type | SQLite type       |
| --------------- | ----------------- |
| `null`          | `NULL`            |
| `undefined`     | `NULL`            |
| `number`        | `INTEGER`/`FLOAT` |
| `bigint`        | `INTEGER`         |
| `string`        | `TEXT`            |
| `boolean`       | `INTEGER`         |
| `Date`          | `TEXT` (ISO)      |
| `Uint8Array`    | `BLOB`            |

When retrieving rows, the types are mapped back to JavaScript types:

| SQLite type | JavaScript type   |
| ----------- | ----------------- |
| `NULL`      | `null`            |
| `INTEGER`   | `number`/`bigint` |
| `FLOAT`     | `number`          |
| `TEXT`      | `string`          |
| `BLOB`      | `Uint8Array`      |
