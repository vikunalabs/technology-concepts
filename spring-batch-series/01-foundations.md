# Part 1 — Foundations

*Spring Batch 6.x with JDBC — a small but rich tutorial*

---

## 1. What is Spring Batch, and why does it exist?

To understand Spring Batch, it helps to start with the *shape* of problem it was built to solve, rather than the framework itself.

**Batch applications** process large volumes of data in bulk — on a schedule, or on demand — with no human waiting on the other end of an HTTP request. A few concrete examples make this concrete:

- Reading 2 million rows from a database, transforming each one, and writing them to another system.
- Generating and dispatching thousands of report files every night at 2 AM.
- Importing a vendor's CSV drop, validating every row, rejecting the bad ones, and loading the rest.

It's entirely possible to write this as a `for` loop inside a `main()` method. The reason people move away from that approach is not that the loop is wrong — it's that a small set of predictable failure modes always shows up once the loop meets production:

- The JVM dies at row 1.4 million, and there is no record of where processing had gotten to, so there's no way to resume.
- Someone accidentally runs the same job twice, and now customers have duplicate charges.
- The loop needs to commit every 500 rows for performance, but also needs to roll back cleanly when row 501 fails.
- Ops wants to know, mid-run, how many records have been processed and how many have failed.
- A single row fails validation, and the job needs to skip it, log it, and keep going rather than crash outright.

None of these problems is individually difficult to solve. What makes them worth a framework is that they are the *same* handful of hard-won patterns — chunked processing, transactional commit boundaries, restart from the last good checkpoint, skip/retry policies, execution metadata — recurring across the industry, independent of what any particular job actually does. Spring Batch exists so that these patterns are owned once, by the framework, freeing your own code to focus on *what* to read, transform, and write, rather than *how* to make that reliable.

Put in a single sentence: **Spring Batch gives you a structured, restartable, transactional pipeline** (`Job` → `Step`s → chunks of `Reader`/`Processor`/`Writer`) **that persists exactly what happened at every stage, so the pipeline can recover instead of starting over.**

---

## 2. When *not* to use Spring Batch

Understanding a framework's boundaries is as important as understanding its features, because reaching for Spring Batch in the wrong situation adds ceremony without adding value. The table below works through several situations where that mismatch shows up, and why.

| Situation | Why Spring Batch is a poor fit | What to use instead |
|---|---|---|
| A single, simple `INSERT` or bulk update you run once, manually | The ceremony (Job/Step config, metadata tables, JobRepository) outweighs the benefit | A plain script, or a one-off `@Transactional` method |
| Real-time or low-latency processing (e.g., respond to a user action in under 200ms) | Spring Batch is designed around chunked, throughput-oriented processing, not latency-sensitive request/response | Spring MVC/WebFlux, message-driven consumers |
| Continuous stream processing (unbounded data, no natural "job" boundary) | Batch jobs assume a bounded unit of work with a defined start and end | Spring Cloud Stream, Kafka Streams |
| Tiny, infrequent tasks with no need for restart/retry/skip semantics | The metadata tracking (JobRepository tables, JobInstance uniqueness) is overhead you won't use | `@Scheduled` methods, plain services |
| Sub-second cron-like tasks running very frequently | JobRepository writes on every execution add measurable overhead at high frequency | Lightweight schedulers |
| Triggering a job via REST API with an immediate response expected | Batch jobs are long-running by nature; a synchronous HTTP handler shouldn't block on one | Launch via `JobOperator.start()` from the controller, return a job/execution ID immediately, and let the client poll a status endpoint |

The common thread in the right-hand column is that each alternative is lighter-weight than Spring Batch precisely because it doesn't need what Spring Batch provides. That gives a useful rule of thumb: reach for Spring Batch when you have **bounded, bulk, transactional data processing** that needs to be **resilient to failure and re-runnable**. If restart-from-checkpoint and execution tracking aren't things you need, the framework's overhead has nothing to buy you.

---

## 3. Core domain model: Job → Step → JobInstance → JobExecution → StepExecution

This section covers the part that trips up almost everyone new to the framework — not because any single concept is hard, but because there are five closely related nouns, and it isn't immediately obvious which ones describe *configuration* and which ones describe *runtime history*. Sorting that out is the key that unlocks the rest of the framework, so it's worth slowing down here.

### The configuration side: `Job` and `Step`

Two of the five nouns are *definitions* — reusable blueprints that don't change from run to run:

- **`Job`** is the definition of a batch process, in the same sense that a `class` is a definition. It's composed of one or more `Step`s, executed in order, or according to conditional flow logic.
- **`Step`** is a single, independently manageable phase of a `Job`. Most steps you'll write are **chunk-oriented**: read an item, process it, repeat until a chunk size is reached, then write the whole chunk in one transaction.

The important thing to notice is what a `Job` or `Step` *doesn't* hold: neither one carries any state about a particular run. They describe *how* work should happen, not what has already happened.

### The runtime side: `JobInstance`, `JobExecution`, `StepExecution`

The remaining three nouns describe what happens when a `Job` definition is actually put to work.

- **`JobInstance`** is one **logical run** of a `Job`, identified by the combination of the job's name and its `JobParameters`. "Run the `dailySalesReportJob` for 2026-07-13" is a `JobInstance`. Change the date parameter, and — even though the underlying `Job` definition hasn't changed at all — you get a *different* `JobInstance`.

  `JobParameters` are simply the values that distinguish one `JobInstance` from another: typically a date, a file path, or a run ID. Part 3 covers typed parameters in depth.

  An analogy makes the relationship easier to hold onto: think of a `Job` as a recipe — chocolate cake. A `JobInstance` is making that recipe for a specific occasion (a birthday, 2026-07-13). A `JobExecution`, described next, is each *attempt* to bake it: the first cake burns (`FAILED`), so you try again — a new `JobExecution` — but it's still the same birthday cake, the same `JobInstance`.

- **`JobExecution`** is one **physical attempt** to run a `JobInstance`. If the job fails and is restarted, that restart produces a *second* `JobExecution`, attached to the *same* `JobInstance` as the first. This is the object that carries status (`STARTED`, `COMPLETED`, `FAILED`), start/end timestamps, and exit codes — in other words, it answers "what actually happened when this attempt ran?"

- **`StepExecution`** applies the same idea one level down: it's one attempt to run a `Step`, as part of a specific `JobExecution`. It tracks read count, write count, skip count, commit count, and rollback count for that step.

### Why the split matters

Separating *logical run* (`JobInstance`) from *physical attempt* (`JobExecution`) is precisely what makes **restart** both possible and *safe*. Two consequences follow directly from that split:

- A `JobInstance` can only ever be `COMPLETED` once. If a launch request's parameters match an already-completed `JobInstance`, Spring Batch refuses to proceed. This is a **double-fire guard** you get for free, as long as your parameters are chosen well — a topic Part 3 returns to.
- If a `JobExecution` fails, launching again with the *same* parameters doesn't start a new `JobInstance` — it creates a new `JobExecution` against the *same* one, and, for restartable steps, resumes from the last successfully committed chunk instead of starting over.

Seeing the two attempts and their steps laid out together makes the relationship concrete:

```
Job "dailySalesReportJob"                 (definition — like a class)
 └── JobInstance (name + params: date=2026-07-13)   (a specific logical run)
      ├── JobExecution #1 — FAILED at 02:14am        (first attempt)
      └── JobExecution #2 — COMPLETED at 02:20am      (retry, same JobInstance)
           ├── StepExecution "extractStep"  — COMPLETED
           └── StepExecution "loadStep"     — COMPLETED
```

There is one `JobInstance` here, but two `JobExecution`s, because the first attempt failed. The successful `JobExecution` has two `StepExecution`s beneath it — one per step that ran during that attempt.

The same idea, expressed as a cardinality relationship:

```
Job (1) ──── (0..*) JobInstance (1) ──── (1..*) JobExecution (1) ──── (0..*) StepExecution
```

Reading this left to right: a `Job` can have many `JobInstance`s, one per distinct parameter set. Each `JobInstance` has at least one `JobExecution` — more than one only if earlier attempts failed and were retried. And each `JobExecution` has exactly one `StepExecution` per step that ran during that particular attempt.

---

## 4. The `JobRepository` — what it tracks and why it matters

Everything described above would be meaningless without somewhere durable to record it — that's the role the `JobRepository` plays. Every `JobInstance`, `JobExecution`, `StepExecution`, and their associated `JobParameters` and `ExecutionContext` data get persisted through it — in this tutorial, to a relational database via JDBC.

Concretely, the `JobRepository` is responsible for four things:

1. **Creating and looking up `JobInstance`s.** Given a job name and parameters, it answers the question: has this logical run been seen before?
2. **Persisting `JobExecution` and `StepExecution` state as the job runs, not just at the end.** Status transitions, counts, and timestamps are written incrementally, which is what allows the last-persisted state to survive even if the JVM is killed mid-run.
3. **Storing `ExecutionContext` data** — a key/value bag attached to each execution, used to remember "where we got to" (for example, the last processed ID), so that a step can resume correctly after a restart. Part 5 shows `ExecutionContext` in action, when it's used to pass data between steps.
4. **Enforcing the completed-instance rule** — refusing to create a new `JobExecution` for a `JobInstance` that already completed successfully. This is the mechanism underneath the double-fire guard mentioned above.

One detail worth understanding rather than just memorizing: the transaction that creates a new `JobExecution` runs at `SERIALIZABLE` isolation by default — the strictest level available. This is a deliberate choice, not an oversight. It's what guarantees that if two processes try to launch the same job with the same parameters at the same moment, only one of them wins. Because the create step is short-lived, this stricter-than-usual isolation rarely causes contention in practice, even though it's more aggressive than most of an application's everyday transactions. It's configurable, if a given system ever needs to relax it.

Because all of this lives in a database rather than in memory, a JDBC-backed `JobRepository` gives you three things that an in-memory approach couldn't:

- **Durability across JVM restarts.** Kill the process, restart it, and the framework still knows exactly what happened.
- **Auditability.** Querying `BATCH_JOB_EXECUTION` and its related tables directly answers "did last night's job run, and did it succeed?" — no custom logging table required.
- **A foundation for restart.** Restart logic is only possible *because* there's a durable record of what already completed; without that record, there would be nothing to resume from.

Part 2 opens up the actual tables the JDBC-backed `JobRepository` uses, works through how they relate to one another, and configures one against an H2 database.

---

### A preview

Everything covered so far has been configuration and concepts, with no code yet. To make it a little less abstract before moving on, here's a glimpse of what a `Job` definition actually looks like — Part 3 builds this for real:

```java
@Bean
public Job salesReportJob(JobRepository jobRepository, Step extractAndLoadStep) {
    return new JobBuilder("salesReportJob", jobRepository)
            .start(extractAndLoadStep)
            .build();
}
```

Notice that `jobRepository` is passed directly into the builder. This is not incidental wiring — every `Job` and `Step` in Spring Batch is connected directly to the repository that will track its execution history, which is exactly why the durability guarantees described in Section 4 hold from the moment a job is defined.

### Checkpoint

Before moving on, you should be able to answer:

- What's the difference between a `Job` and a `JobInstance`?
- If a `JobExecution` fails and you re-launch with the same parameters, what happens?
- Why can't a completed `JobInstance` be re-run with the same parameters?
- What does the `JobRepository` actually persist, and why does that matter for restart?

**Next:** [Part 2 — Spring Batch JDBC Setup](./02-jdbc-setup.md)
