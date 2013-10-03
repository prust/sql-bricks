# SQL Bricks.js

SQL is a complicated, expressive DSL. SQL Bricks is not an abstraction layer and makes no attempt to hide SQL syntax. On the contrary, it is designed to be transparent, matching SQL so faithfully that developers with SQL experience will immediately know the API.

SQL Bricks provides easy parameter substitution, automatic quoting of columns that collide with SQL keywords ("order", "desc", etc), a nice chainable syntax, a few conveniences (support for user-supplied abbreviations and auto-generated join criteria) and, most importantly, **easy composition and re-use of SQL**.

### Composable

The primary goal of SQL Bricks is to enable the elimination of DRY in SQL-heavy applications by allowing easy composition and modification of SQL statements, like building blocks. To enable this, statements can be cloned and clauses can be added in any order (if a `WHERE` clause already exists, the new one will be `AND`ed to it):

```javascript
var active_users = select('*').from('user').where({'active': true});
// SELECT * FROM user WHERE active = true
var local_users = active_users.clone().where({'local': true});
// SELECT * FROM user WHERE active = true AND local = true
```

### Zero Configuration

SQL Bricks doesn't use or require a schema (though you can provide a set of table abbreviations for convenience, see below).

### Matches the SQL Language

The SQL Bricks API mirrors SQL as faithfully as possible. SQL keywords are chainable camelCase methods and non-keywords are passed in as strings. The result is that one can write and read long, complex SQL statements easily, without needing to reference the documentation:

```javascript
update('user').set('first_name', 'Fred').set('last_name', 'Flintstone');
// UPDATE user SET first_name = 'Fred', last_name = 'Flintstone'
insertInto('user', 'first_name', 'last_name').values('Fred', 'Flintstone');
// INSERT INTO user (first_name, last_name) VALUES ('Fred', 'Flintstone')
select('*').from('user').innerJoin('address').on('user.addr_id', 'address.id');
// SELECT * FROM user INNER JOIN address ON user.addr_id = address.id
select('*').from('user').where('first_name', 'Fred');
// SELECT * FROM user WHERE first_name = 'Fred'
```

The SQL Bricks API also allows object literals to be used wherever they make sense, to make the API more javascript-friendly:

```javascript
update('user').set({'first_name': 'Fred', 'last_name': 'Flintstone'});
// UPDATE user SET first_name = 'Fred', last_name = 'Flintstone'
insertInto('user').values({'first_name': 'Fred', 'last_name': 'Flintstone'});
// INSERT INTO user (first_name, last_name) VALUES ('Fred', 'Flintstone')
select('*').from('user').join('address').on({'user.addr_id': 'address.id'});
// SELECT * FROM user INNER JOIN address ON user.addr_id = address.id
```

For added convenience, `select()` defaults to `'*'`, shorter one-word method aliases are provided and in cases where a pair of keywords always go together (`upset().set()`, `insert().values()`, `.join().on()`), the second can be omitted, with the key/value pairs passed to the first method:

```javascript
update('user', {'first_name': 'Fred', 'last_name': 'Flintstone'});
// UPDATE user SET first_name = 'Fred', last_name = 'Flintstone'
insert('user', {'first_name': 'Fred', 'last_name': 'Flintstone'});
// INSERT INTO user (first_name, last_name) VALUES ('Fred', 'Flintstone')
select().from('user').join('address', {'user.addr_id': 'address.id'});
// SELECT * FROM user INNER JOIN address ON user.addr_id = address.id
```

While it is possible to chain `WHERE` criteria at the top-level via repeated calls to `.where()` and `.and()`, method chaining cannot express nested `AND`, `OR` and `NOT` groupings. To handle this, SQL Bricks provides a set of nestable functions for building `WHERE` criteria: `and()`, `or()`, `not()`, `like()`, `in()`, `isNull()`, `isNotNull()`, `eq()`, `lt()`, `lte()`, `gt()` and `gte()`. Object literals can also be used: `{name: 'Fred'}` renders as `name = 'Fred'` and multiple key/value pairs in an object literal are `AND`ed together:

```javascript
select('*').from('user').where(or(like('last_name', 'Flint%'), {'first_name': 'Fred'}));
// SELECT * FROM user WHERE last_name LIKE 'Flint%' OR first_name = 'Fred'
select('*').from('user').where('last_name', 'Flintstone').and('first_name', 'Fred');
// SELECT * FROM user WHERE last_name = 'Flintstone' AND first_name = 'Fred'
select('*').from('user').where({'last_name': 'Flintstone', 'first_name': 'Fred'});
// SELECT * FROM user WHERE last_name = 'Flintstone' AND first_name = 'Fred'
```

### Pseudo-Views

Another way that SQL Bricks allows re-use is through pseudo-views. This may not be as helpful for Postgres, where native views are fast, but it is helpful for MySQL and SQLite, where views can introduce performance problems (unless they can be flattened, see the "Subquery Flattening" section of [the SQLite Query Planner](http://www.sqlite.org/optoverview.html)).

The definition of a pseudo-view consists of a main table and, optionally, join tables and where criteria. Queries can then join to (and alias) this pseudo-view (the pseudo-view's join tables are prefixed with the view's alias):

```javascript
sql.defineView('localUser', 'user')
  .join('address').on({'user.addr_id': 'address.id'})
  .where({'address.local': true});

select('*').from('person')
  .join('localUser l_usr').on({'person.usr_id': 'l_usr.id'});
// SELECT * FROM person
// INNER JOIN user l_usr ON person.usr_id = l_usr.id
// INNER JOIN address l_usr_address ON l_usr.addr_id = l_usr_address.id
// WHERE l_usr_address.local = true
```

### Parameterized SQL

Calling `.toParams()` (as opposed to `.toString()`) will return an object with a `text` property that contains `$1, $2, etc` placeholders in the SQL and a corresponding `values` array. Anything on the right-hand side of a `WHERE` criteria is assumed to be a value, as well as anything values passed into an `insert()` or `update()` statement:

```javascript
update('user').set('first_name', 'Fred').where('last_name', 'Flintstone').toParams();
// {"text": "UPDATE user SET first_name = $1 WHERE last_name = $2", "values": ["Fred", "Flintstone"]}

update('user', {'first_name': 'Fred'}).where({'last_name': 'Flintstone'}).toParams();
// {"text": "UPDATE user SET first_name = $1 WHERE last_name = $2", "values": ["Fred", "Flintstone"]}
```

If you need to pass in SQL (such as a column name) somewhere that SQL Bricks expects a value (the right-hand side of `WHERE` criteria or something passed into `insert()` or `update()`), you can do this by wrapping the string in the `sql()` function:

```javascript
select('*').from('user').where({'billing_addr_id': sql('mailing_addr_id')})
// SELECT * FROM user WHERE billing_addr_id = mailing_addr_id
```

### Conveniences for a Higher Signal/Noise Ratio

#### Table Abbreviations

Frequently-used table abbreviations can be set via `setAbbrs()`:

```javascript
sql.setAbbrs({'usr': 'user', 'addr': 'address', 'zip': 'zipcode', 'psn': 'person'});
select().from('usr').join('addr', {'usr.addr_id': 'addr.id'});
// SELECT * FROM user usr INNER JOIN address addr ON usr.addr_id = addr.id
```

#### User-Supplied Join Criteria Function

The user can supply a function to automatically generate the `.on()` criteria for joins whenever it is not supplied explicitly, via a `joinCriteria()` function:

```javascript
sql.joinCriteria = function(left_tbl, left_alias, right_tbl, right_alias) {
  var criteria = {};
  criteria[left_alias + '.' + sql.getAbbr(right_tbl) + '_id'] = right_alias + '.id';
  return criteria;
};

select().from('user').join('address');
// SELECT * FROM user INNER JOIN address ON user.addr_id = address.id
```

The "left table" passed to the join criteria generator function will always be the most recently used table -- either the most recently join()ed table or, if there is none, the main table in the statement. If you want to perform a "chain" of joins, where each table joins from the previous one, you can call `.join()` multiple times, but if you want to join from one table directly to a number of related tables, you can call `.join()` once and pass the table names in as separate arguments:

```javascript
select().from('usr').join('addr').join('zip');
// SELECT * FROM user usr
// INNER JOIN address addr ON usr.addr_id = addr.id
// INNER JOIN zipcode zip ON addr.zip_id = zip.id

select().from('usr').join('addr', 'psn');
// SELECT * FROM user usr
// INNER JOIN address addr ON usr.addr_id = addr.id
// INNER JOIN person psn ON usr.psn_id = psn.id
```

If multiple tables are passed to `.join()`, the last one is the most recently used one and it will be used as the basis for the next `.join()`:

```javascript
select().from('usr').join('psn', 'addr').join('zip');
// SELECT * FROM user usr
// INNER JOIN person psn ON usr.psn_id = psn.id
// INNER JOIN address addr ON usr.addr_id = addr.id
// INNER JOIN zipcode zip ON addr.zip_id = zip.id
```

Note that this scheme doesn't support complex JOIN table layouts: if you do something like `.join('psn', 'addr').join('zip')` above, it is impossible to also join something to the `'psn'` table. This *could* be achieved by adding a way to explicitly specify the table you're joining from: `.join('psn', 'addr').join('zip').join('psn->employer')`, but this hasn't been implemented.

## To-Do

Add support for:

* .into()
* .using()
* .leftJoin / .rightJoin / .fullJoin / .crossJoin
* .union() / .intersect() / .except()
* .limit() / .offset()
* .fetch()
* .forUpdate() / .forShare()
* Allow more reuse by supporting .join()s for `UPDATE` and `DELETE` statements, implemented via `WHERE` criteria and placing the table name in the `FROM` and the `USING` clause, respectively.
* querying directly off of a pseudo-view: `select().from(viewName)`
* SQLite dialect (server-side and client-side examples)
* old browsers (via polyfills)

## Contributions

The tests can be run via `npm test` (provided `npm install` has been run to install the dependencies).

## Acknowledgements

Huge thanks to [Brian C](https://github.com/brianc) for his work on the [node-sql](https://github.com/brianc/node-sql) library, his patience with me as I hacked on it and his encouragement when I pitched the idea for an alternative approach to SQL generation.

## License

SQL Bricks is [MIT licensed](https://github.com/CSNW/sql-bricks/raw/master/LICENSE.md).
