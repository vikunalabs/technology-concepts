# Technology Concepts

Deep-dive, production-oriented tutorials on individual technologies — each one written to build real understanding (the "why," not just a working config) rather than serve as a quick-start.

## Series

| Series | Status | Description |
|---|---|---|
| [`quartz-mastery/`](./quartz-mastery) | 13 parts, in progress | The Quartz Scheduler in Spring Boot — jobs, triggers, misfires, the `JobFactory`/DI integration, JDBC persistence, clustering, dynamic scheduling, idempotency, listeners, and calendars. |
| [`kubernetes-mastery-series/`](./kubernetes-mastery-series) | 10-part plan | Kubernetes for Spring Boot developers, from a first Docker/Helm deploy through networking, storage, security, observability, scaling, and a production playbook. |
| [`spring-batch-series/`](./spring-batch-series) | 3 parts, in progress | Spring Batch 6.x with JDBC — chunked processing, the `JobRepository`, and a mini project applying both. |

## Conventions

- Each series lives in its own top-level directory and is self-contained.
- Numbered files (`01-...`, `02-...`) are meant to be read in order — later parts assume earlier ones.
- Where a series has its own `README.md`, that's the place to start for its table of contents and prerequisites.
