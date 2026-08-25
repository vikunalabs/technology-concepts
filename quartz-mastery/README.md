# Quartz Mastery

A deep, production-oriented guide to the Quartz Scheduler in Spring Boot applications — from the four core concepts to clustering, dynamic scheduling, and idempotency. Written for developers who want to actually understand Quartz, not just copy a working configuration.

**Version baseline:** Spring Boot 4.x / Quartz 2.5.x (see the tutorial's own intro for the Spring Boot 3.x note).

## What's here

| File | What it is |
|---|---|
| [`01.quartz-scheduler-tutorial.md`](./01.quartz-scheduler-tutorial.md) | The main tutorial — 13 parts, start to finish |
| [`0x-quartz-scheduler-exercises.md`](./0x-quartz-scheduler-exercises.md) | Hands-on exercises with solutions, keyed to the tutorial's parts |

## Who this is for

You should be comfortable with core Spring Boot concepts (beans, the `ApplicationContext`, dependency injection) and writing everyday Java. No prior Quartz knowledge is assumed — Part 1 starts from the four core concepts, and Part 3 includes a primer on the Spring vocabulary (beans, DI, autoconfiguration) used throughout. If you're already fluent in that, skip straight past it.

Before investing in the rest of the guide, Part 1.7 is worth reading first: it's an honest comparison against Spring's own `@Scheduled`, since Quartz is meaningfully more machinery than some scheduling needs actually require.

## Table of contents

1. **Foundations** — Job, JobDetail, Trigger, Scheduler; identity (name + group); end-to-end scheduling; JobDataMap basics; graceful shutdown; `@Scheduled` vs. Quartz
2. **Triggers in Depth** — SimpleTrigger vs. CronTrigger; misfire instructions; `storeDurably`/`requestRecovery`; the public `TriggerState` enum vs. internal JobStore states
3. **Job Instantiation and the JobFactory** — why the default factories break for constructor-injected jobs, the flawed singleton fix, and the correct `createBean()`-based approach, wired via `SchedulerFactoryBeanCustomizer`
4. **Data Flow: JobDataMap** — job vs. trigger data, merge precedence, the `useProperties` string-only constraint
5. **Persistence and the JDBC JobStore** — schema management, a dedicated `DataSource`, the `overwrite-existing-jobs` redeploy gotcha
6. **Clustering** — how node coordination actually works, the `QRTZ_LOCKS` table, crash recovery
7. **Concurrency Control** — `@DisallowConcurrentExecution`, `@PersistJobDataAfterExecution`, and the sync-vs-async `execute()` pitfall
8. **Putting It Together** — a complete, realistic config shape plus a debugging checklist
9. **Dynamic Scheduling** — creating, rescheduling, pausing, and running jobs at runtime through the `Scheduler` API
10. **Idempotency, Retries, and JobExecutionException** — why `execute()` will run more than once for the same logical work, and how to make that safe
11. **Testing Quartz Jobs** — unit-testing business logic, integration-testing the schedule, cluster and concurrency tests
12. **Listeners** — `JobListener`, `TriggerListener`, `SchedulerListener`, and how to wire them in Spring Boot
13. **Calendars** — the exclusion mechanism for skipping holidays and maintenance windows

Followed by a **Production Rules of Thumb** checklist and a **Where to go from here** pointer to topics deliberately left out (plugins, `JobStoreCMT`, DST behavior, and more).

## How to use it

Read the tutorial in order — later parts assume earlier ones. Each part ends with a "common pitfalls" table you can use as a standalone reference once you've read it in full.

The exercises file currently covers Parts 1–2 (multiple jobs, job data, job groups, cron triggers, misfire simulation, `storeDurably`, request recovery), with solutions included — work through each exercise before expanding its solution.
