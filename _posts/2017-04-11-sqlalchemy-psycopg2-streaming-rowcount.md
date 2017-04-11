---
title: "PostgreSQL rowcount with SQLAlchemy/psycopg2 Streaming Cursor"
layout: "post"
permalink: "/2017/04/sqlalchemy-psycopg2-streaming-rowcount.html"
tags: ["sqlalchemy", "postgresql"]
---

SQLAlchemy [`ResultProxy.rowcount`](http://docs.sqlalchemy.org/en/latest/core/connections.html?highlight=rowcount#sqlalchemy.engine.ResultProxy.rowcount) does work with `SELECT` statements when using psycopg2 with PostgreSQL, despite the warnings. This is because psycopg2 uses libpq [`PQexec`](https://www.postgresql.org/docs/current/static/libpq-async.html) along with [`PQcmdTuples`](https://www.postgresql.org/docs/current/static/libpq-exec.html) to retreive the result count (`PQexec` always collects the command's entire result, buffering it in a single `PGresult`).

Using SQLAlchemy [`stream_results`](http://docs.sqlalchemy.org/en/latest/core/connections.html#sqlalchemy.engine.Connection.execution_options.params.stream_results) causes psycopg2 to use a named [server-side cursor](http://initd.org/psycopg/docs/usage.html#server-side-cursors) via PostgreSQL [`DECLARE`](https://www.postgresql.org/docs/current/static/ecpg-sql-declare.html). This will stream result records on demand, but `ResultProxy.rowcount` will not reflect the total result count.

To workaround this you can configure the psycopg2 server-side cursor to be [`scrollable`](http://initd.org/psycopg/docs/cursor.html#cursor.scrollable) (this allows moving backwards in the resultset). Then after the streaming query, execute [`MOVE FORWARD ALL`](https://www.postgresql.org/docs/current/static/sql-move.html) to move to the end of the results without fetching any. `PQcmdTuples` will then set the pscyopg2 rowcount and you can then scroll absolute back to the beginning of the results and process them streaming.

```python
>>> import sqlalchemy as sa
>>>
>>> table = sa.Table("testable", sa.MetaData(),
...                  sa.Column("_id", sa.Integer, primary_key=True),
...                  sa.Column("value", sa.String))
>>>
>>> engine = sa.create_engine("postgresql://scott:tiger@localhost:5432/mydatabase")
>>> table.drop(engine, checkfirst=True)
>>> table.create(engine, checkfirst=True)
>>> values = ["frog", "horse", "frog", "fish", "frog", "dog", "cow", "cat", "frog"]
>>> engine.execute(table.insert(), [{"_id": i, "value": v} for i, v in enumerate(values)])
<sqlalchemy.engine.result.ResultProxy object at 0x7f2e04d6cc50>
>>>
>>> query = table.select().where(table.c.value == "frog")
>>>
>>> print "non-streaming rowcount", engine.execute(query).rowcount
non-streaming rowcount 4
>>>
>>> with engine.connect().execution_options(stream_results=True) as conn:
...     results = conn.execute(query)
...     print "incorrect streaming rowcount", results.rowcount
...     for r in results:
...         print r
...
incorrect streaming rowcount 1
(0, u'frog')
(2, u'frog')
(4, u'frog')
(8, u'frog')
>>>
>>> with engine.connect() as conn:
...     def before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
...         cursor.scrollable = True
...     streamconn = conn.execution_options(stream_results=True)
...     sa.event.listen(streamconn, "before_cursor_execute", before_cursor_execute)
...     results = streamconn.execute(query)
...     r = conn.execute("MOVE FORWARD ALL FROM {}".format(results.cursor.name))
...     print "correct streaming rowcount", r.rowcount + results.rowcount
...     results.cursor.scroll(results.rowcount, mode="absolute")
...     for r in results:
...         print r
...
correct streaming rowcount 4
(0, u'frog')
(2, u'frog')
(4, u'frog')
(8, u'frog')
```