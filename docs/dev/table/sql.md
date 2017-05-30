---
title: "SQL"
nav-parent_id: tableapi
nav-pos: 30
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

SQL queries are specified using the `sql()` method of the `TableEnvironment`. The method returns the result of the SQL query as a `Table` which can be converted into a `DataSet` or `DataStream`, used in subsequent Table API queries, or written to a `TableSink` (see [Writing Tables to External Sinks](#writing-tables-to-external-sinks)). SQL and Table API queries can seamlessly mixed and are holistically optimized and translated into a single DataStream or DataSet program.

A `Table`, `DataSet`, `DataStream`, or external `TableSource` must be registered in the `TableEnvironment` in order to be accessible by a SQL query (see [Registering Tables](#registering-tables)). For convenience `Table.toString()` will automatically register an unique table name under the `Table`'s `TableEnvironment` and return the table name. So it allows to call SQL directly on tables in a string concatenation (see examples below).

*Note: Flink's SQL support is not feature complete, yet. Queries that include unsupported SQL features will cause a `TableException`. The limitations of SQL on batch and streaming tables are listed in the following sections.*

**TODO: Rework intro. Move some parts below. **

* This will be replaced by the TOC
{:toc}

Specifying a Query
---------------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// ingest a DataStream from an external source
DataStream<Tuple3<Long, String, Integer>> ds = env.addSource(...);

// call SQL on unregistered tables
Table table = tableEnv.toTable(ds, "user, product, amount");
Table result = tableEnv.sql(
  "SELECT SUM(amount) FROM " + table + " WHERE product LIKE '%Rubber%'");

// call SQL on registered tables
// register the DataStream as table "Orders"
tableEnv.registerDataStream("Orders", ds, "user, product, amount");
// run a SQL query on the Table and retrieve the result as a new Table
Table result2 = tableEnv.sql(
  "SELECT product, amount FROM Orders WHERE product LIKE '%Rubber%'");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// read a DataStream from an external source
val ds: DataStream[(Long, String, Integer)] = env.addSource(...)

// call SQL on unregistered tables
val table = ds.toTable(tableEnv, 'user, 'product, 'amount)
val result = tableEnv.sql(
  s"SELECT SUM(amount) FROM $table WHERE product LIKE '%Rubber%'")

// call SQL on registered tables
// register the DataStream under the name "Orders"
tableEnv.registerDataStream("Orders", ds, 'user, 'product, 'amount)
// run a SQL query on the Table and retrieve the result as a new Table
val result2 = tableEnv.sql(
  "SELECT product, amount FROM Orders WHERE product LIKE '%Rubber%'")
{% endhighlight %}
</div>
</div>

**TODO: Add some intro.**

{% top %}

Supported Syntax
----------------

Flink uses [Apache Calcite](https://calcite.apache.org/docs/reference.html) for SQL parsing. Currently, Flink SQL only supports query-related SQL syntax and only a subset of the comprehensive SQL standard. The following BNF-grammar describes the supported SQL features:

```

query:
  values
  | {
      select
      | selectWithoutFrom
      | query UNION [ ALL ] query
      | query EXCEPT query
      | query INTERSECT query
    }
    [ ORDER BY orderItem [, orderItem ]* ]
    [ LIMIT { count | ALL } ]
    [ OFFSET start { ROW | ROWS } ]
    [ FETCH { FIRST | NEXT } [ count ] { ROW | ROWS } ONLY]

orderItem:
  expression [ ASC | DESC ]

select:
  SELECT [ ALL | DISTINCT ]
  { * | projectItem [, projectItem ]* }
  FROM tableExpression
  [ WHERE booleanExpression ]
  [ GROUP BY { groupItem [, groupItem ]* } ]
  [ HAVING booleanExpression ]

selectWithoutFrom:
  SELECT [ ALL | DISTINCT ]
  { * | projectItem [, projectItem ]* }

projectItem:
  expression [ [ AS ] columnAlias ]
  | tableAlias . *

tableExpression:
  tableReference [, tableReference ]*
  | tableExpression [ NATURAL ] [ LEFT | RIGHT | FULL ] JOIN tableExpression [ joinCondition ]

joinCondition:
  ON booleanExpression
  | USING '(' column [, column ]* ')'

tableReference:
  tablePrimary
  [ [ AS ] alias [ '(' columnAlias [, columnAlias ]* ')' ] ]

tablePrimary:
  [ TABLE ] [ [ catalogName . ] schemaName . ] tableName
  | LATERAL TABLE '(' functionName '(' expression [, expression ]* ')' ')'
  | UNNEST '(' expression ')'

values:
  VALUES expression [, expression ]*

groupItem:
  expression
  | '(' ')'
  | '(' expression [, expression ]* ')'
  | CUBE '(' expression [, expression ]* ')'
  | ROLLUP '(' expression [, expression ]* ')'
  | GROUPING SETS '(' groupItem [, groupItem ]* ')'
```

For a better definition of SQL queries within a Java String, Flink SQL uses a lexical policy similar to Java:

- The case of identifiers is preserved whether or not they are quoted.
- After which, identifiers are matched case-sensitively.
- Unlike Java, back-ticks allow identifiers to contain non-alphanumeric characters (e.g. <code>"SELECT a AS `my field` FROM t"</code>).

{% top %}

Example Queries
---------------

**TODO: Add a examples for different operations with similar structure as for the Table API. Add highlighted tags if an operation is not supported by stream / batch.**

* Scan & Values
* Selection & Projection
* Aggregations (distinct only Batch)
  * GroupBy
  * GroupBy Windows (TUMBLE, HOP, SESSION)
  * OVER windows (Only Stream)
  * Grouping sets, rollup, cube (only batch)
  * Having (only batch?)
* Joins
  * Inner equi joins (only batch)
  * Outer equi joins (only batch)
  * TableFunction
* Set operations (only batch, except Union ALL)
* OrderBy + Limit + Offset

{% top %}

### GroupBy Windows

**TODO: Integrate this with the examples**

### Group Windows

Group windows are defined in the `GROUP BY` clause of a SQL query. Just like queries with regular `GROUP BY` clauses, queries with a `GROUP BY` clause that includes a group window function compute a single result row per group. The following group windows functions are supported for SQL on batch and streaming tables.

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 30%">Group Window Function</th>
      <th class="text-left">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><code>TUMBLE(time_attr, interval)</code></td>
      <td>Defines a tumbling time window. A tumbling time window assigns rows to non-overlapping, continuous windows with a fixed duration (<code>interval</code>). For example, a tumbling window of 5 minutes groups rows in 5 minutes intervals. Tumbling windows can be defined on event-time (stream + batch) or processing-time (stream).</td>
    </tr>
    <tr>
      <td><code>HOP(time_attr, interval, interval)</code></td>
      <td>Defines a hopping time window (called sliding window in the Table API). A hopping time window has a fixed duration (second <code>interval</code> parameter) and hops by a specified hop interval (first <code>interval</code> parameter). If the hop interval is smaller than the window size, hopping windows are overlapping. Thus, rows can be assigned to multiple windows. For example, a hopping window of 15 minutes size and 5 minute hop interval assigns each row to 3 different windows of 15 minute size, which are evaluated in an interval of 5 minutes. Hopping windows can be defined on event-time (stream + batch) or processing-time (stream).</td>
    </tr>
    <tr>
      <td><code>SESSION(time_attr, interval)</code></td>
      <td>Defines a session time window. Session time windows do not have a fixed duration but their bounds are defined by a time <code>interval</code> of inactivity, i.e., a session window is closed if no event appears for a defined gap period. For example a session window with a 30 minute gap starts when a row is observed after 30 minutes inactivity (otherwise the row would be added to an existing window) and is closed if no row is added within 30 minutes. Session windows can work on event-time (stream + batch) or processing-time (stream).</td>
    </tr>
  </tbody>
</table>

For SQL queries on streaming tables, the `time_attr` argument of the group window function must be one of the `rowtime()` or `proctime()` time-indicators, which distinguish between event or processing time, respectively. For SQL on batch tables, the `time_attr` argument of the group window function must be an attribute of type `TIMESTAMP`. 

#### Selecting Group Window Start and End Timestamps

The start and end timestamps of group windows can be selected with the following auxiliary functions:

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Auxiliary Function</th>
      <th class="text-left">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        <code>TUMBLE_START(time_attr, interval)</code><br/>
        <code>HOP_START(time_attr, interval, interval)</code><br/>
        <code>SESSION_START(time_attr, interval)</code><br/>
      </td>
      <td>Returns the start timestamp of the corresponding tumbling, hopping, and session window.</td>
    </tr>
    <tr>
      <td>
        <code>TUMBLE_END(time_attr, interval)</code><br/>
        <code>HOP_END(time_attr, interval, interval)</code><br/>
        <code>SESSION_END(time_attr, interval)</code><br/>
      </td>
      <td>Returns the end timestamp of the corresponding tumbling, hopping, and session window.</td>
    </tr>
  </tbody>
</table>

Note that the auxiliary functions must be called with exactly same arguments as the group window function in the `GROUP BY` clause.

The following examples show how to specify SQL queries with group windows on streaming tables. 

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// ingest a DataStream from an external source
DataStream<Tuple3<Long, String, Integer>> ds = env.addSource(...);
// register the DataStream as table "Orders"
tableEnv.registerDataStream("Orders", ds, "user, product, amount");

// compute SUM(amount) per day (in event-time)
Table result1 = tableEnv.sql(
  "SELECT user, " +
  "  TUMBLE_START(rowtime(), INTERVAL '1' DAY) as wStart,  " +
  "  SUM(amount) FROM Orders " + 
  "GROUP BY TUMBLE(rowtime(), INTERVAL '1' DAY), user");

// compute SUM(amount) per day (in processing-time)
Table result2 = tableEnv.sql(
  "SELECT user, SUM(amount) FROM Orders GROUP BY TUMBLE(proctime(), INTERVAL '1' DAY), user");

// compute every hour the SUM(amount) of the last 24 hours in event-time
Table result3 = tableEnv.sql(
  "SELECT product, SUM(amount) FROM Orders GROUP BY HOP(rowtime(), INTERVAL '1' HOUR, INTERVAL '1' DAY), product");

// compute SUM(amount) per session with 12 hour inactivity gap (in event-time)
Table result4 = tableEnv.sql(
  "SELECT user, " +
  "  SESSION_START(rowtime(), INTERVAL '12' HOUR) AS sStart, " +
  "  SESSION_END(rowtime(), INTERVAL '12' HOUR) AS snd, " + 
  "  SUM(amount) " + 
  "FROM Orders " + 
  "GROUP BY SESSION(rowtime(), INTERVAL '12' HOUR), user");

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// read a DataStream from an external source
val ds: DataStream[(Long, String, Int)] = env.addSource(...)
// register the DataStream under the name "Orders"
tableEnv.registerDataStream("Orders", ds, 'user, 'product, 'amount)

// compute SUM(amount) per day (in event-time)
val result1 = tableEnv.sql(
    """
      |SELECT
      |  user, 
      |  TUMBLE_START(rowtime(), INTERVAL '1' DAY) as wStart,
      |  SUM(amount)
      | FROM Orders
      | GROUP BY TUMBLE(rowtime(), INTERVAL '1' DAY), user
    """.stripMargin)

// compute SUM(amount) per day (in processing-time)
val result2 = tableEnv.sql(
  "SELECT user, SUM(amount) FROM Orders GROUP BY TUMBLE(proctime(), INTERVAL '1' DAY), user")

// compute every hour the SUM(amount) of the last 24 hours in event-time
val result3 = tableEnv.sql(
  "SELECT product, SUM(amount) FROM Orders GROUP BY HOP(rowtime(), INTERVAL '1' HOUR, INTERVAL '1' DAY), product")

// compute SUM(amount) per session with 12 hour inactivity gap (in event-time)
val result4 = tableEnv.sql(
    """
      |SELECT
      |  user, 
      |  SESSION_START(rowtime(), INTERVAL '12' HOUR) AS sStart,
      |  SESSION_END(rowtime(), INTERVAL '12' HOUR) AS sEnd,
      |  SUM(amount)
      | FROM Orders
      | GROUP BY SESSION(rowtime(), INTERVAL '12' HOUR), user
    """.stripMargin)

{% endhighlight %}
</div>
</div>

{% top %}

### Limitations

**TODO: Integrate this with the examples**

#### Batch

The current version supports selection (filter), projection, inner equi-joins, grouping, aggregates, and sorting on batch tables.

Among others, the following SQL features are not supported, yet:

- Timestamps and intervals are limited to milliseconds precision
- Interval arithmetic is currenly limited
- Non-equi joins and Cartesian products
- Efficient grouping sets

*Note: Tables are joined in the order in which they are specified in the `FROM` clause. In some cases the table order must be manually tweaked to resolve Cartesian products.*

#### Streaming

Joins, set operations, and non-windowed aggregations are not supported yet.
`UNNEST` supports only arrays and does not support `WITH ORDINALITY` yet.

Data Types
----------

The SQL runtime is built on top of Flink's DataSet and DataStream APIs. Internally, it also uses Flink's `TypeInformation` to distinguish between types. The SQL support does not include all Flink types so far. All supported simple types are listed in `org.apache.flink.table.api.Types`. The following table summarizes the relation between SQL Types, Table API types, and the resulting Java class.

| Table API              | SQL                         | Java type              |
| :--------------------- | :-------------------------- | :--------------------- |
| `Types.STRING`         | `VARCHAR`                   | `java.lang.String`     |
| `Types.BOOLEAN`        | `BOOLEAN`                   | `java.lang.Boolean`    |
| `Types.BYTE`           | `TINYINT`                   | `java.lang.Byte`       |
| `Types.SHORT`          | `SMALLINT`                  | `java.lang.Short`      |
| `Types.INT`            | `INTEGER, INT`              | `java.lang.Integer`    |
| `Types.LONG`           | `BIGINT`                    | `java.lang.Long`       |
| `Types.FLOAT`          | `REAL, FLOAT`               | `java.lang.Float`      |
| `Types.DOUBLE`         | `DOUBLE`                    | `java.lang.Double`     |
| `Types.DECIMAL`        | `DECIMAL`                   | `java.math.BigDecimal` |
| `Types.DATE`           | `DATE`                      | `java.sql.Date`        |
| `Types.TIME`           | `TIME`                      | `java.sql.Time`        |
| `Types.TIMESTAMP`      | `TIMESTAMP(3)`              | `java.sql.Timestamp`   |
| `Types.INTERVAL_MONTHS`| `INTERVAL YEAR TO MONTH`    | `java.lang.Integer`    |
| `Types.INTERVAL_MILLIS`| `INTERVAL DAY TO SECOND(3)` | `java.lang.Long`       |
| `Types.PRIMITIVE_ARRAY`| `ARRAY`                     | e.g. `int[]`           |
| `Types.OBJECT_ARRAY`   | `ARRAY`                     | e.g. `java.lang.Byte[]`|
| `Types.MAP`            | `MAP`                       | `java.util.HashMap`    |


Advanced types such as generic types, composite types (e.g. POJOs or Tuples), and array types (object or primitive arrays) can be fields of a row. 

Generic types are treated as a black box within Table API and SQL yet.

Composite types, however, are fully supported types where fields of a composite type can be accessed using the `.get()` operator in Table API and dot operator (e.g. `MyTable.pojoColumn.myField`) in SQL. Composite types can also be flattened using `.flatten()` in Table API or `MyTable.pojoColumn.*` in SQL.

Array types can be accessed using the `myArray.at(1)` operator in Table API and `myArray[1]` operator in SQL. Array literals can be created using `array(1, 2, 3)` in Table API and `ARRAY[1, 2, 3]` in SQL.

{% top %}

Built-In Functions
------------------

Both the Table API and SQL come with a set of built-in functions for data transformations. This section gives a brief overview of the available functions so far.

<!--
This list of SQL functions should be kept in sync with SqlExpressionTest to reduce confusion due to the large amount of SQL functions.
The documentation is split up and ordered like the tests in SqlExpressionTest.
-->

The Flink SQL functions (including their syntax) are a subset of Apache Calcite's built-in functions. Most of the documentation has been adopted from the [Calcite SQL reference](https://calcite.apache.org/docs/reference.html).


<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Comparison functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        {% highlight text %}
value1 = value2
{% endhighlight %}
      </td>
      <td>
        <p>Equals.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 <> value2
{% endhighlight %}
      </td>
      <td>
        <p>Not equal.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 > value2
{% endhighlight %}
      </td>
      <td>
        <p>Greater than.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 >= value2
{% endhighlight %}
      </td>
      <td>
        <p>Greater than or equal.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 < value2
{% endhighlight %}
      </td>
      <td>
        <p>Less than.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 <= value2
{% endhighlight %}
      </td>
      <td>
        <p>Less than or equal.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value IS NULL
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>value</i> is null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value IS NOT NULL
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>value</i> is not null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 IS DISTINCT FROM value2
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if two values are not equal, treating null values as the same.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 IS NOT DISTINCT FROM value2
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if two values are equal, treating null values as the same.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 BETWEEN [ASYMMETRIC | SYMMETRIC] value2 AND value3
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>value1</i> is greater than or equal to <i>value2</i> and less than or equal to <i>value3</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value1 NOT BETWEEN value2 AND value3
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>value1</i> is less than <i>value2</i> or greater than <i>value3</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
string1 LIKE string2 [ ESCAPE string3 ]
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>string1</i> matches pattern <i>string2</i>. An escape character can be defined if necessary.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
string1 NOT LIKE string2 [ ESCAPE string3 ]
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>string1</i> does not match pattern <i>string2</i>. An escape character can be defined if necessary.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
string1 SIMILAR TO string2 [ ESCAPE string3 ]
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>string1</i> matches regular expression <i>string2</i>. An escape character can be defined if necessary.</p>
      </td>
    </tr>


    <tr>
      <td>
        {% highlight text %}
string1 NOT SIMILAR TO string2 [ ESCAPE string3 ]
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>string1</i> does not match regular expression <i>string2</i>. An escape character can be defined if necessary.</p>
      </td>
    </tr>


    <tr>
      <td>
        {% highlight text %}
value IN (value [, value]* )
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>value</i> is equal to a value in a list.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value NOT IN (value [, value]* )
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>value</i> is not equal to every value in a list.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
EXISTS (sub-query)
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>sub-query</i> returns at least one row. Only supported if the operation can be rewritten in a join and group operation.</p>
      </td>
    </tr>

<!-- NOT SUPPORTED SO FAR
    <tr>
      <td>
        {% highlight text %}
value IN (sub-query)
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>value</i> is equal to a row returned by sub-query.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
value NOT IN (sub-query)
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>value</i> is not equal to every row returned by sub-query.</p>
      </td>
    </tr>
    -->

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Logical functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        {% highlight text %}
boolean1 OR boolean2
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>boolean1</i> is TRUE or <i>boolean2</i> is TRUE. Supports three-valued logic.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean1 AND boolean2
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>boolean1</i> and <i>boolean2</i> are both TRUE. Supports three-valued logic.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
NOT boolean
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>boolean</i> is not TRUE; returns UNKNOWN if <i>boolean</i> is UNKNOWN.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean IS FALSE
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>boolean</i> is FALSE; returns FALSE if <i>boolean</i> is UNKNOWN.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean IS NOT FALSE
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>boolean</i> is not FALSE; returns TRUE if <i>boolean</i> is UNKNOWN.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean IS TRUE
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>boolean</i> is TRUE; returns FALSE if <i>boolean</i> is UNKNOWN.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean IS NOT TRUE
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>boolean</i> is not TRUE; returns TRUE if <i>boolean</i> is UNKNOWN.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean IS UNKNOWN
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>boolean</i> is UNKNOWN.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
boolean IS NOT UNKNOWN
{% endhighlight %}
      </td>
      <td>
        <p>Returns TRUE if <i>boolean</i> is not UNKNOWN.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Arithmetic functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        {% highlight text %}
+ numeric
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
- numeric
{% endhighlight %}
      </td>
      <td>
        <p>Returns negative <i>numeric</i>.</p>
      </td>
    </tr>
    
    <tr>
      <td>
        {% highlight text %}
numeric1 + numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> plus <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
numeric1 - numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> minus <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
numeric1 * numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> multiplied by <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
numeric1 / numeric2
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> divided by <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
POWER(numeric1, numeric2)
{% endhighlight %}
      </td>
      <td>
        <p>Returns <i>numeric1</i> raised to the power of <i>numeric2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ABS(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the absolute value of <i>numeric</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
MOD(numeric1, numeric2)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the remainder (modulus) of <i>numeric1</i> divided by <i>numeric2</i>. The result is negative only if <i>numeric1</i> is negative.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
SQRT(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the square root of <i>numeric</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
LN(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the natural logarithm (base e) of <i>numeric</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
LOG10(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the base 10 logarithm of <i>numeric</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
EXP(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Returns e raised to the power of <i>numeric</i>.</p>
      </td>
    </tr>   

    <tr>
      <td>
        {% highlight text %}
CEIL(numeric)
CEILING(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Rounds <i>numeric</i> up, and returns the smallest number that is greater than or equal to <i>numeric</i>.</p>
      </td>
    </tr>  

    <tr>
      <td>
        {% highlight text %}
FLOOR(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Rounds <i>numeric</i> down, and returns the largest number that is less than or equal to <i>numeric</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
SIN(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the sine of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
COS(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the cosine of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
TAN(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the tangent of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
COT(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the cotangent of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ASIN(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the arc sine of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ACOS(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the arc cosine of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ATAN(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the arc tangent of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
DEGREES(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Converts <i>numeric</i> from radians to degrees.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
RADIANS(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Converts <i>numeric</i> from degrees to radians.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
SIGN(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Calculates the signum of a given number.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ROUND(numeric, int)
{% endhighlight %}
      </td>
      <td>
        <p>Rounds the given number to <i>integer</i> places right to the decimal point.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
PI()
{% endhighlight %}
      </td>
      <td>
        <p>Returns a value that is closer than any other value to pi.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">String functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        {% highlight text %}
string || string
{% endhighlight %}
      </td>
      <td>
        <p>Concatenates two character strings.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CHAR_LENGTH(string)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the number of characters in a character string.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CHARACTER_LENGTH(string)
{% endhighlight %}
      </td>
      <td>
        <p>As CHAR_LENGTH(<i>string</i>).</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
UPPER(string)
{% endhighlight %}
      </td>
      <td>
        <p>Returns a character string converted to upper case.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
LOWER(string)
{% endhighlight %}
      </td>
      <td>
        <p>Returns a character string converted to lower case.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
POSITION(string1 IN string2)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the position of the first occurrence of <i>string1</i> in <i>string2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
TRIM( { BOTH | LEADING | TRAILING } string1 FROM string2)
{% endhighlight %}
      </td>
      <td>
        <p>Removes leading and/or trailing characters from <i>string2</i>. By default, whitespaces at both sides are removed.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
OVERLAY(string1 PLACING string2 FROM integer [ FOR integer2 ])
{% endhighlight %}
      </td>
      <td>
        <p>Replaces a substring of <i>string1</i> with <i>string2</i>.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
SUBSTRING(string FROM integer)
{% endhighlight %}
      </td>
      <td>
        <p>Returns a substring of a character string starting at a given point.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
SUBSTRING(string FROM integer FOR integer)
{% endhighlight %}
      </td>
      <td>
        <p>Returns a substring of a character string starting at a given point with a given length.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
INITCAP(string)
{% endhighlight %}
      </td>
      <td>
        <p>Returns string with the first letter of each word converter to upper case and the rest to lower case. Words are sequences of alphanumeric characters separated by non-alphanumeric characters.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Conditional functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        {% highlight text %}
CASE value
WHEN value1 [, value11 ]* THEN result1
[ WHEN valueN [, valueN1 ]* THEN resultN ]*
[ ELSE resultZ ]
END
{% endhighlight %}
      </td>
      <td>
        <p>Simple case.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CASE
WHEN condition1 THEN result1
[ WHEN conditionN THEN resultN ]*
[ ELSE resultZ ]
END
{% endhighlight %}
      </td>
      <td>
        <p>Searched case.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
NULLIF(value, value)
{% endhighlight %}
      </td>
      <td>
        <p>Returns NULL if the values are the same. For example, <code>NULLIF(5, 5)</code> returns NULL; <code>NULLIF(5, 0)</code> returns 5.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
COALESCE(value, value [, value ]* )
{% endhighlight %}
      </td>
      <td>
        <p>Provides a value if the first value is null. For example, <code>COALESCE(NULL, 5)</code> returns 5.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Type conversion functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        {% highlight text %}
CAST(value AS type)
{% endhighlight %}
      </td>
      <td>
        <p>Converts a value to a given type.</p>
      </td>
    </tr>
  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Value constructor functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
  <!-- Disabled temporarily in favor of composite type support
    <tr>
      <td>
        {% highlight text %}
ROW (value [, value]* )
{% endhighlight %}
      </td>
      <td>
        <p>Creates a row from a list of values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
(value [, value]* )
{% endhighlight %}
      </td>
      <td>
        <p>Creates a row from a list of values.</p>
      </td>
    </tr>
-->

    <tr>
      <td>
        {% highlight text %}
array ‘[’ index ‘]’
{% endhighlight %}
      </td>
      <td>
        <p>Returns the element at a particular position in an array. The index starts at 1.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ARRAY ‘[’ value [, value ]* ‘]’
{% endhighlight %}
      </td>
      <td>
        <p>Creates an array from a list of values.</p>
      </td>
    </tr>

  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Temporal functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        {% highlight text %}
DATE string
{% endhighlight %}
      </td>
      <td>
        <p>Parses a date string in the form "yy-mm-dd" to a SQL date.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
TIME string
{% endhighlight %}
      </td>
      <td>
        <p>Parses a time <i>string</i> in the form "hh:mm:ss" to a SQL time.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
TIMESTAMP string
{% endhighlight %}
      </td>
      <td>
        <p>Parses a timestamp <i>string</i> in the form "yy-mm-dd hh:mm:ss.fff" to a SQL timestamp.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
INTERVAL string range
{% endhighlight %}
      </td>
      <td>
        <p>Parses an interval <i>string</i> in the form "dd hh:mm:ss.fff" for SQL intervals of milliseconds or "yyyy-mm" for SQL intervals of months. An interval range might be e.g. <code>DAY</code>, <code>MINUTE</code>, <code>DAY TO HOUR</code>, or <code>DAY TO SECOND</code> for intervals of milliseconds; <code>YEAR</code> or <code>YEAR TO MONTH</code> for intervals of months. E.g. <code>INTERVAL '10 00:00:00.004' DAY TO SECOND</code>, <code>INTERVAL '10' DAY</code>, or <code>INTERVAL '2-10' YEAR TO MONTH</code> return intervals.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CURRENT_DATE
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL date in UTC time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CURRENT_TIME
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL time in UTC time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CURRENT_TIMESTAMP
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL timestamp in UTC time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
LOCALTIME
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL time in local time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
LOCALTIMESTAMP
{% endhighlight %}
      </td>
      <td>
        <p>Returns the current SQL timestamp in local time zone.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
EXTRACT(timeintervalunit FROM temporal)
{% endhighlight %}
      </td>
      <td>
        <p>Extracts parts of a time point or time interval. Returns the part as a long value. E.g. <code>EXTRACT(DAY FROM DATE '2006-06-05')</code> leads to 5.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
FLOOR(timepoint TO timeintervalunit)
{% endhighlight %}
      </td>
      <td>
        <p>Rounds a time point down to the given unit. E.g. <code>FLOOR(TIME '12:44:31' TO MINUTE)</code> leads to 12:44:00.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
CEIL(timepoint TO timeintervalunit)
{% endhighlight %}
      </td>
      <td>
        <p>Rounds a time point up to the given unit. E.g. <code>CEIL(TIME '12:44:31' TO MINUTE)</code> leads to 12:45:00.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
QUARTER(date)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the quarter of a year from a SQL date. E.g. <code>QUARTER(DATE '1994-09-27')</code> leads to 3.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
(timepoint, temporal) OVERLAPS (timepoint, temporal)
{% endhighlight %}
      </td>
      <td>
        <p>Determines whether two anchored time intervals overlap. Time point and temporal are transformed into a range defined by two time points (start, end). The function evaluates <code>leftEnd >= rightStart && rightEnd >= leftStart</code>. E.g. <code>(TIME '2:55:00', INTERVAL '1' HOUR) OVERLAPS (TIME '3:30:00', INTERVAL '2' HOUR)</code> leads to true; <code>(TIME '9:00:00', TIME '10:00:00') OVERLAPS (TIME '10:15:00', INTERVAL '3' HOUR)</code> leads to false.</p>
      </td>
    </tr>
  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Aggregate functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        {% highlight text %}
COUNT(value [, value]* )
{% endhighlight %}
      </td>
      <td>
        <p>Returns the number of input rows for which <i>value</i> is not null.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
COUNT(*)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the number of input rows.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
AVG(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the average (arithmetic mean) of <i>numeric</i> across all input values.</p>
      </td>
    </tr>
    
    <tr>
      <td>
        {% highlight text %}
SUM(numeric)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sum of <i>numeric</i> across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
MAX(value)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the maximum value of <i>value</i> across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
MIN(value)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the minimum value of <i>value</i> across all input values.</p>
      </td>
    </tr>
    <tr>
      <td>
        {% highlight text %}
STDDEV_POP(value)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the population standard deviation of the numeric field across all input values.</p>
      </td>
    </tr>
    
<tr>
      <td>
        {% highlight text %}
STDDEV_SAMP(value)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sample standard deviation of the numeric field across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
VAR_POP(value)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the population variance (square of the population standard deviation) of the numeric field across all input values.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
VAR_SAMP(value)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sample variance (square of the sample standard deviation) of the numeric field across all input values.</p>
      </td>
    </tr>
  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Grouping functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        {% highlight text %}
GROUP_ID()
{% endhighlight %}
      </td>
      <td>
        <p>Returns an integer that uniquely identifies the combination of grouping keys.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
GROUPING(expression)
{% endhighlight %}
      </td>
      <td>
        <p>Returns 1 if <i>expression</i> is rolled up in the current row’s grouping set, 0 otherwise.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
GROUPING_ID(expression [, expression]* )
{% endhighlight %}
      </td>
      <td>
        <p>Returns a bit vector of the given grouping expressions.</p>
      </td>
    </tr>
  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Value access functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        {% highlight text %}
tableName.compositeType.field
{% endhighlight %}
      </td>
      <td>
        <p>Accesses the field of a Flink composite type (such as Tuple, POJO, etc.) by name and returns it's value.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
tableName.compositeType.*
{% endhighlight %}
      </td>
      <td>
        <p>Converts a Flink composite type (such as Tuple, POJO, etc.) and all of its direct subtypes into a flat representation where every subtype is a separate field.</p>
      </td>
    </tr>
  </tbody>
</table>

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Array functions</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>
        {% highlight text %}
CARDINALITY(ARRAY)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the number of elements of an array.</p>
      </td>
    </tr>

    <tr>
      <td>
        {% highlight text %}
ELEMENT(ARRAY)
{% endhighlight %}
      </td>
      <td>
        <p>Returns the sole element of an array with a single element. Returns <code>null</code> if the array is empty. Throws an exception if the array has more than one element.</p>
      </td>
    </tr>
  </tbody>
</table>

### Limitations

The following operations are not supported yet:

- Binary string operators and functions
- System functions
- Collection functions
- Aggregate functions like STDDEV_xxx, VAR_xxx, and REGR_xxx
- Distinct aggregate functions like COUNT DISTINCT

{% top %}

Reserved Keywords
-----------------

Although not every SQL feature is implemented yet, some string combinations are already reserved as keywords for future use. If you want to use one of the following strings as a field name, make sure to surround them with backticks (e.g. `` `value` ``, `` `count` ``).

{% highlight sql %}

A, ABS, ABSOLUTE, ACTION, ADA, ADD, ADMIN, AFTER, ALL, ALLOCATE, ALLOW, ALTER, ALWAYS, AND, ANY, ARE, ARRAY, AS, ASC, ASENSITIVE, ASSERTION, ASSIGNMENT, ASYMMETRIC, AT, ATOMIC, ATTRIBUTE, ATTRIBUTES, AUTHORIZATION, AVG, BEFORE, BEGIN, BERNOULLI, BETWEEN, BIGINT, BINARY, BIT, BLOB, BOOLEAN, BOTH, BREADTH, BY, C, CALL, CALLED, CARDINALITY, CASCADE, CASCADED, CASE, CAST, CATALOG, CATALOG_NAME, CEIL, CEILING, CENTURY, CHAIN, CHAR, CHARACTER, CHARACTERISTICTS, CHARACTERS, CHARACTER_LENGTH, CHARACTER_SET_CATALOG, CHARACTER_SET_NAME, CHARACTER_SET_SCHEMA, CHAR_LENGTH, CHECK, CLASS_ORIGIN, CLOB, CLOSE, COALESCE, COBOL, COLLATE, COLLATION, COLLATION_CATALOG, COLLATION_NAME, COLLATION_SCHEMA, COLLECT, COLUMN, COLUMN_NAME, COMMAND_FUNCTION, COMMAND_FUNCTION_CODE, COMMIT, COMMITTED, CONDITION, CONDITION_NUMBER, CONNECT, CONNECTION, CONNECTION_NAME, CONSTRAINT, CONSTRAINTS, CONSTRAINT_CATALOG, CONSTRAINT_NAME, CONSTRAINT_SCHEMA, CONSTRUCTOR, CONTAINS, CONTINUE, CONVERT, CORR, CORRESPONDING, COUNT, COVAR_POP, COVAR_SAMP, CREATE, CROSS, CUBE, CUME_DIST, CURRENT, CURRENT_CATALOG, CURRENT_DATE, CURRENT_DEFAULT_TRANSFORM_GROUP, CURRENT_PATH, CURRENT_ROLE, CURRENT_SCHEMA, CURRENT_TIME, CURRENT_TIMESTAMP, CURRENT_TRANSFORM_GROUP_FOR_TYPE, CURRENT_USER, CURSOR, CURSOR_NAME, CYCLE, DATA, DATABASE, DATE, DATETIME_INTERVAL_CODE, DATETIME_INTERVAL_PRECISION, DAY, DEALLOCATE, DEC, DECADE, DECIMAL, DECLARE, DEFAULT, DEFAULTS, DEFERRABLE, DEFERRED, DEFINED, DEFINER, DEGREE, DELETE, DENSE_RANK, DEPTH, DEREF, DERIVED, DESC, DESCRIBE, DESCRIPTION, DESCRIPTOR, DETERMINISTIC, DIAGNOSTICS, DISALLOW, DISCONNECT, DISPATCH, DISTINCT, DOMAIN, DOUBLE, DOW, DOY, DROP, DYNAMIC, DYNAMIC_FUNCTION, DYNAMIC_FUNCTION_CODE, EACH, ELEMENT, ELSE, END, END-EXEC, EPOCH, EQUALS, ESCAPE, EVERY, EXCEPT, EXCEPTION, EXCLUDE, EXCLUDING, EXEC, EXECUTE, EXISTS, EXP, EXPLAIN, EXTEND, EXTERNAL, EXTRACT, FALSE, FETCH, FILTER, FINAL, FIRST, FIRST_VALUE, FLOAT, FLOOR, FOLLOWING, FOR, FOREIGN, FORTRAN, FOUND, FRAC_SECOND, FREE, FROM, FULL, FUNCTION, FUSION, G, GENERAL, GENERATED, GET, GLOBAL, GO, GOTO, GRANT, GRANTED, GROUP, GROUPING, HAVING, HIERARCHY, HOLD, HOUR, IDENTITY, IMMEDIATE, IMPLEMENTATION, IMPORT, IN, INCLUDING, INCREMENT, INDICATOR, INITIALLY, INNER, INOUT, INPUT, INSENSITIVE, INSERT, INSTANCE, INSTANTIABLE, INT, INTEGER, INTERSECT, INTERSECTION, INTERVAL, INTO, INVOKER, IS, ISOLATION, JAVA, JOIN, K, KEY, KEY_MEMBER, KEY_TYPE, LABEL, LANGUAGE, LARGE, LAST, LAST_VALUE, LATERAL, LEADING, LEFT, LENGTH, LEVEL, LIBRARY, LIKE, LIMIT, LN, LOCAL, LOCALTIME, LOCALTIMESTAMP, LOCATOR, LOWER, M, MAP, MATCH, MATCHED, MAX, MAXVALUE, MEMBER, MERGE, MESSAGE_LENGTH, MESSAGE_OCTET_LENGTH, MESSAGE_TEXT, METHOD, MICROSECOND, MILLENNIUM, MIN, MINUTE, MINVALUE, MOD, MODIFIES, MODULE, MONTH, MORE, MULTISET, MUMPS, NAME, NAMES, NATIONAL, NATURAL, NCHAR, NCLOB, NESTING, NEW, NEXT, NO, NONE, NORMALIZE, NORMALIZED, NOT, NULL, NULLABLE, NULLIF, NULLS, NUMBER, NUMERIC, OBJECT, OCTETS, OCTET_LENGTH, OF, OFFSET, OLD, ON, ONLY, OPEN, OPTION, OPTIONS, OR, ORDER, ORDERING, ORDINALITY, OTHERS, OUT, OUTER, OUTPUT, OVER, OVERLAPS, OVERLAY, OVERRIDING, PAD, PARAMETER, PARAMETER_MODE, PARAMETER_NAME, PARAMETER_ORDINAL_POSITION, PARAMETER_SPECIFIC_CATALOG, PARAMETER_SPECIFIC_NAME, PARAMETER_SPECIFIC_SCHEMA, PARTIAL, PARTITION, PASCAL, PASSTHROUGH, PATH, PERCENTILE_CONT, PERCENTILE_DISC, PERCENT_RANK, PLACING, PLAN, PLI, POSITION, POWER, PRECEDING, PRECISION, PREPARE, PRESERVE, PRIMARY, PRIOR, PRIVILEGES, PROCEDURE, PUBLIC, QUARTER, RANGE, RANK, READ, READS, REAL, RECURSIVE, REF, REFERENCES, REFERENCING, REGR_AVGX, REGR_AVGY, REGR_COUNT, REGR_INTERCEPT, REGR_R2, REGR_SLOPE, REGR_SXX, REGR_SXY, REGR_SYY, RELATIVE, RELEASE, REPEATABLE, RESET, RESTART, RESTRICT, RESULT, RETURN, RETURNED_CARDINALITY, RETURNED_LENGTH, RETURNED_OCTET_LENGTH, RETURNED_SQLSTATE, RETURNS, REVOKE, RIGHT, ROLE, ROLLBACK, ROLLUP, ROUTINE, ROUTINE_CATALOG, ROUTINE_NAME, ROUTINE_SCHEMA, ROW, ROWS, ROW_COUNT, ROW_NUMBER, SAVEPOINT, SCALE, SCHEMA, SCHEMA_NAME, SCOPE, SCOPE_CATALOGS, SCOPE_NAME, SCOPE_SCHEMA, SCROLL, SEARCH, SECOND, SECTION, SECURITY, SELECT, SELF, SENSITIVE, SEQUENCE, SERIALIZABLE, SERVER, SERVER_NAME, SESSION, SESSION_USER, SET, SETS, SIMILAR, SIMPLE, SIZE, SMALLINT, SOME, SOURCE, SPACE, SPECIFIC, SPECIFICTYPE, SPECIFIC_NAME, SQL, SQLEXCEPTION, SQLSTATE, SQLWARNING, SQL_TSI_DAY, SQL_TSI_FRAC_SECOND, SQL_TSI_HOUR, SQL_TSI_MICROSECOND, SQL_TSI_MINUTE, SQL_TSI_MONTH, SQL_TSI_QUARTER, SQL_TSI_SECOND, SQL_TSI_WEEK, SQL_TSI_YEAR, SQRT, START, STATE, STATEMENT, STATIC, STDDEV_POP, STDDEV_SAMP, STREAM, STRUCTURE, STYLE, SUBCLASS_ORIGIN, SUBMULTISET, SUBSTITUTE, SUBSTRING, SUM, SYMMETRIC, SYSTEM, SYSTEM_USER, TABLE, TABLESAMPLE, TABLE_NAME, TEMPORARY, THEN, TIES, TIME, TIMESTAMP, TIMESTAMPADD, TIMESTAMPDIFF, TIMEZONE_HOUR, TIMEZONE_MINUTE, TINYINT, TO, TOP_LEVEL_COUNT, TRAILING, TRANSACTION, TRANSACTIONS_ACTIVE, TRANSACTIONS_COMMITTED, TRANSACTIONS_ROLLED_BACK, TRANSFORM, TRANSFORMS, TRANSLATE, TRANSLATION, TREAT, TRIGGER, TRIGGER_CATALOG, TRIGGER_NAME, TRIGGER_SCHEMA, TRIM, TRUE, TYPE, UESCAPE, UNBOUNDED, UNCOMMITTED, UNDER, UNION, UNIQUE, UNKNOWN, UNNAMED, UNNEST, UPDATE, UPPER, UPSERT, USAGE, USER, USER_DEFINED_TYPE_CATALOG, USER_DEFINED_TYPE_CODE, USER_DEFINED_TYPE_NAME, USER_DEFINED_TYPE_SCHEMA, USING, VALUE, VALUES, VARBINARY, VARCHAR, VARYING, VAR_POP, VAR_SAMP, VERSION, VIEW, WEEK, WHEN, WHENEVER, WHERE, WIDTH_BUCKET, WINDOW, WITH, WITHIN, WITHOUT, WORK, WRAPPER, WRITE, XML, YEAR, ZONE

{% endhighlight %}

{% top %}
