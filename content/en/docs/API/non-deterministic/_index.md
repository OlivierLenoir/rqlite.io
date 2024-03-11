---
title: "Non-deterministic functions"
linkTitle: "Non-deterministic functions"
description: "How rqlite handles non-deterministic functions"
weight: 35
date: 2017-01-05
---

## Understanding the problem
rqlite peforms _statement-based replication_. This means that every SQL statement is usually stored in the Raft log exactly in the form it was received. Each rqlite node then reads the Raft log and applies the SQL statements it finds there to its own local copy of SQLite.

But if a SQL statement contains a [non-deterministic function](https://www.sqlite.org/deterministic.html), this type of replication can result in different SQLite data under each node -- which is not meant to happen. For example, the following statement could result in a different SQLite database under each node:
```
INSERT INTO foo (n) VALUES(random());
```
This is because `RANDOM()` is evaluated by each node independently, and `RANDOM()` will almost certainly return a different value on each node.

## How rqlite solves this problem
An rqlite node addresses this issue by _rewriting_ received SQL statements that contain certain non-deterministic functions, evaluating the non-determinstic factor, before writing the statement to the Raft log. The rewritten statement is then applied to the SQLite database as usual.

## What does rqlite rewrite?

### `RANDOM()`

Any SQL statement containing `RANDOM()` is rewritten following these rules:
- The statement is part of a write-request i.e. the request is sent to the `/db/execute` HTTP API.
- The statement is part of a read-request i.e. the request is sent to the `/db/query` HTTP API **and** the read-request is made with _strong_ read consistency.
- If `RANDOM()` is used as an `ORDER BY` qualifier it is not rewritten.
  - This does mean that certain `INSERT` statements are not rewritten e.g. `INSERT INTO foo (x) SELECT x FROM bar ORDER BY RANDOM()`. Executing such a statement may result in different data under each node. 
- The HTTP request containing the SQL statement does not have the query parameter `norwrandom` present.

`RANDOM()` is replaced with a random integer between -9223372036854775808 and +9223372036854775807 by the rqlite node that first receives the SQL statement.

#### Examples
```bash
# Will be rewritten
curl -XPOST 'localhost:4001/db/execute' -H "Content-Type: application/json" -d '[
    "INSERT INTO foo(id, age) VALUES(1234, RANDOM())"
]'

# RANDOM() rewriting explicitly disabled at request-time
curl -XPOST 'localhost:4001/db/execute?norwrandom' -H "Content-Type: application/json" -d '[
    "INSERT INTO foo(id, age) VALUES(1234, RANDOM())"
]' 

# Not rewritten
curl -G 'localhost:4001/db/query' --data-urlencode 'q=SELECT * FROM foo WHERE id = RANDOM()'

# Rewritten
curl -G 'localhost:4001/db/query?level=strong' --data-urlencode 'q=SELECT * FROM foo WHERE id = RANDOM()'
```

### Date and time functions
rqlite does not yet rewrite [SQLite date and time functions](https://www.sqlite.org/lang_datefunc.html) that are non-deterministic in nature, but will in an upcoming release. A example of a non-deterministic time function is

`INSERT INTO datetime_text (d1, d2) VALUES(datetime('now'),datetime('now', 'localtime'))`

Using such functions will result in undefined behavior. Date and time functions that use absolute values will work without issue.

#### CURRENT_TIME*
Using `CURRENT_TIMESTAMP`, `CURRENT_TIME`, and `CURRENT_DATE` can also be problematic, depending on your use case.

## Credits
Many thanks to [Ben Johnson](https://github.com/benbjohnson) who wrote the SQLite parser used by rqlite.
