# JdbcTemplate

## What it is

**JdbcTemplate** is Spring’s abstraction over JDBC. It handles connection acquisition/release, exception translation, and leaves you to write SQL and map results. You inject a `DataSource` (or use Boot’s auto-configured one); Spring provides a `JdbcTemplate` bean when `spring-jdbc` is on the classpath.

---

## Why use it (vs plain JDBC)

| Plain JDBC | JdbcTemplate |
|------------|--------------|
| Manual `Connection`, `PreparedStatement`, `ResultSet`, try/finally/close | Connections and statements managed for you |
| Checked `SQLException` everywhere | Spring translates to unchecked `DataAccessException` hierarchy |
| Boilerplate for every query | One-liners for common patterns (query, update, batch) |
| Easy to leak connections | Connection always returned to the pool |

---

## Core usage

**Inject it:** constructor-inject `JdbcTemplate` (or use `JdbcTemplate` + `DataSource` in a `@Configuration` `@Bean`).

**Query returning a list:** `jdbcTemplate.query(sql, rowMapper, args...)`  
**Query returning a single object:** `jdbcTemplate.queryForObject(sql, rowMapper, args...)`  
**Update/insert/delete:** `jdbcTemplate.update(sql, args...)` — returns number of affected rows.  
**Arbitrary execution:** `jdbcTemplate.execute(sql)` for DDL or when you don’t need a return.

Use **positional parameters** `?` in SQL; pass arguments in the same order as the `?` placeholders.

---

## RowMapper

Maps one `ResultSet` row to an object. Used for `query()` and `queryForObject()`.

```java
RowMapper<User> mapper = (rs, rowNum) -> new User(
    rs.getLong("id"),
    rs.getString("name")
);
List<User> users = jdbcTemplate.query("SELECT id, name FROM user", mapper);
```

Reusable; prefer a static/lambda `RowMapper` per entity. For single column, you can use `queryForObject(sql, Long.class, args)` etc.

---

## NamedParameterJdbcTemplate

Uses named parameters (`:name`) instead of `?`, so you pass a `Map<String, ?>` or `SqlParameterSource`. Reduces mistakes when SQL has many parameters.

```java
Map<String, Object> params = Map.of("id", 1L, "status", "ACTIVE");
jdbcTemplate.query("SELECT * FROM user WHERE id = :id AND status = :status", params, rowMapper);
```

Wraps a `JdbcTemplate` (or `DataSource`). In Boot you can inject `NamedParameterJdbcTemplate` directly.

---

## Batch operations

**batchUpdate(String sql, BatchPreparedStatementSetter setter)** — runs the same SQL with different parameters in a batch (fewer round-trips).  
**batchUpdate(String sql, List<Object[]> batchArgs)** — convenience when each execution has an array of args.

Use for bulk inserts/updates; much faster than many single `update()` calls.

---

## Exception handling

Spring wraps `SQLException` in **DataAccessException** subclasses (e.g. `DuplicateKeyException`, `EmptyResultDataAccessException`). They are unchecked. Use for flow control (e.g. catch `EmptyResultDataAccessException` when `queryForObject` may return no row) or let them propagate.

---

## Quick comparison with JPA

| | JdbcTemplate | JPA / Hibernate |
|--|--------------|-----------------|
| Control | Full SQL, you map rows | ORM maps entities, generates SQL |
| Overhead | Minimal | Session, cache, lazy loading |
| Best for | Simple CRUD, reports, bulk, tuned SQL | Rich domain, relationships, less SQL |

---

## Interview Q&A

**Q: What is JdbcTemplate and why use it instead of plain JDBC?**

A: It’s Spring’s helper for JDBC. It manages connections and statements, translates SQLExceptions into DataAccessException, and reduces boilerplate (query/update/batch). You still write SQL and map results yourself.

**Q: How do you run a SELECT and map results to objects?**

A: Use `query(sql, RowMapper, args...)` for a list or `queryForObject(sql, RowMapper, args...)` for one row. The RowMapper maps each ResultSet row to your object. For a single column you can use `queryForObject(sql, Long.class, args)`.

**Q: What is RowMapper?**

A: A callback that maps one row of a ResultSet to an object. Spring calls it for each row. You implement it (e.g. as a lambda) and pass it to query/queryForObject.

**Q: How do you do parameterized queries with JdbcTemplate?**

A: Put `?` in the SQL and pass the arguments in order to the method: `query(sql, rowMapper, arg1, arg2)`. For named parameters use NamedParameterJdbcTemplate with `:paramName` and a Map or SqlParameterSource.

**Q: What is NamedParameterJdbcTemplate?**

A: An extension that supports named placeholders (`:name`) instead of `?`. You pass a Map or SqlParameterSource. Easier to maintain when there are many parameters.

**Q: How do you do batch inserts/updates?**

A: Use `batchUpdate(sql, BatchPreparedStatementSetter)` or `batchUpdate(sql, List<Object[]>)` to run the same statement with many parameter sets in one batch, reducing round-trips.

**Q: What happens when a JDBC exception is thrown?**

A: Spring’s JdbcTemplate catches SQLException and translates it into a DataAccessException subclass (e.g. DuplicateKeyException, EmptyResultDataAccessException). These are unchecked.

**Q: JdbcTemplate vs JPA — when would you choose which?**

A: JdbcTemplate for simple CRUD, reporting, bulk operations, or when you want full control over SQL and minimal overhead. JPA when you have a rich domain model, relationships, and want ORM to handle mapping and SQL generation.
