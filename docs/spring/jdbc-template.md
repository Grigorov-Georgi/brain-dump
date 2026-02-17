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

## JPA & JpaRepository

### The stack (bottom → top)

| Layer | Role |
|-------|------|
| Database | Persistence store |
| **JDBC** | Low-level SQL execution, connections |
| **JPA provider** (e.g. Hibernate) | ORM: object ↔ relational mapping |
| **Spring Data JPA** (JpaRepository) | Abstraction over JPA: repository interfaces |
| Your code | Repository interface extending `JpaRepository<Entity, Id>` |

JPA uses JDBC underneath; you normally don’t deal with SQL or connections.

### Your repository interface

```java
public interface OrderRepository extends JpaRepository<Order, UUID>
```

This is Spring Data JPA. No implementation needed — Spring generates it.

### What JpaRepository gives you (out of the box)

- `save()`, `findById()`, `findAll()`, `delete()`
- Paging and sorting
- Query derivation from method names

Example derived query (no JPQL):

```java
Optional<Order> findByCustomerId(UUID customerId);
```

Spring generates the SQL.

### Custom query with relationships

Prefer entity relationships and `JOIN FETCH` to avoid N+1. Don’t override built-in methods unless needed; add a new method instead.

```java
@Query("""
    SELECT o
    FROM Order o
    LEFT JOIN FETCH o.items
    WHERE o.id = :id
    """)
Optional<Order> findWithItemsById(UUID id);
```

### JdbcTemplate vs JPA: who does what

| | JdbcTemplate | JPA |
|--|-------------|-----|
| SQL | You write it | Generated |
| Mapping | You (RowMapper) | ORM |
| Joins / object graph | You build it | Relationships, dirty checking, caching |

### When to use which

| Prefer JPA when | Prefer JDBC / JdbcTemplate when |
|-----------------|----------------------------------|
| Standard CRUD, complex object graphs | Very complex SQL, heavy reporting |
| Domain-driven design, most business apps | Batch operations, performance-critical queries |
| | Stored procedures |

Using both in the same app is common (e.g. JPA for domain, JdbcTemplate for reports).

---

## Hibernate and the JPA stack

Hibernate is the implementation that does the work when you use JPA in most Spring apps.

### Where Hibernate sits

Typical Spring Data JPA stack:

**Your code** (JpaRepository, EntityManager) → **JPA** (spec/API) → **Hibernate** (implementation) → **JDBC** → **Database**

- **JPA** = standard (interfaces, annotations, rules). Not a concrete engine.
- **Hibernate** = engine that implements JPA: turns entity operations into SQL, runs it via JDBC, maps results back to objects.
- **JDBC** = low-level access to the database.

Mental model: JPA is the rules and the steering wheel; Hibernate is the engine; JDBC is the drive shaft to the database.

### What Hibernate does for you

**1) ORM: object ↔ table mapping**

You define entities with `@Entity`, `@Id`, `@OneToMany`, etc. Hibernate manages how they map to tables, joins, and foreign keys.

```java
@Entity
class Order {
    @Id UUID id;
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    List<OrderItem> items;
}
```

**2) SQL generation and execution**

When you call `orderRepository.save(order)`, Hibernate chooses INSERT vs UPDATE, generates SQL, binds parameters, and runs it through JDBC.

**3) Persistence context (first-level cache)**

Inside a transaction, Hibernate keeps a **persistence context** (Session). Same entity loaded twice in one transaction usually comes from memory, not the DB. Same row → same Java instance (identity guarantee).

```java
@Transactional
public void demo(UUID id) {
    Order a = em.find(Order.class, id);
    Order b = em.find(Order.class, id);
    // a == b is typically true (same managed instance)
}
```

**4) Dirty checking (automatic updates)**

You can change a managed entity and not call `save()`; Hibernate detects changes and issues UPDATE on flush/commit.

```java
@Transactional
public void changeStatus(UUID id) {
    Order o = orderRepository.findById(id).orElseThrow();
    o.setStatus("PAID"); // no save()
} // commit -> Hibernate flushes UPDATE
```

**5) Lazy loading and LazyInitializationException**

Associations marked `LAZY` are not loaded until accessed. If you access them **outside** a transaction/session, you get `LazyInitializationException`.

Fixes: load what you need with **JOIN FETCH**; or keep access inside `@Transactional`; or map to DTOs in the service layer.

**6) Flush behavior**

Hibernate does not always run SQL immediately. It queues changes and flushes typically before commit or before certain queries (for consistency). That allows batching of multiple changes.

**7) Other features**

Batching inserts/updates, second-level cache (optional), query cache (optional), optimistic locking (`@Version`), entity graphs, fetch plans.

### JPA vs Hibernate-specific features

If you stick to JPA APIs (EntityManager, JPQL, standard annotations), you can in theory switch to another JPA provider (e.g. EclipseLink). Hibernate also offers its own APIs and annotations (e.g. `Session`, custom types). Using those ties you to Hibernate.

---

## Transactions

### Declarative: @Transactional

Spring handles commit/rollback. No manual `begin`/`commit`/`rollback` in business code.

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;

    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);
    }
}
```

- Transaction starts when the method is entered.
- If the method completes normally → commit.
- If a **runtime exception** is thrown → rollback.

### Rollback on checked exceptions

By default Spring rolls back only on **unchecked** exceptions. To roll back on checked exceptions too:

```java
@Transactional(rollbackFor = Exception.class)
public void placeOrder(Order order) throws Exception {
    orderRepository.save(order);
    if (somethingWrong) {
        throw new Exception("Checked failure");
    }
}
```

### Programmatic rollback (no throw)

Force rollback without throwing:

```java
import org.springframework.transaction.interceptor.TransactionAspectSupport;

@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order);
    if (somethingWrong) {
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```

### Manual transaction management (rare)

Explicit `getTransaction` / `commit` / `rollback`:

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final PlatformTransactionManager txManager;

    public void manualTransaction() {
        TransactionStatus status = txManager.getTransaction(new DefaultTransactionDefinition());
        try {
            // business logic
            txManager.commit(status);
        } catch (Exception e) {
            txManager.rollback(status);
        }
    }
}
```

Rarely needed; prefer `@Transactional`.

### JdbcTemplate and transactions

Same mechanism: put `@Transactional` on the service method. Spring’s transaction management is technology-agnostic (works for JPA, JdbcTemplate, etc.).

```java
@Transactional
public void transferMoney() {
    jdbcTemplate.update("UPDATE account SET balance = balance - ? WHERE id = ?", amount, fromId);
    jdbcTemplate.update("UPDATE account SET balance = balance + ? WHERE id = ?", amount, toId);
}
```

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

**Q: What is JpaRepository and how does it relate to JDBC/JPA?**

A: JpaRepository is a Spring Data JPA interface. You extend it with your entity and ID type; Spring provides the implementation. It sits on top of a JPA provider (e.g. Hibernate), which uses JDBC. So: your interface → Spring Data JPA → JPA (Hibernate) → JDBC → database. You get CRUD, paging, and query derivation without writing SQL.

**Q: How does @Transactional work? When does Spring roll back?**

A: Spring starts a transaction when the method is entered and commits on normal return. On unchecked (runtime) exception it rolls back. Checked exceptions do not trigger rollback unless you set `rollbackFor` (e.g. `rollbackFor = Exception.class`). You can also call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` to force rollback without throwing.

**Q: Can you use JPA and JdbcTemplate in the same application?**

A: Yes. Same transaction management applies. Common pattern: JPA (JpaRepository) for domain CRUD and relationships, JdbcTemplate for reporting, batch, or very specific SQL.

**Q: What is the difference between JPA and Hibernate?**

A: JPA is the specification (API, annotations, behavior). Hibernate is an implementation of JPA. In most Spring apps, Hibernate is the engine: it generates SQL, uses JDBC, manages the persistence context, dirty checking, lazy loading. You code to JPA; Hibernate does the work.

**Q: What is the persistence context? What is LazyInitializationException?**

A: The persistence context (Hibernate Session) is a first-level cache per transaction. It keeps managed entities in memory and guarantees one Java instance per DB row within that transaction. LazyInitializationException happens when you access a lazy association (e.g. `order.getItems()`) after the session/transaction is closed. Fix by loading with JOIN FETCH, keeping access inside @Transactional, or mapping to DTOs.
