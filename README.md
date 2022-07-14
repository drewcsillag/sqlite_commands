# sqlite_commands

While I've also got [sqlp](https://github.com/drewcsillag/sqlp), there
have been times where it's not been available for use, or weird things
have constrained things, such as the sqlite compiled into python being
"undercompiled" or "underfeatured", such that it's not been ideal in
all cases.

This was more of a way to think "If SQLite could have some notion of
stored procedures for at least command line use..."

So what can you do with this? 

All of this assumes you're using the `sqlite3` command-line tool.
First you have to initialize the api:

`.read api_init.sql` to make sure the api is initialized.

**N.B.** it does mess with `.mode` settings so after an api call,
you'll need to reset it if you vary from `list`. Alternatively, you
can just hack `api.sql` to call `.mode` with your desired mode. I may
decide to go crazy and figure a way to have it remember.

## Load a file into a table where each row is a line

This will load the contents of the file `data.json` into a table named `log`
(which it will create if it doesn't already exist) and put in one row per
line. The table has two columns `name` and `line` and `name` is the name of 
the file it was loaded from and `line` is the line from the file.

```
insert into api._call (func, arg1, arg2) values ('loadlines', 'data.json', 'log');
.read api.sql
```

## Explode a column of json arrays into rows
This will take arrays from the table and column named in arg1 and arg2 respectively, and create
a new table (arg3) with a single column named `value` that has each of the array items in a separate row.

TODO: like `json_explode_obj`, it should track `src_rowid`
```
sqlite> create table foo (data);
sqlite> insert into foo (data) values ('[1,2]'), ('[3,4]');
sqlite> insert into api._call (func, arg1, arg2 ,arg3) values ('json_explode_arr', 'foo', 'data', 'exfoo');
sqlite> .read api.sql
sqlite> select * from exfoo;
+-------+
| value |
+-------+
| 1     |
| 2     |
| 3     |
| 4     |
+-------+
```

## Explode a column of json objects into a table where every top level key is column

The `src_rowid` column has the value of the `rowid` column from where the data originated from.
```
sqlite> create table bar (data);
sqlite> insert into bar (data) values ('{"a": 5, "b": 6}'), ('{"b": 7, "c":8}');
sqlite> insert into api._call(func, arg1, arg2, arg3) values ('json_explode_obj', 'bar', 'data', 'exbar');
sqlite> .read api.sql
sqlite> select * from exbar;
+-----------+---+---+---+
| src_rowid | a | b | c |
+-----------+---+---+---+
| 1         | 5 | 6 |   |
| 2         |   | 7 | 8 |
+-----------+---+---+---+
```

## Print a value
```
sqlite> insert into api._call (func, arg1) values ('print', 'HELLO!');
sqlite> .read api.sql
HELLO!
```

## Do nothing

Just does nothing.
```
sqlite> insert into api._call (func) values ('noop');
sqlite> .read api.sql
```

# How does it work?

In short, by making heavy use of `.once`, generating more sql with a
`SELECT`, and then calling `.read` with the file specified in
`.once`. The simplest case is in `api.sql` itself.  It calls `.once
docall.out` and using a `SELECT` generates a call to `.read` with a
generated filename based on func. More specifically, it just prepends
it with `call_` and suffixes it with `.sql`.

The downstream commands, except `noop` then basically repeat the
pattern, doing `.once docall2.out`, and using a `SELECT` to generate
more sql that it later `.read`s. In the case of `json_explode_obj`, it
actually goes one level deeper to do all it does.

I suspect you could get to full recursion if you wrote to the same
named file at every level, but so far haven't tried it.

# TODOs

As part of API initialization, I'd like to be able to have the
"library" in a different directory. You'd have to insert this info
somewhere, but that would allow the library to properly locate other
library files as needed.