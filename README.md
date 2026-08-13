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
SQLITE_H=$(xcrun --show-sdk-path)/usr/include      # macOS; on Linux it is /usr/include
sysl run . --include-path sqlite3=$SQLITE_H -- mydata.db
sysl run . --include-path sqlite3=$SQLITE_H        # no file named, so an in-memory database
```

**The flag is the `sqlite3` package saying it does not carry SQLite's header**, and it is the one
thing here that is not just `sysl run .`. That package's C includes `<sqlite3.h>`, no copy of it is in
the package, and `package.hocon` there declares as much — so a build has to be told where one is, by
that name. The other two dependencies carry their own C and ask for nothing.

On macOS this points at a directory clang searches anyway. It is still required, deliberately: a bare
`--include-path` does not answer a declaration, because the check asks what a build says it has
rather than what it might happen to find. On a Linux box with no `libsqlite3-dev` it is the whole
difference between a refusal that names what to install and `fatal error: 'sqlite3.h' file not found`
out of a package you did not write.

Built and run against sysl **0.0.48**. The floor is `sqlite3` 0.6.0's rather than this program's, since
the binding reads SQLite's result codes with `c const` and declares the header it needs — neither of
which the compiler had in the 0.0.23 this file used to name.

That older number is worth keeping for its reason: before 0.0.23 a dependency's top-level *directory*
was bound rather than its modules, so all three packages here — each laid out as `sh/sysl/<name>/` —
claimed the single name `sh` and refused to resolve together. This is the program that found it,
because one dependency binds `sh` with nothing to collide with and the shape that fails is the shape
nobody had written.

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
  `text_at` with a `Result[Option[string], Error]`, and `add` takes anything that renders — so every
  seam is `Option`, `Result` or `Display`, which all three packages were written against separately
  and none of them agreed with the others about. sqlite's own error type reaches the screen through a
  function that knows nothing about it, because it renders.

- **A row is built at run time out of values, not strings.** How many columns a query has is not
  known until it has been compiled, so each row is a `Buf[&Display]` filled in a loop and handed to
  `add`. A `&Display` is what lets a cell be any type that renders; the REPL's cells all happen to be
  text, but nothing in the table package knows that.

- **Nothing here releases anything, and this is where that changed.** The program used to call
  `finalize` on every statement and `close` on the database, and it carried a paragraph explaining
  that a `Query`'s column count had to be taken *before* the `finalize` — because asking afterwards
  was a use-after-free that segfaulted with no output at all, the crash discarding whatever stdout had
  buffered. That was the one mistake this program made while it was being written.

  **It is not a mistake that can be made now.** `sqlite3` 0.6.0 holds both handles behind a `&` with a
  destructor, so a statement is finalized when the last reference to it goes and a connection is closed
  after the last statement compiled from it — the order SQLite requires, kept by the type rather than
  by a comment. Six lines went, and so did the paragraph that used to be here warning about them.

- **A text column is read for two different failures.** `Ok(None)` is SQL NULL and `Err` is bytes
  that are not UTF-8, and they are not the same thing — NULL is shown as `NULL` rather than as an
  empty cell, because a blank cell cannot tell the reader which of the two it met.

- **The rows are rendered as they arrive.** A text column's bytes belong to SQLite and stay valid
  only until the next `step`, so a row held back to be rendered later is a row whose contents have
  moved underneath it. `add` renders each cell as it is handed over, which is what makes the loop
  safe rather than lucky.

- **A line may hold several statements, and each is run and reported in turn.** `CREATE TABLE a (x);
  INSERT INTO a VALUES (1)` runs both. That is `Db.statements`, the binding's cursor over
  `sqlite3_prepare_v2`'s trailing-text pointer: SQLite compiles the first statement and says where
  the rest begins, so reaching the second one is a matter of receiving that pointer.

  This program used to refuse such a line, because the binding passed null for it — and the refusal
  needed a quote-aware scan of its own to know that the semicolon in `SELECT ';' AS semi` is not a
  separator. **The whole of that scan is gone**, and SQLite decides where a statement ends, which it
  was always going to be better at. It is the clearest thing this example has to say about binding a
  C library: the workaround was in the consumer and the gap was in the binding, and closing the gap
  is measured by the workaround disappearing.

- **Numbers are not right-aligned**, though the table package would do it. Deciding which columns are
  numeric means asking SQLite the type of each column, and the binding does not offer
  `sqlite3_column_type` yet. Aligning on what the text *looks* like would be a guess, and a table
  that guesses wrong about one row is worse than one that never tried.

## Licence

Public domain (CC0). It vendors nothing; each package carries its own licence.
