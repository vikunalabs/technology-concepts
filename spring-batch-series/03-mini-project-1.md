# Part 3 — Mini-Project #1: Your First Job

*Spring Batch 6.x with JDBC — a small but rich tutorial*

**Previous:** [Part 2 — Spring Batch JDBC Setup](./02-jdbc-setup.md)

---

## 8. Chunk-oriented processing explained: Reader → Processor → Writer

Most Spring Batch steps you'll write are **chunk-oriented**. The idea is simple, but it's worth being precise about it, because the transaction boundary it creates is the single most important thing to understand before writing any code.

### The loop

A chunk-oriented step repeats this cycle until the input is exhausted:

```
┌─────────────────────────────────────────────────────────┐
│  repeat until chunk is full (or input is exhausted):     │
│      item = reader.read()                                │
│      transformedItem = processor.process(item)           │
│      add transformedItem to the chunk buffer              │
│                                                            │
│  writer.write(chunk buffer)     ◄── happens ONCE per chunk│
│  COMMIT                          ◄── transaction boundary │
└─────────────────────────────────────────────────────────┘
```

Three collaborators, three distinct jobs:

- **`ItemReader<I>`** — produces one item at a time. Returns `null` to signal "no more input." Reads a row from a file, a database cursor, a queue — anything that can be modeled as a sequence of items.
- **`ItemProcessor<I, O>`** — transforms, validates, or enriches a single item. Optional — plenty of steps skip this and write what they read. Can also *filter*: returning `null` from `process()` drops that item from the chunk entirely (it won't be written, and it's counted separately as a filtered item, not a written one):

  ```java
  @Override
  public OrderWithTax process(Order order) {
      if (order.getAmount().compareTo(BigDecimal.ZERO) == 0) {
          return null;   // filtered — not written, not an error, just skipped
      }
      // ... normal transformation
  }
  ```
- **`ItemWriter<O>`** — receives a **whole chunk at once** (as a list), not one item at a time. This matters: writers are batch-oriented by design, which is exactly what lets a JDBC writer issue one efficient batch `INSERT` instead of N individual round-trips.

### Why the commit happens per-chunk, not per-item

The commit interval — the "chunk size" — is a deliberate trade-off, and it's the answer to "how much work do I redo if something fails partway through?"

- **Chunk size 1** behaves like committing after every single item — safest in terms of blast radius, but the worst possible throughput (a transaction commit per row).
- **A large chunk size** (say, 1,000) means far fewer commits and much better throughput, but if item 999 in that chunk throws an exception, the whole chunk rolls back — items 1 through 998 in that chunk are *not* written, even though they were individually fine.

This is why chunk size is a genuine tuning decision, not just a config value to leave at some default: it's a knob balancing throughput against how much reprocessing a mid-chunk failure costs you. We'll pick a small chunk size in this mini-project purely so it's easy to observe multiple commits happening.

| Chunk size | Throughput | Failure blast radius | Reasonable for |
|---|---|---|---|
| Small (1–10) | Lower | Minimal — little to reprocess | Critical or expensive-to-reprocess records |
| Medium (50–200) | Good | Moderate | General-purpose default for most jobs |
| Large (1,000+) | High | Large — a whole chunk re-does on failure | High-volume bulk loads, especially with idempotent writers |

This isn't a precise formula — the right value depends on item size, writer cost, and how expensive re-processing actually is for your data — but it's a reasonable starting mental model.

### What "restart" actually means at this level

Recall from Part 2: after a successful chunk commit, the step's position gets checkpointed into `BATCH_STEP_EXECUTION_CONTEXT`. So if the JVM dies after chunk 3 commits but before chunk 4 finishes, a restart resumes reading from wherever chunk 4 would have started — chunks 1 through 3 are not re-processed. This is *why* chunk-oriented processing and restartability are so tightly linked: the chunk boundary **is** the checkpoint.

---

## 9. Building it: CSV of orders → calculate tax → H2

We're building a complete, runnable job: read a CSV of customer orders, calculate an 8% tax on each, and write the enriched result into an H2 table. Every piece of code below is meant to be copy-pasteable into a real project.

### Project layout

```
spring-batch-tutorial/
├── pom.xml
├── src/main/resources/
│   ├── application.yml
│   ├── schema.sql          ← our OWN business table (not the BATCH_* ones)
│   └── data/orders.csv
└── src/main/java/com/example/batchtutorial/
    ├── BatchTutorialApplication.java
    ├── config/BatchConfig.java
    ├── domain/Order.java
    ├── domain/OrderWithTax.java
    └── batch/OrderTaxProcessor.java
```

### `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.1.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>batch-tutorial</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-batch-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

This targets Spring Boot 4.1.0, which is built on Spring Framework 7 and is the generation Spring Batch 6.x is designed for. Spring Boot 4.x keeps a Java 17 minimum baseline; we're using Java 21 (an LTS release) purely as a comfortable, current choice — 17 would work identically for everything in this tutorial.

### `application.yml`

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:batchdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
  sql:
    init:
      mode: always          # runs OUR schema.sql (business tables)
  batch:
    jdbc:
      initialize-schema: always   # runs Spring Batch's own BATCH_* schema
    job:
      enabled: false         # we launch explicitly, not on context startup
  h2:
    console:
      enabled: true
      path: /h2-console
```

### `schema.sql` — our business table

This is a plain Spring Boot `schema.sql`, unrelated to the `BATCH_*` metadata tables — it defines *our* output table.

```sql
CREATE TABLE IF NOT EXISTS PROCESSED_ORDERS (
    order_id       VARCHAR(20)    NOT NULL,
    customer_name  VARCHAR(100)   NOT NULL,
    amount         DECIMAL(10,2)  NOT NULL,
    tax            DECIMAL(10,2)  NOT NULL,
    total          DECIMAL(10,2)  NOT NULL
);
```

Notice we now have **two separate schema mechanisms** running at startup, triggered by two separate properties: `spring.sql.init.mode=always` runs this `schema.sql` for *our* business table, while `spring.batch.jdbc.initialize-schema=always` (Part 2) independently creates the `BATCH_*` metadata tables. They don't know about each other and don't need to — keep that separation in mind if you ever wonder why a schema change to one doesn't show up in the other.

### `data/orders.csv`

Create this file at `src/main/resources/data/orders.csv` — that's where the `ClassPathResource("data/orders.csv")` in the reader bean (below) will look for it.

```csv
orderId,customerName,amount
ORD-001,Alice Johnson,150.00
ORD-002,Bob Martinez,89.99
ORD-003,Priya Natarajan,432.10
ORD-004,Wei Chen,25.50
ORD-005,Fatima Al-Sayed,999.00
ORD-006,Diego Fernandez,12.75
ORD-007,Grace Kim,310.20
ORD-008,Liam O'Brien,64.00
```

### Domain classes

```java
// domain/Order.java — the raw CSV row
package com.example.batchtutorial.domain;

import java.math.BigDecimal;

public class Order {
    private String orderId;
    private String customerName;
    private BigDecimal amount;

    // no-args constructor + getters/setters required —
    // FlatFileItemReader's BeanWrapperFieldSetMapper populates via setters
    public Order() {}

    public String getOrderId() { return orderId; }
    public void setOrderId(String orderId) { this.orderId = orderId; }
    public String getCustomerName() { return customerName; }
    public void setCustomerName(String customerName) { this.customerName = customerName; }
    public BigDecimal getAmount() { return amount; }
    public void setAmount(BigDecimal amount) { this.amount = amount; }
}
```

`Order` needs a no-args constructor and setters because `BeanWrapperFieldSetMapper` populates it via reflection — that's the tradeoff for the convenience of not hand-writing a mapper. If you'd rather keep your domain objects immutable, a custom `FieldSetMapper<Order>` (or a mapper built around a Java `record`) works just as well and avoids mutable setters entirely; we're using `BeanWrapperFieldSetMapper` here because it's the simplest starting point and the most common pattern you'll see in Spring Batch examples.

```java
// domain/OrderWithTax.java — the enriched output item
package com.example.batchtutorial.domain;

import java.math.BigDecimal;

public class OrderWithTax {
    private final String orderId;
    private final String customerName;
    private final BigDecimal amount;
    private final BigDecimal tax;
    private final BigDecimal total;

    public OrderWithTax(String orderId, String customerName, BigDecimal amount,
                         BigDecimal tax, BigDecimal total) {
        this.orderId = orderId;
        this.customerName = customerName;
        this.amount = amount;
        this.tax = tax;
        this.total = total;
    }

    public String getOrderId() { return orderId; }
    public String getCustomerName() { return customerName; }
    public BigDecimal getAmount() { return amount; }
    public BigDecimal getTax() { return tax; }
    public BigDecimal getTotal() { return total; }
}
```

### The processor

```java
// batch/OrderTaxProcessor.java
package com.example.batchtutorial.batch;

import com.example.batchtutorial.domain.Order;
import com.example.batchtutorial.domain.OrderWithTax;
import org.springframework.batch.item.ItemProcessor;

import java.math.BigDecimal;
import java.math.RoundingMode;

public class OrderTaxProcessor implements ItemProcessor<Order, OrderWithTax> {

    private static final BigDecimal TAX_RATE = new BigDecimal("0.08");

    @Override
    public OrderWithTax process(Order order) {
        BigDecimal tax = order.getAmount()
                .multiply(TAX_RATE)
                .setScale(2, RoundingMode.HALF_UP);
        BigDecimal total = order.getAmount().add(tax);

        return new OrderWithTax(
                order.getOrderId(),
                order.getCustomerName(),
                order.getAmount(),
                tax,
                total
        );
    }
}
```

Nothing filters here (every item returns a result), but note the shape: if we wanted to reject, say, zero-dollar orders, we'd simply `return null;` for those — Spring Batch drops them from the chunk automatically and counts them as "filtered," distinct from both "written" and "skipped."

### The reader, writer, step, and job — `BatchConfig.java`

```java
package com.example.batchtutorial.config;

import com.example.batchtutorial.batch.OrderTaxProcessor;
import com.example.batchtutorial.domain.Order;
import com.example.batchtutorial.domain.OrderWithTax;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.database.builder.JdbcBatchItemWriterBuilder;
import org.springframework.batch.item.database.JdbcBatchItemWriter;
import org.springframework.batch.item.file.FlatFileItemReader;
import org.springframework.batch.item.file.builder.FlatFileItemReaderBuilder;
import org.springframework.batch.item.file.mapping.BeanWrapperFieldSetMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ClassPathResource;
import org.springframework.transaction.PlatformTransactionManager;

import javax.sql.DataSource;

@Configuration
public class BatchConfig {

    @Bean
    public FlatFileItemReader<Order> orderItemReader() {
        BeanWrapperFieldSetMapper<Order> mapper = new BeanWrapperFieldSetMapper<>();
        mapper.setTargetType(Order.class);

        return new FlatFileItemReaderBuilder<Order>()
                .name("orderItemReader")
                .resource(new ClassPathResource("data/orders.csv"))
                .delimited()
                .names("orderId", "customerName", "amount")
                .linesToSkip(1)                 // skip the CSV header row
                .fieldSetMapper(mapper)
                .build();
    }

    @Bean
    public OrderTaxProcessor orderTaxProcessor() {
        return new OrderTaxProcessor();
    }

    @Bean
    public JdbcBatchItemWriter<OrderWithTax> orderItemWriter(DataSource dataSource) {
        return new JdbcBatchItemWriterBuilder<OrderWithTax>()
                .dataSource(dataSource)
                .sql("""
                     INSERT INTO PROCESSED_ORDERS (order_id, customer_name, amount, tax, total)
                     VALUES (:orderId, :customerName, :amount, :tax, :total)
                     """)
                .beanMapped()
                .build();
    }

    @Bean
    public Step processOrdersStep(JobRepository jobRepository,
                                   PlatformTransactionManager transactionManager,
                                   FlatFileItemReader<Order> orderItemReader,
                                   OrderTaxProcessor orderTaxProcessor,
                                   JdbcBatchItemWriter<OrderWithTax> orderItemWriter) {
        return new StepBuilder("processOrdersStep", jobRepository)
                .<Order, OrderWithTax>chunk(3, transactionManager)   // small on purpose — see below
                .reader(orderItemReader)
                .processor(orderTaxProcessor)
                .writer(orderItemWriter)
                .build();
    }

    @Bean
    public Job processOrdersJob(JobRepository jobRepository, Step processOrdersStep) {
        return new JobBuilder("processOrdersJob", jobRepository)
                .start(processOrdersStep)
                .build();
    }
}
```

A chunk size of **3** against our 8-row CSV means the writer fires three times (3 + 3 + 2 rows) — small enough that if you watch the logs or query `BATCH_STEP_EXECUTION` mid-run, you can actually observe multiple commits happening, which is the whole point of building intuition here.

We'll launch the job from Section 10, once we've covered how `JobParameters` should be constructed — launching is inseparable from choosing good parameters, so it belongs there rather than being bolted on here.

### Verifying it worked

Once the job has run (Section 10 shows exactly how), the fastest sanity check is direct SQL — either via a test, or the H2 console at `/h2-console`:

```sql
SELECT * FROM PROCESSED_ORDERS;
```

```
ORDER_ID  CUSTOMER_NAME       AMOUNT   TAX    TOTAL
ORD-001   Alice Johnson       150.00   12.00  162.00
ORD-002   Bob Martinez         89.99    7.20   97.19
ORD-003   Priya Natarajan     432.10   34.57  466.67
...
```

And to see the batch metadata this run produced (tying straight back to Part 2's tables):

```sql
SELECT je.JOB_EXECUTION_ID, je.STATUS, se.STEP_NAME, se.READ_COUNT, se.WRITE_COUNT, se.COMMIT_COUNT
FROM BATCH_JOB_EXECUTION je
JOIN BATCH_STEP_EXECUTION se ON se.JOB_EXECUTION_ID = je.JOB_EXECUTION_ID;
```

You should see `READ_COUNT = 8`, `WRITE_COUNT = 8`, and `COMMIT_COUNT = 3` (one commit per chunk, plus the framework's own bookkeeping commits) — a very concrete, checkable confirmation that chunking is doing exactly what Section 8 described.

---

## 10. Typed `JobParameters` and why they matter for re-runnability

We now have a `Job` bean. We haven't launched it yet, because *how* you construct its `JobParameters` directly determines whether re-running the job is safe, refused, or silently wrong — and that's worth understanding before writing the one line of code that starts it.

### Typed parameters

`JobParameters` in modern Spring Batch (typed `JobParameter<T>`, available since 5.0 and unchanged in 6.x) are not just strings. Each parameter carries its actual type — `String`, `Long`, `Double`, `LocalDate`, `LocalDateTime`, etc. — which means you can query and compare them meaningfully, rather than parsing strings back out by convention.

```java
JobParameters params = new JobParametersBuilder()
        .addString("inputFile", "data/orders.csv", true)
        .addLocalDate("businessDate", LocalDate.of(2026, 7, 13), true)
        .addString("triggeredBy", "manual-run", false)
        .toJobParameters();
```

The third argument to each `add*` method is the important one: **is this parameter identifying?**

### Identifying vs. non-identifying — the double-fire guard in practice

This is the concept flagged back in Part 2, now made concrete:

- **Identifying parameters** (`true`) are part of what defines *which* `JobInstance` this is. Two launches with the same job name and the same identifying parameters refer to the **same** `JobInstance` — and if that instance already completed successfully, Spring Batch refuses to run it again.
- **Non-identifying parameters** (`false`) are just extra data along for the ride — useful for auditing or logging, but they don't affect instance identity. Two launches differing *only* in a non-identifying parameter are still the same `JobInstance`. A common real use: a correlation/tracing ID that's genuinely different on every launch but shouldn't itself define what the job is:

  ```java
  .addString("correlationId", UUID.randomUUID().toString(), false)  // non-identifying
  ```

**The mistake almost everyone makes once:** using the current timestamp as an identifying parameter "to make sure the job always runs."

```java
// DON'T do this — defeats the entire purpose of JobInstance identity
JobParameters params = new JobParametersBuilder()
        .addLocalDateTime("runTimestamp", LocalDateTime.now(), true)  // identifying!
        .toJobParameters();
```

Every launch produces a different timestamp, so every launch becomes a *new* `JobInstance` — Spring Batch's built-in protection against accidental double-runs is completely bypassed, because as far as the framework is concerned, no two runs are ever "the same job" to begin with. If you actually need a timestamp for logging or tracing, add it as **non-identifying** — that gives you the audit trail without disabling the safety net.

The right identifying parameter is almost always something that describes *what data this run is for* — a business date, a file name, a batch/run ID handed to you by an upstream system — not *when the run happened to be launched*.

**Restart vs. new run — this distinction is automatic, not something you code for.** If you launch with the *same* identifying parameters as a `JobInstance` whose last `JobExecution` failed, Spring Batch treats it as a **restart** of that instance and resumes from the last checkpoint (Section 8). If the last execution *completed*, the same parameters get refused. If the parameters are different, it's simply a new, unrelated `JobInstance`. You never have to tell the framework "this is a restart" — it infers that entirely from whether the identifying parameters match a prior instance and what that instance's last execution status was.

**One more thing worth internalizing now, before this goes anywhere near production:** once a job is live, its set of identifying parameters is effectively a contract. Existing `JobInstance` rows in `BATCH_JOB_INSTANCE` are keyed on the current identifying-parameter shape; if you later add, remove, or rename an identifying parameter, previously-run instances won't match the new shape, and you lose the ability to detect "has this logical run already happened?" against your existing history. Treat changes to identifying parameters as you would a schema migration — deliberate, and not done casually.

### Launching the job

With that settled, launching is one line, via the `JobOperator` we covered in Part 2 (not the deprecated `JobLauncher`):

```java
package com.example.batchtutorial;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobOperator;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

import java.time.LocalDate;

@SpringBootApplication
public class BatchTutorialApplication {

    public static void main(String[] args) {
        SpringApplication.run(BatchTutorialApplication.class, args);
    }

    @Bean
    public CommandLineRunner runOrdersJob(JobOperator jobOperator, Job processOrdersJob) {
        return args -> {
            JobParameters params = new JobParametersBuilder()
                    .addString("inputFile", "data/orders.csv", true)
                    .addLocalDate("businessDate", LocalDate.of(2026, 7, 13), true)
                    .toJobParameters();

            jobOperator.start(processOrdersJob, params);
        };
    }
}
```

A `CommandLineRunner` that launches on every application startup is perfect for this tutorial — it makes the job runnable with zero extra plumbing — but it's a demo convenience, not a production pattern. In a real application you'd remove it and trigger the job from something that decides *when* a run should happen: a `@Scheduled` method, Quartz, a message listener, or a REST endpoint that calls `jobOperator.start()` on request. We'll look at scheduling and control more deliberately in Part 4.

### Running it

```bash
mvn spring-boot:run
```

You should see log output ending with something like:

```
o.s.batch.core.job.AbstractJob : Job: [SimpleJob: [name=processOrdersJob]] completed with the following parameters: [{'businessDate':'{value=2026-07-13, type=class java.time.LocalDate, identifying=true}', ...}] and the following status: [COMPLETED]
```

Then verify with the SQL queries shown above, or the H2 console at `http://localhost:8080/h2-console`.

Run it a second time **without changing `businessDate`**, and the launch is rejected — a `JobInstanceAlreadyCompleteException` — which is Spring Batch protecting you from silently double-processing the same day's orders. Change `businessDate` to a new value, and it runs again cleanly as a new `JobInstance`.

That rejection isn't a bug to work around — it's the payoff for choosing identifying parameters correctly, and it's the same mechanism Part 1 called the double-fire guard.

**One more failure-mode worth being explicit about, tying back to Section 8:** if this job failed partway — say, on item 5, mid-chunk — the chunk containing items 4–6 would roll back entirely; items 1–3 (already-committed chunks) would stay written. A restart with the same `businessDate` would resume from chunk 2 (items 4–6 onward), not reprocess items 1–3. The chunk boundary is the restart checkpoint, exactly as described in Section 8 — this mini-project's chunk size of 3 makes that boundary something you could actually go verify in `BATCH_STEP_EXECUTION` if you engineered a failure to test it.

---

### Checkpoint

Before moving to Part 4, you should be able to:

- Explain why an `ItemWriter` receives a list of items rather than one at a time, and how that connects to the commit interval.
- Predict what happens to items already processed in a chunk if a later item in that same chunk throws an exception.
- Explain the difference between an identifying and a non-identifying `JobParameter`, and why using "now" as an identifying parameter is almost always a mistake.
- State what exception you'd expect (and why) from re-launching this mini-project's job with unchanged parameters.

**Next:** Part 4 — Running & Controlling Jobs (`JobOperator.start()` in more depth, restart/retry/skip behavior, and job/step scoping)
