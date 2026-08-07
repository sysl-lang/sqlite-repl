# sqlite-repl

**A worked example, not a library.** One [sysl](https://github.com/sysl-lang/sysl) program that
depends on **three** packages and is a SQL prompt: type a statement, see the answer laid out.

[`monocypher-example`](https://github.com/sysl-lang/monocypher-example) is the shortest answer to
"what does a sysl project with *a* dependency look like". This is the answer to the next question,
which is a different one — what happens when several packages have to be a program together.

```
linenoise   reads a line, with history and the arrow keys
sqlite      runs it
table       lays the rows out so the columns line up
```

## Run it

```
sysl run . -- mydata.db
sysl run .                  # no file named, so an in-memory database
```

Needs sysl **0.0.23 or newer**. Older versions bound a dependency's top-level *directory* rather than
its modules, so all three packages here — each laid out as `sh/sysl/<name>/` — claimed the single
name `sh` and refused to resolve together. This is the program that found it: one dependency binds
`sh` with nothing to collide with, so the shape that fails is the shape nobody had written.

```
sqlite> CREATE TABLE stock (item TEXT, qty INTEGER, note TEXT)
ok
sqlite> INSERT INTO stock VALUES ('apple', 12, NULL)
ok
sqlite> INSERT INTO stock VALUES ('café', 3, 'accented')
ok
sqlite> INSERT INTO stock VALUES ('日本の梨', 100, 'wide')
ok
sqlite> SELECT * FROM stock
┌──────────┬─────┬──────────┐
│   item   │ qty │   note   │
│ apple    │ 12  │ NULL     │
│ café     │ 3   │ accented │
│ 日本の梨 │ 100 │ wide     │
└──────────┴─────┴──────────┘
3 rows
sqlite> .style markdown
sqlite> SELECT item, qty FROM stock ORDER BY qty
|   item   | qty |
| -------- | --- |
| café     | 3   |
| apple    | 12  |
| 日本の梨 | 100 |
3 rows
```

**Look at where the borders fall in that first table.** `café` is five bytes and four columns;
`日本の梨` is twelve bytes, four characters and **eight** columns. Only the last of those numbers
says where the next border goes, which is why the table package measures text the way it does and
why a format specifier's width — which counts bytes, deliberately — could not have laid this out.

## The commands

```
.help            this
.tables          the tables and views in this database
.schema          the SQL that made them
.style <name>    plain, ascii, light, heavy, double, rounded, markdown
.quit            leave, saving the history
```

Anything else is one SQL statement. Ctrl-D leaves too, and the history is written to
`.sqlite-repl-history` in the directory you started in.

## What the three files are

```
package.hocon    the three dependencies, and what the program needs of the machine
main.sysl        the program
sysl.sum         the digest of what was fetched, written by the first run
```

## The parts worth reading

- **Nothing glues the packages together, and that is the thing to notice.** There is no adapter, no
  shared type and no configuration passed between them. `read` answers with an `Option[string]`,
  `text_at` with a `Result[Option[string], string]`, and `add` takes anything that renders — so every
  seam is one of the standard module's own types, which all three packages were written against
  separately and none of them agreed with the others about.

- **A row is built at run time out of values, not strings.** How many columns a query has is not
  known until it has been compiled, so each row is a `Buf[&Display]` filled in a loop and handed to
  `add`. A `&Display` is what lets a cell be any type that renders; the REPL's cells all happen to be
  text, but nothing in the table package knows that.

- **`finalize` is asked nothing afterwards.** Every method on a `Query` is a call through a handle
  into SQLite's storage, and `finalize` gives that storage back — so the column count is taken
  *before* and kept. Asking again after is a use-after-free that segfaults with no output at all,
  because the crash discards whatever stdout had buffered. It is the one mistake this program made
  while it was being written, and it is written down here because a binding to a library that
  allocates is exactly where it is waiting.

- **A text column is read for two different failures.** `Ok(None)` is SQL NULL and `Err` is bytes
  that are not UTF-8, and they are not the same thing — NULL is shown as `NULL` rather than as an
  empty cell, because a blank cell cannot tell the reader which of the two it met.

- **The rows are rendered as they arrive.** A text column's bytes belong to SQLite and stay valid
  only until the next `step`, so a row held back to be rendered later is a row whose contents have
  moved underneath it. `add` renders each cell as it is handed over, which is what makes the loop
  safe rather than lucky.

- **A line holding two statements is refused rather than half-run.** The binding compiles a single
  statement and passes SQLite no pointer to whatever followed it, so `CREATE TABLE a (x); INSERT INTO
  a VALUES (1)` would run the `CREATE` and drop the `INSERT` silently. Refusing is the honest answer
  while that is true. The scan that decides knows a semicolon inside a quoted string is not a
  separator — SQL escapes a quote by doubling it, which needs no case of its own — so
  `SELECT ';' AS semi` is one statement and is run.

- **Numbers are not right-aligned**, though the table package would do it. Deciding which columns are
  numeric means asking SQLite the type of each column, and the binding does not offer
  `sqlite3_column_type` yet. Aligning on what the text *looks* like would be a guess, and a table
  that guesses wrong about one row is worse than one that never tried.

## Licence

Public domain (CC0). It vendors nothing; each package carries its own licence.
