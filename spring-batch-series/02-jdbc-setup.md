# Part 2 — Spring Batch JDBC Setup

*Spring Batch 6.x with JDBC — a small but rich tutorial*

**Previous:** [Part 1 — Foundations](./01-foundations.md)

---

## 5. `spring-boot-starter-batch-jdbc`: what it wires up automatically

Part 1 established that everything Spring Batch does depends on the `JobRepository` being backed by something durable. This section covers how that durability actually gets switched on in a Spring Boot application — and, just as importantly, what the framework leaves for you to decide.

In Spring Batch 6.x, the JDBC-backed `JobRepository` is enabled declaratively via `@EnableJdbcJobRepository`. In practice, though, you rarely write that annotation yourself for a default setup, because Spring Boot's starter does the wiring for you. Understanding *what* it wires up — and why each piece matters — is the goal of this section.

### Dependencies

Getting the starter and an embedded database onto the classpath is the trigger for everything that follows:

```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

```groovy
// Gradle
implementation 'org.springframework.boot:spring-boot-starter-batch-jdbc'
runtimeOnly 'com.h2database:h2'
```

### What you get for free

Once this starter and a `DataSource` bean (H2, in our case) are both on the classpath, Spring Boot does four things automatically, each of which corresponds to a concept introduced in Part 1:

1. **An auto-configured `JobRepository`**, backed by JDBC, wired to your `DataSource` and a `PlatformTransactionManager`. This is the durable store Part 1 described — Boot builds it for you rather than leaving you to assemble it by hand.
2. **Automatic schema initialization.** Spring Boot detects the batch starter and, by default, runs the appropriate `schema-*.sql` script for your database platform (H2, PostgreSQL, MySQL, Oracle, etc.) against your `DataSource` at startup, creating the `BATCH_*` metadata tables the `JobRepository` needs to write to. Section 6 opens these tables up directly.
3. **An auto-configured `JobOperator`** — the interface you'll actually use to start, stop, and restart jobs (covered in Part 4).
4. **Job auto-run behavior.** By default, Spring Boot attempts to run any `Job` beans found in the context automatically on startup, using `JobParameters` derived from the command line. This is convenient for a five-minute demo, but it's worth turning off almost immediately in any real project: production jobs are triggered explicitly — by a scheduler, a message, an API call — not "whenever the app happens to start."

**A note on `@EnableJdbcJobRepository` specifically:** because the starter auto-configures this for you, you don't need to add it yourself for a default setup. The reason you'd add it explicitly is to *customize* something — a table prefix, an isolation level, a dedicated `DataSource` — which is exactly the scenario Section 7 walks through. It's typically paired with `@EnableBatchProcessing`: the latter enables the common batch infrastructure that's shared across all storage backends, while `@EnableJdbcJobRepository` specifically selects and configures the JDBC-backed repository implementation (as opposed to, say, `@EnableMongoJobRepository`).

> **Coming from Spring Batch 5.x?** Before 6.0, `@EnableBatchProcessing` was tied directly to JDBC infrastructure — configuring things like `dataSourceRef` right on that one annotation. In 6.0, store-specific configuration was split out into dedicated annotations (`@EnableJdbcJobRepository`, `@EnableMongoJobRepository`), while `@EnableBatchProcessing` now only configures store-agnostic infrastructure. If you're porting a 5.x config, that split is the main shape of the change to account for.

### Minimal `application.yml`

With the dependencies in place, two properties are enough to get a working, sensibly-configured setup for this tutorial:

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:batchdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
  batch:
    jdbc:
      initialize-schema: always   # embedded DB: safe to always (re)create on startup
    job:
      enabled: false              # don't auto-run jobs on context startup
```

Each of these two `batch` properties is worth understanding rather than copying blindly:

- `spring.batch.jdbc.initialize-schema` accepts `always`, `never`, or `embedded` (the default). The default only auto-initializes for embedded databases like H2 — not for a real Postgres or MySQL instance, where you're expected to manage migrations yourself, typically via Flyway or Liquibase. We set it explicitly to `always` here because H2 is in-memory and gets wiped on every restart anyway, so there's no migration history to protect.
- `spring.batch.job.enabled: false` is worth calling out on its own, because its default value is easy to overlook. Left at its default (`true`), Boot runs every `Job` bean at startup — harmless for a five-minute demo, but a genuine footgun in a real application, where a `JobOperator` or scheduler, not the application's own startup sequence, should be in control of *when* jobs run.

---

## 6. Batch metadata schema — the real tables

Section 5 described the `JobRepository` as "auto-configured," which can make it feel like a black box. It isn't one — everything it persists lives in a small, fixed set of tables, all prefixed `BATCH_` by default. Looking at these tables directly demystifies a lot of "how does restart even work?" confusion, because the answer turns out to be refreshingly unmagical: it's just rows in a database.

### The core tables

The six tables below map directly onto the runtime concepts from Part 1 — each `JobInstance`, `JobExecution`, and `StepExecution` has a row (or several) somewhere in this list:

| Table | Purpose |
|---|---|
| `BATCH_JOB_INSTANCE` | One row per `JobInstance` — job name + a hash of its identifying parameters |
| `BATCH_JOB_EXECUTION` | One row per `JobExecution` — status, start/end time, exit code, linked to a `JOB_INSTANCE_ID` |
| `BATCH_JOB_EXECUTION_PARAMS` | The actual `JobParameters` key/value pairs for a given execution (typed: string, date, long, double) |
| `BATCH_JOB_EXECUTION_CONTEXT` | Serialized `ExecutionContext` data at the job level |
| `BATCH_STEP_EXECUTION` | One row per `StepExecution` — read/write/commit/rollback/skip counts, status, linked to a `JOB_EXECUTION_ID` |
| `BATCH_STEP_EXECUTION_CONTEXT` | Serialized `ExecutionContext` data at the step level (this is what makes a chunk-based step resumable mid-stream) |

**Why there are two separate context tables, not one:** the split between job-level and step-level context follows directly from the split between `JobExecution` and `StepExecution` covered in Part 1. Job-level context (`BATCH_JOB_EXECUTION_CONTEXT`) persists data that's relevant across the whole job — aggregate counters, summary information gathered by one step that a later step needs. Step-level context (`BATCH_STEP_EXECUTION_CONTEXT`) is scoped to a single step, and is specifically what that step's reader uses to remember its position — the last row ID read, the current file offset — so a restart can resume exactly where *that step* left off, without needing to know or care about any other step in the job.

Concretely, a step-level context looks something like this once serialized:

```json
{
  "lastProcessedId": 45239,
  "currentFileLine": 1024,
  "processedCount": 1024
}
```

Once you see that JSON, "restart" stops being an abstract capability and becomes a concrete mechanism: on restart, the framework reads this data back out of `BATCH_STEP_EXECUTION_CONTEXT`, hands it to your reader via `ExecutionContext.get(...)`, and your reader uses it to skip ahead to `currentFileLine: 1024` instead of starting from line 0. That's genuinely the entire mechanism — nothing more exotic is happening underneath it.

Two other tables exist alongside the six above — `BATCH_JOB_SEQ` and `BATCH_STEP_EXECUTION_SEQ` (or platform-specific sequence/identity mechanisms) — but they're pure plumbing, used only to generate primary keys, and not conceptually interesting on their own.

**A preview worth flagging here, since it will matter directly in Part 3:** not every parameter stored in `BATCH_JOB_EXECUTION_PARAMS` necessarily *identifies* the `JobInstance`. Spring Batch distinguishes **identifying** parameters — the ones that, combined with the job name, define which `JobInstance` you're running (a report date, for example) — from **non-identifying** ones, which are extra data you want carried along, like a correlation or tracing ID, that shouldn't cause two otherwise-identical runs to be treated as different instances. Part 3 uses this distinction directly when designing our job's parameters.

### How they relate

Seeing the six tables laid out as a hierarchy makes the earlier prose description concrete:

```
BATCH_JOB_INSTANCE (1) ──────< (0..*) BATCH_JOB_EXECUTION
        │                              │
        │ JOB_INSTANCE_ID              │ JOB_EXECUTION_ID
        │                              ├──< BATCH_JOB_EXECUTION_PARAMS
        │                              ├──< BATCH_JOB_EXECUTION_CONTEXT
        │                              │
        │                              └──< (0..*) BATCH_STEP_EXECUTION
        │                                            │
        │                                            └──< BATCH_STEP_EXECUTION_CONTEXT
```

Reading this top to bottom, in the same order the data would actually be written during a run: a `JobInstance` row fans out to one or more `JobExecution` rows — more than one only if there were failures or restarts, exactly as Part 1 described. Each `JobExecution` row fans out to its parameters, its job-level context, and one `StepExecution` row per step that ran during that attempt. Each `StepExecution`, in turn, has its own context row underneath it.

### Seeing it for yourself

Reading a schema diagram only goes so far — the fastest way to build real intuition is to query these tables directly, once you have a job running against H2 (Part 3 gets you there):

```sql
-- What job instances exist, and for what parameters?
SELECT ji.JOB_INSTANCE_ID, ji.JOB_NAME, jp.PARAMETER_NAME, jp.PARAMETER_VALUE
FROM BATCH_JOB_INSTANCE ji
JOIN BATCH_JOB_EXECUTION je ON je.JOB_INSTANCE_ID = ji.JOB_INSTANCE_ID
JOIN BATCH_JOB_EXECUTION_PARAMS jp ON jp.JOB_EXECUTION_ID = je.JOB_EXECUTION_ID;

-- Did last night's run succeed?
SELECT JOB_EXECUTION_ID, STATUS, START_TIME, END_TIME, EXIT_CODE
FROM BATCH_JOB_EXECUTION
ORDER BY START_TIME DESC;

-- Per-step counts for a given execution
SELECT STEP_NAME, STATUS, READ_COUNT, WRITE_COUNT, SKIP_COUNT, COMMIT_COUNT, ROLLBACK_COUNT
FROM BATCH_STEP_EXECUTION
WHERE JOB_EXECUTION_ID = 1;
```

If you enable H2's web console during development, you can go a step further and browse these tables interactively *while a job runs* — watching rows appear in `BATCH_STEP_EXECUTION_CONTEXT` as chunks commit is genuinely one of the best ways to build a mental model of the framework, and worth doing at least once even if you never touch H2 in production:

```yaml
spring:
  h2:
    console:
      enabled: true
      path: /h2-console
  datasource:
    url: jdbc:h2:mem:batchdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
```

---

## 7. Configuring a JDBC-backed `JobRepository` with H2

### What's automatic vs. what you control

Sections 5 and 6 covered what the `JobRepository` does and what it persists. This section turns to a more practical question: given `spring-boot-starter-batch-jdbc` and an H2 `DataSource` on the classpath, what actually requires configuration from you, and what doesn't?

Without any explicit `@Bean` declarations, the following already happens:

- The `BATCH_*` schema is created, because `initialize-schema` defaults to `embedded`, and H2 counts as embedded.
- A `JobRepository` bean is configured, using your primary `DataSource` and `PlatformTransactionManager`.
- A `JobOperator` bean is configured on top of that repository.

Explicit configuration only becomes necessary when you want to **override** one of these defaults: a non-default table prefix, a specific transaction isolation level, a secondary `DataSource` dedicated to batch metadata (common in larger systems, so batch metadata writes don't compete with business-data transactions), or manual schema management against a real database. Each of these is covered below.

### What's *not* auto-configured

It's equally important to be clear about the other side of the boundary: the starter gives you infrastructure, not business logic. Three things remain entirely yours to define, and no amount of auto-configuration will supply them:

- The `Job` bean(s) — the actual pipeline definition.
- The `Step` bean(s) that make up each job.
- Your `ItemReader`, `ItemProcessor`, and `ItemWriter` implementations — the actual read/transform/write logic.

None of that is magic, and none of it is covered by this section — it's exactly what Part 3 builds, from scratch.

### The `PlatformTransactionManager`

One auto-configured piece deserves a moment of attention on its own, because it's easy to use without ever noticing it: alongside the `JobRepository`, Spring Boot also auto-configures a `PlatformTransactionManager` (a `JdbcTransactionManager`/`DataSourceTransactionManager` for our JDBC setup), bound to the same `DataSource`. This is the object that actually coordinates transaction boundaries — both the chunk-commit transactions inside your steps, and the `JobRepository`'s own updates to the `BATCH_*` tables, happen through it. You won't usually interact with it directly, but it's worth knowing it's there, because it's the reason a chunk commit and its corresponding `BATCH_STEP_EXECUTION` update stay consistent with each other — if one rolls back, so does the other.

### Baseline configuration class

Putting the last three sections together, a from-scratch project needs remarkably little explicit configuration:

```java
@Configuration
@EnableBatchProcessing
@EnableJdbcJobRepository(
        // all optional — shown for awareness, these ARE the defaults
        tablePrefix = "BATCH_",
        isolationLevelForCreate = "SERIALIZABLE",
        maxVarCharLength = 2500
)
public class BatchConfig {
    // Job and Step beans go here, or in their own @Configuration classes
}
```

`@EnableBatchProcessing` bootstraps the core batch infrastructure beans; `@EnableJdbcJobRepository` specifically selects and configures the JDBC-backed `JobRepository` implementation (as opposed to, say, the MongoDB-backed one, or the in-memory `ResourcelessJobRepository` used for repository-free testing). The `isolationLevelForCreate` attribute is worth pausing on, since it connects directly back to Part 1: `SERIALIZABLE` is the default specifically to guarantee that concurrent launch attempts for the same job can't both succeed — the double-fire guard depends on it.

If you *do* need a dedicated `DataSource` for batch metadata, the same annotation is where you point it there, by bean name:

```java
@EnableBatchProcessing
@EnableJdbcJobRepository(
        dataSourceRef = "batchDataSource",
        transactionManagerRef = "batchTransactionManager"
)
public class BatchConfig {
    // ...
}
```

### `DataSource` and transaction manager

For our H2 setup, Spring Boot auto-configures both of these from `application.yml`, so you don't need to declare them explicitly — unless you're pointing batch metadata at a *different* database than your business data, which is the one case worth showing explicitly:

```java
// Only needed if you want batch metadata in a SEPARATE database
// from your business/domain data — a common pattern at scale.
@Bean
@Qualifier("batchDataSource")
public DataSource batchDataSource() {
    return DataSourceBuilder.create()
            .url("jdbc:h2:mem:batchmeta;DB_CLOSE_DELAY=-1")
            .driverClassName("org.h2.Driver")
            .username("sa")
            .build();
}
```

For this tutorial, we're keeping it simple and using one `DataSource`, shared by business data and batch metadata alike — the separation above is worth knowing about, but not something we need here.

### Manual schema initialization (when you'd need it)

Everything so far has relied on H2's convenient auto-initialization, but that convenience doesn't travel well to a real deployment. Against a real database — Postgres, MySQL, Oracle — you would **not** want `initialize-schema: always` running on every deploy, since that risks re-running schema creation against a database that already has data and a migration history to protect. The safer path looks like this instead:

1. Set `spring.batch.jdbc.initialize-schema: never`.
2. Pull the platform-specific schema script — Spring Batch ships one per database, e.g. `org/springframework/batch/core/schema-postgresql.sql` inside `spring-batch-core.jar` — and run it once, manually or as a Flyway/Liquibase migration, as part of your normal schema-management process.
3. From then on, schema changes to `BATCH_*` tables (rare — they change only across major Spring Batch versions) get versioned alongside your own migrations, rather than silently re-applied by the framework at every startup.

The table below is a reference for step 2 — which script to pull, depending on target database:

| Database | Schema script path (inside `spring-batch-core.jar`) | Notes |
|---|---|---|
| PostgreSQL | `org/springframework/batch/core/schema-postgresql.sql` | Straightforward default |
| MySQL | `org/springframework/batch/core/schema-mysql.sql` | Watch collation settings on the metadata tables |
| Oracle | `org/springframework/batch/core/schema-oracle.sql` | Uses Oracle-specific sequence syntax |
| H2 | `org/springframework/batch/core/schema-h2.sql` | What Boot uses automatically for our tutorial setup |

**A real pitfall worth knowing about, precisely because it's easy to trip over:** if you manage schema via Flyway/Liquibase *and* leave `spring.batch.jdbc.initialize-schema` at its default, you can end up with both mechanisms trying to create the same tables — or with Boot's batch auto-init running before your migration tool has had a chance to run, depending on startup ordering. The safe pattern is to make the boundary explicit: `initialize-schema: never`, and let Flyway/Liquibase own the `BATCH_*` schema as just another migration, exactly like any of your own tables.

This distinction is worth internalizing now, even though we're relying on H2's auto-init for the rest of this tutorial, because it won't hold once a real database enters the picture: **auto-schema-init is a development convenience, not a production pattern.**

---

### Checkpoint

Before moving to Part 3, you should be able to:

- Explain what `spring-boot-starter-batch-jdbc` configures for you automatically, and why `spring.batch.job.enabled: false` matters.
- Name the six core `BATCH_*` tables and what each one is responsible for.
- Explain why `BATCH_STEP_EXECUTION_CONTEXT` is the table that makes mid-step restart possible.
- Describe the production-appropriate way to initialize the batch schema against a real database (vs. the H2 shortcut we're using here).

**Next:** Part 3 — Mini-Project #1: Your First Job. With the infrastructure now configured, we'll build our first real job — reading orders from a CSV, calculating tax, and writing the results to H2 (chunk-oriented processing, a runnable end-to-end example, and typed `JobParameters`).
