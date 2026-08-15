# Quartz Scheduler — From Basics to Advanced

A ground-up tutorial on the Quartz Job Scheduling library, with a strong focus on how it integrates with Spring Boot — using patterns drawn from a real clustered, JDBC-backed report-scheduling system along the way.

---

## Part 1 — Foundations

### 1.1 What Quartz actually is

Quartz is a standalone Java scheduling library. It existed long before Spring, before dependency injection was standard, before most of the frameworks you use today. This matters more than it sounds like it should — a lot of Quartz's design (its default instantiation model, its data-map-of-strings mentality) makes much more sense once you remember it was built to work in *any* Java application, not just Spring ones.

At its core, Quartz answers one question: **"run this piece of code, at this time (or these times)."** Everything else — clustering, persistence, misfire handling, listeners — exists in service of that one job.

### 1.2 The four core concepts

| Concept | What it is | Analogy |
|---|---|---|
| **Job** | The actual work to perform — a class implementing `org.quartz.Job` | A recipe |
| **JobDetail** | Metadata describing a Job: its class, identity, data, durability | A recipe card with notes |
| **Trigger** | *When* to fire a JobDetail — cron expression, simple interval, etc. | An alarm clock |
| **Scheduler** | The engine that ties triggers to jobs and fires them | The kitchen, running everything |

The critical relationship: **a `JobDetail` is not a `Job` instance.** It's a *description* of one. Quartz creates a fresh `Job` instance every time a trigger fires (more on this in Part 3 — it's the source of a very common integration bug). Multiple `Trigger`s can point at the same `JobDetail`, each firing it on a different schedule.

```java
public class HelloJob implements Job {
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        System.out.println("Hello, Quartz!");
    }
}
```

That's the entire contract: implement `execute()`. Quartz calls it; you do work; you throw `JobExecutionException` if something goes wrong.

### 1.3 Identity: name + group

Every `JobDetail` and every `Trigger` has an identity made of a **name** and a **group** — two strings, together forming a `JobKey` or `TriggerKey`. Groups exist so you can namespace and bulk-operate on related jobs ("pause the entire `reporting` group") without every job needing a globally unique name.

```java
JobDetail job = JobBuilder.newJob(HelloJob.class)
        .withIdentity("myJob", "myGroup")
        .build();
```

If you don't specify a group, Quartz uses `"DEFAULT"`. In any system with more than one kind of scheduled work, treat groups as load-bearing from day one — retrofitting them later means touching every job/trigger identity in your system.

### 1.4 Building and scheduling, end to end

```java
Scheduler scheduler = new StdSchedulerFactory().getScheduler();
scheduler.start();

JobDetail job = JobBuilder.newJob(HelloJob.class)
        .withIdentity("myJob", "myGroup")
        .build();

Trigger trigger = TriggerBuilder.newTrigger()
        .withIdentity("myTrigger", "myGroup")
        .startNow()
        .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                .withIntervalInSeconds(10)
                .repeatForever())
        .build();

scheduler.scheduleJob(job, trigger);
```

Five steps, always in this order: get a scheduler → build a `JobDetail` → build a `Trigger` → schedule them together → start the scheduler (or start it first — order between building and starting doesn't matter, but nothing fires until `start()` is called).

---

## Part 2 — Triggers in Depth

### 2.1 SimpleTrigger vs CronTrigger

These are the two trigger types you'll use 95% of the time.

**`SimpleTrigger`** — fire once, or repeat at a fixed interval, a fixed number of times (or forever):
```java
SimpleScheduleBuilder.simpleSchedule()
        .withIntervalInMinutes(5)
        .repeatForever();
```

**`CronTrigger`** — fire according to a cron expression, for calendar-based schedules ("every weekday at 9am", "the 1st of every month"):
```java
CronScheduleBuilder.cronSchedule("0 0 9 ? * MON-FRI");
```

Quartz cron expressions have **7 fields**, not 5 or 6 like Unix cron — this trips up almost everyone the first time:

```
seconds  minutes  hours  day-of-month  month  day-of-week  [year]
   0        0        9        ?          *        MON-FRI
```

The `?` is Quartz-specific: it means "no specific value," and you're required to use it in exactly one of `day-of-month`/`day-of-week` — because specifying both would be contradictory (which one wins if the 15th isn't a Monday?), Quartz forces you to leave one of them unspecified.

### 2.2 Misfires — the concept most people skip and later regret

A **misfire** happens when a trigger's scheduled fire time passes without it actually firing — because the scheduler was down, all worker threads were busy, or the previous firing of a `@DisallowConcurrentExecution` job was still running.

Every trigger type has a **misfire instruction** governing what happens next. For `CronTrigger`, the two you'll actually use:

- `withMisfireHandlingInstructionFireAndProceed()` — fire once immediately to "catch up," then resume the normal schedule.
- `withMisfireHandlingInstructionDoNothing()` — skip the missed firing entirely, wait for the next naturally scheduled time.

There's no universally "correct" choice — it depends on whether a missed firing represents lost work that must happen eventually, or a snapshot that's only meaningful at its exact scheduled moment. A "generate today's report" job usually wants `FireAndProceed` (the report still needs to exist); a "sample current queue depth every minute" job usually wants `DoNothing` (a late sample of a now-different queue depth isn't useful). **Pick deliberately per job — don't leave it at Quartz's default and hope.**

### 2.3 `storeDurably` and `requestRecovery`

Two `JobBuilder` flags that matter far more than their one-line names suggest:

```java
JobBuilder.newJob(MyJob.class)
        .withIdentity("myJob", "myGroup")
        .storeDurably(true)
        .requestRecovery(true)
        .build();
```

- **`storeDurably(true)`** — by default, Quartz deletes a `JobDetail` from its store once it has no triggers pointing at it. This is almost never what you want for application-managed jobs (imagine losing your job definition because someone temporarily removed its trigger). Setting this to `true` tells Quartz "keep this `JobDetail` around even with zero triggers."
- **`requestRecovery(true)`** — if the scheduler process dies *mid-execution* of this job, should Quartz re-run it on restart? This only matters for jobs where "started but didn't finish" leaves things in a state that's safe (or necessary) to redo — it should line up with whether your job's own logic is idempotent/restart-safe, not be flipped on reflexively for every job.

---

## Part 3 — Job Instantiation and the JobFactory (the part that bites Spring users)

This is the single most common integration failure when wiring Quartz into a DI framework, so it gets its own Part.

### 3.1 The problem, stated precisely

Recall from 1.2: **Quartz creates a new `Job` instance every time a trigger fires.** It does this through a `JobFactory`. The default implementation, `SimpleJobFactory`, does the simplest possible thing:

```java
// conceptually what SimpleJobFactory does
Class<? extends Job> jobClass = bundle.getJobDetail().getJobClass();
return jobClass.newInstance();   // calls the no-arg constructor
```

That's fine for a `HelloJob` with no dependencies. It's fatal for any real-world job class that needs constructor-injected collaborators:

```java
@Component
@RequiredArgsConstructor
public class ReportSchedulingJob implements Job {
    private final ReportPipelineTrigger reportPipelineTrigger; // no no-arg constructor exists
    ...
}
```

`SimpleJobFactory.newInstance()` has no way to supply `reportPipelineTrigger`. There is no no-arg constructor to even call. The job fails to instantiate the moment its trigger first fires — not at application startup, which makes this an especially nasty class of bug: everything looks fine when the app boots, and it only blows up later when a trigger actually fires.

### 3.2 The fix: a Spring-aware JobFactory

Spring provides `SpringBeanJobFactory`, which instead:

1. Instantiates the job via its no-arg constructor (same as before) — **or**
2. If the job class happens to already be a bean in the `ApplicationContext`, hands you that Spring-managed bean instead — fully constructed, fully wired.
3. Either way, applies Spring's autowiring machinery over the result for any remaining `@Autowired` fields.

A common pattern (and the one Commander uses) subclasses it to make the "already a Spring bean" path the deliberate, explicit first choice, rather than an implicit side effect:

```java
@Component
public class AutowiringSpringBeanJobFactory extends SpringBeanJobFactory
        implements ApplicationContextAware {

    private ApplicationContext applicationContext;
    private AutowireCapableBeanFactory beanFactory;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
        this.beanFactory = applicationContext.getAutowireCapableBeanFactory();
    }

    @Override
    protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
        Class<?> jobClass = bundle.getJobDetail().getJobClass();
        if (applicationContext.getBeanNamesForType(jobClass).length > 0) {
            return applicationContext.getBean(jobClass);   // fully Spring-managed
        }
        Object job = super.createJobInstance(bundle);       // fallback: reflective + autowire
        beanFactory.autowireBean(job);
        return job;
    }
}
```

### 3.3 The step everyone forgets: actually wiring the factory in

Writing the `JobFactory` class is necessary but **not sufficient**. Quartz's `Scheduler` has to be told to use it:

```java
schedulerFactoryBean.setJobFactory(jobFactory);
```

Here's the critical, easy-to-miss fact, confirmed straight from Spring Boot's own `QuartzAutoConfiguration` source:

```java
@Bean
@ConditionalOnMissingBean
public SchedulerFactoryBean quartzScheduler(QuartzProperties properties,
        ObjectProvider<SchedulerFactoryBeanCustomizer> customizers,
        ObjectProvider<JobDetail> jobDetails, Map<String, Calendar> calendars,
        ObjectProvider<Trigger> triggers, ApplicationContext applicationContext) {
    SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
    SpringBeanJobFactory jobFactory = new SpringBeanJobFactory();
    jobFactory.setApplicationContext(applicationContext);
    schedulerFactoryBean.setJobFactory(jobFactory);
    // ...
}
```

Spring Boot's autoconfiguration **auto-detects and wires in every `JobDetail` bean and every `Trigger` bean** in your context automatically — that part is real, and documented. But `JobFactory` is not on that auto-detected list. Instead, Spring Boot's own `quartzScheduler()` method **constructs its own plain `SpringBeanJobFactory` right there and calls `setJobFactory()` on it, unconditionally** — before your `SchedulerFactoryBeanCustomizer` beans, if any, ever run.

The practical consequence: **declaring your `AutowiringSpringBeanJobFactory` as a `@Component` does nothing by itself.** Spring Boot still builds its own default, plain `SpringBeanJobFactory` and installs *that*. Your custom factory bean just sits in the context, unused, unless you explicitly override the scheduler's job factory — typically via a `SchedulerFactoryBeanCustomizer`:

```java
@Bean
public SchedulerFactoryBeanCustomizer jobFactoryCustomizer(AutowiringSpringBeanJobFactory jobFactory) {
    return schedulerFactoryBean -> schedulerFactoryBean.setJobFactory(jobFactory);
}
```

`SchedulerFactoryBeanCustomizer` beans run *after* Spring Boot's own `quartzScheduler()` method has already set its default factory — so a customizer is exactly the mechanism designed to let you override it afterward. This is also why job classes with constructor injection and no no-arg constructor will build and boot successfully, and only fail the moment a trigger actually fires — until this customizer is added, Spring Boot's own default (plain, non-autowiring) `SpringBeanJobFactory` is quietly the one doing the instantiating.

**Rule of thumb:** if any of your `Job` classes use constructor injection (which almost all real ones will, especially with Lombok's `@RequiredArgsConstructor`), you *must* explicitly wire a Spring-aware `JobFactory` in via a customizer. It is never automatic.

---

## Part 4 — Data Flow: JobDataMap

### 4.1 Job data vs trigger data

Both `JobDetail` and `Trigger` carry their own `JobDataMap` — a `Map<String, Object>`-like structure for passing parameters into `execute()`. They're merged at execution time (trigger data takes precedence on key collisions), and accessed like this:

```java
public void execute(JobExecutionContext context) {
    JobDataMap jobData = context.getJobDetail().getJobDataMap();
    JobDataMap triggerData = context.getTrigger().getJobDataMap();
    String reportType = jobData.getString("reportType");
}
```

The distinction matters: **job-level data is shared by every trigger firing that `JobDetail`**, while **trigger-level data is specific to that one trigger**. A common pattern (used in Commander's `ReportJobScheduleBuilder`): a single `JobDetail` per report type carries `reportType`/`reportFrequency` in its job data, while several triggers pointing at it — one per daily "window" boundary — each carry their own `windowSequence` in trigger data to distinguish which slot fired.

### 4.2 `useProperties` — the string-only constraint

By default, `JobDataMap` accepts any `Object`. But if you configure Quartz with:

```
org.quartz.jobStore.useProperties = true
```

...every value in every `JobDataMap` **must be a `String`**. This is a JDBC-JobStore-specific setting: when job data is persisted to a database (see Part 5), Quartz needs to serialize it somehow. `useProperties=true` stores it as flat string key-value pairs instead of Java serialization — which is far more robust across app redeploys (no `serialVersionUID` mismatches breaking a persisted trigger) and human-readable in the database. The tradeoff is exactly the constraint above: no storing a `LocalDateTime` or an `Instant` directly in job data — encode it as a string and parse it back out in `execute()`.

---

## Part 5 — Persistence and the JDBC JobStore

### 5.1 Why persistence matters

By default, Quartz keeps everything — jobs, triggers, schedule state — in memory (`RAMJobStore`). That's fine for a toy app; it's unacceptable for anything real, because **a process restart loses every scheduled job**. The `JDBCJobStore` persists all of it to a relational database, so scheduling state survives restarts and (critically) can be shared across multiple application instances — see Part 6.

### 5.2 Enabling it

```yaml
spring:
  quartz:
    job-store-type: jdbc
    jdbc:
      initialize-schema: never   # you control schema migration, e.g. via Flyway/Liquibase
```

Quartz ships SQL scripts (`tables_*.sql`, one per database vendor) defining a substantial schema — `QRTZ_JOB_DETAILS`, `QRTZ_TRIGGERS`, `QRTZ_CRON_TRIGGERS`, `QRTZ_SIMPLE_TRIGGERS`, `QRTZ_FIRED_TRIGGERS`, `QRTZ_LOCKS`, and more. In any application already managing its own schema migrations, run these once through your own migration tool rather than letting Quartz auto-initialize — `initialize-schema: never` above is exactly that choice.

### 5.3 A dedicated DataSource for Quartz

A pattern worth knowing even before you need it: giving Quartz's `JDBCJobStore` its **own connection pool**, separate from the application's primary `DataSource`:

```java
@Bean
@QuartzDataSource
@ConfigurationProperties("commander.quartz.datasource.hikari")
public DataSource quartzDataSource(DataSourceProperties props) {
    return props.initializeDataSourceBuilder()
            .type(HikariDataSource.class)
            .build();
}
```

The `@QuartzDataSource` qualifier tells Spring Boot's Quartz autoconfiguration to use *this* pool instead of the primary one. Why bother? Quartz's clustered lock acquisition (Part 6) involves frequent short-lived polling queries competing for connections. Under load, if Quartz shares a pool with application traffic, the two can starve each other — a burst of report-generation queries can delay Quartz's own lock check-ins (risking missed fires), or vice versa. A separate, appropriately-sized pool keeps the two workloads from ever fighting over the same limited connections.

---

## Part 6 — Clustering

### 6.1 The problem clustering solves

Run two instances of the same application, both with Quartz configured against the same JDBC job store, and — without clustering support — **both instances will independently fire the same trigger at the same time.** Your "send the daily report" job now runs twice, sends duplicate emails, or double-processes a batch.

### 6.2 How Quartz clustering actually works

Enable it with:

```yaml
spring:
  quartz:
    properties:
      org:
        quartz:
          jobStore:
            isClustered: true
          scheduler:
            instanceId: AUTO
```

Every clustered node polls the shared database on an interval. When a trigger's fire time arrives, nodes race to **acquire a lock** on that trigger via the `QRTZ_LOCKS` table (a `SELECT ... FOR UPDATE`-style row lock, using whatever your JDBC driver's locking support provides). Exactly one node wins the lock, fires the trigger, and releases it. Every other node sees the trigger as already claimed and moves on. This is why the `commander.quartz.datasource` connection pool matters (5.3) — this lock-acquisition polling happens constantly, on every node, all the time the scheduler is running.

`instanceId: AUTO` tells Quartz to generate a unique ID per node (typically host name + timestamp) — required for clustering, since nodes need distinguishable identities to know which fired-trigger records are "theirs" during crash recovery.

### 6.3 Crash recovery in a cluster

If a node dies mid-execution of a job with `requestRecovery(true)` (2.3), another node in the cluster detects the abandoned "fired trigger" record (via the `QRTZ_FIRED_TRIGGERS` table) after a timeout and takes over — either re-firing the job or marking it failed, depending on configuration. This is the other half of why `requestRecovery` matters in a clustered deployment specifically: without it, a job orphaned by a crashed node just silently never completes and nothing else in the cluster picks it up.

---

## Part 7 — Concurrency Control

### 7.1 `@DisallowConcurrentExecution`

By default, if a trigger fires again while the *previous* firing of the same `JobDetail` is still running, Quartz happily starts a second concurrent execution. For most stateful or resource-bound jobs, that's dangerous. Annotate the `Job` class:

```java
@DisallowConcurrentExecution
public class ReportSchedulingJob implements Job {
```

Now Quartz will block a new firing of this `JobDetail` from starting until the current one finishes. Note the granularity: this is enforced **per `JobDetail`**, not per class — two different `JobDetail`s both backed by the same `Job` class can still run concurrently with each other; it's firings of the *same* `JobDetail` that get serialized.

### 7.2 `@PersistJobDataAfterExecution`

A related annotation: if your job mutates its own `JobDataMap` during `execute()` (e.g. incrementing a counter) and you want that change persisted back to the job store, add `@PersistJobDataAfterExecution`. It's commonly paired with `@DisallowConcurrentExecution`, since mutating shared job data safely generally also requires that no second execution is running concurrently.

### 7.3 Synchronous vs asynchronous job launches

Worth calling out because it's a real design decision, not just a Quartz mechanic: if your `Job.execute()` kicks off further async work (e.g. launching a Spring Batch job) and returns immediately, Quartz considers the *trigger firing* complete the moment `execute()` returns — not when the async work finishes. That has two consequences: `@DisallowConcurrentExecution` stops meaning "one at a time" for the actual underlying work (only for the thin synchronous wrapper), and Quartz's own misfire bookkeeping for the *next* scheduled fire time gets computed against the wrapper's completion, not the real work's. If real completion matters for either of those, `execute()` needs to block until the underlying work is actually done, i.e. behave synchronously — even if the thing it launches offers an async API.

---

## Part 8 — Putting It Together: A Realistic Config Shape

Tying the previous parts into one coherent shape (structurally close to a real clustered, JDBC-backed setup):

```java
@Configuration
public class QuartzSchedulerConfig {

    @Bean
    public JobDetail reportJobDetail() {
        return JobBuilder.newJob(ReportSchedulingJob.class)
                .withIdentity("dailyReport", "reporting")
                .storeDurably(true)
                .requestRecovery(true)
                .build();
    }

    @Bean
    public Trigger reportTrigger(JobDetail reportJobDetail) {
        return TriggerBuilder.newTrigger()
                .forJob(reportJobDetail)
                .withIdentity("dailyReport-trigger", "reporting")
                .withSchedule(CronScheduleBuilder.cronSchedule("0 0 6 * * ?")
                        .inTimeZone(TimeZone.getTimeZone("America/New_York"))
                        .withMisfireHandlingInstructionFireAndProceed())
                .build();
    }

    // JobDetail and Trigger beans above are auto-detected by Spring Boot.
    // JobFactory is NOT auto-detected -- must be set explicitly (Part 3.3).
    @Bean
    public SchedulerFactoryBeanCustomizer jobFactoryCustomizer(AutowiringSpringBeanJobFactory factory) {
        return bean -> bean.setJobFactory(factory);
    }
}
```

```yaml
spring:
  quartz:
    job-store-type: jdbc
    jdbc:
      initialize-schema: never
    properties:
      org:
        quartz:
          jobStore:
            isClustered: true
            useProperties: true
          scheduler:
            instanceId: AUTO
```

Read as a checklist, this is also a decent debugging sequence when a Quartz-based scheduler misbehaves: job/trigger not firing at all → check `JobDetail`/`Trigger` bean registration and misfire instructions (Parts 1–2). Job throws on instantiation the first time it fires → check `JobFactory` wiring (Part 3). Job runs twice → check `@DisallowConcurrentExecution` and clustering config (Parts 6–7). Schedule vanishes after a redeploy → check `storeDurably` and whether you're accidentally on `RAMJobStore` (Parts 3, 5).

---

## Part 9 — Where to Go Next

- **Listeners** (`JobListener`, `TriggerListener`, `SchedulerListener`) — hook into job/trigger lifecycle events (before/after fire, misfire, veto) without modifying the job itself. Useful for cross-cutting concerns like metrics or alerting.
- **Calendars** (`org.quartz.Calendar`, not `java.util.Calendar`) — exclude specific date/time ranges from a trigger's schedule (e.g. "run daily, except on these holidays").
- **Plugins** — Quartz's own extension mechanism (e.g. `ShutdownHookPlugin`) for scheduler-level behavior, distinct from jobs/triggers.
- **The Spring Batch + Quartz boundary** — once you're comfortable with everything above, the natural next question is exactly the one that prompted this tutorial: how much responsibility should live in the `Job.execute()` method versus a separate, Quartz-agnostic "trigger" seam it delegates to. That's a design question this tutorial deliberately left for you to reason through with the concrete mechanics now in hand.
