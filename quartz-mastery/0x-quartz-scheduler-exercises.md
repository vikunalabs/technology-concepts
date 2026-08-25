## Part 1 — Exercises

Reinforce your understanding of Quartz fundamentals with these hands-on exercises. Complete them in order — each builds on concepts from the previous.

### Exercise 1.1 — Multiple Jobs

**Objective:** Schedule two different job classes independently.

**Tasks:**
1. Create two job classes: `HelloWorldJob` and `GoodbyeJob`
2. Each job should log a distinct message (e.g., "Hello World!" and "Goodbye!")
3. Schedule `HelloWorldJob` to run every 10 seconds
4. Schedule `GoodbyeJob` to run every 15 seconds
5. Run the application and verify both execute at their respective intervals

**Expected output:**
```
HelloWorldJob: Hello World!
GoodbyeJob: Goodbye!
HelloWorldJob: Hello World!
HelloWorldJob: Hello World!
GoodbyeJob: Goodbye!
```

<details>
<summary><strong>Solution</strong></summary>

**HelloWorldJob.java:**
```java
public class HelloWorldJob implements Job {
    private static final Logger log = LoggerFactory.getLogger(HelloWorldJob.class);
    
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("Hello World!");
    }
}
```

**GoodbyeJob.java:**
```java
public class GoodbyeJob implements Job {
    private static final Logger log = LoggerFactory.getLogger(GoodbyeJob.class);
    
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("Goodbye!");
    }
}
```

**Main application:**
```java
JobDetail helloJob = JobBuilder.newJob(HelloWorldJob.class)
        .withIdentity("helloJob", "group1")
        .build();

JobDetail goodbyeJob = JobBuilder.newJob(GoodbyeJob.class)
        .withIdentity("goodbyeJob", "group1")
        .build();

Trigger helloTrigger = TriggerBuilder.newTrigger()
        .withIdentity("helloTrigger", "group1")
        .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                .withIntervalInSeconds(10)
                .repeatForever())
        .build();

Trigger goodbyeTrigger = TriggerBuilder.newTrigger()
        .withIdentity("goodbyeTrigger", "group1")
        .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                .withIntervalInSeconds(15)
                .repeatForever())
        .build();

scheduler.scheduleJob(helloJob, helloTrigger);
scheduler.scheduleJob(goodbyeJob, goodbyeTrigger);
```
</details>

---

### Exercise 1.2 — Job Data

**Objective:** Pass dynamic data to a job using `JobDataMap`.

**Tasks:**
1. Modify `HelloWorldJob` to accept a `message` parameter from job data
2. Log the message instead of a hardcoded string
3. Schedule the same `JobDetail` with two different triggers
4. Each trigger should pass a different message via `usingJobData()`
5. Verify each trigger's message appears correctly

**Expected output:**
```
HelloWorldJob: Hello from Trigger 1!
HelloWorldJob: Hello from Trigger 2!
```

<details>
<summary><strong>Solution</strong></summary>

**HelloWorldJob.java:**
```java
public class HelloWorldJob implements Job {
    private static final Logger log = LoggerFactory.getLogger(HelloWorldJob.class);
    
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        String message = context.getMergedJobDataMap().getString("message");
        log.info("{}", message);
    }
}
```

**Main application:**
```java
JobDetail job = JobBuilder.newJob(HelloWorldJob.class)
        .withIdentity("helloJob", "group1")
        .build();

Trigger trigger1 = TriggerBuilder.newTrigger()
        .withIdentity("trigger1", "group1")
        .usingJobData("message", "Hello from Trigger 1!")
        .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                .withIntervalInSeconds(5)
                .repeatForever())
        .build();

Trigger trigger2 = TriggerBuilder.newTrigger()
        .withIdentity("trigger2", "group1")
        .usingJobData("message", "Hello from Trigger 2!")
        .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                .withIntervalInSeconds(10)
                .repeatForever())
        .build();

scheduler.scheduleJob(job, trigger1);
scheduler.scheduleJob(job, trigger2);
```
</details>

---

### Exercise 1.3 — Job Groups

**Objective:** Use job groups to perform bulk operations.

**Tasks:**
1. Create two jobs in the "reporting" group and one in the "cleanup" group
2. Start the scheduler and let all jobs run
3. Pause all jobs in the "reporting" group using `scheduler.pauseJobs(GroupMatcher.groupEquals("reporting"))`
4. Verify reporting jobs stop while the cleanup job continues
5. Resume the reporting group and verify they restart

**Expected output:**
```
ReportingJob1: Running
ReportingJob2: Running
CleanupJob: Running
// After pause:
CleanupJob: Running
CleanupJob: Running
// After resume:
ReportingJob1: Running
ReportingJob2: Running
```

<details>
<summary><strong>Solution</strong></summary>

**Main application:**
```java
// Create three jobs
JobDetail reportJob1 = JobBuilder.newJob(ReportingJob.class)
        .withIdentity("reportJob1", "reporting")
        .build();

JobDetail reportJob2 = JobBuilder.newJob(ReportingJob.class)
        .withIdentity("reportJob2", "reporting")
        .build();

JobDetail cleanupJob = JobBuilder.newJob(CleanupJob.class)
        .withIdentity("cleanupJob", "cleanup")
        .build();

// Schedule all three
scheduler.scheduleJob(reportJob1, createTrigger("trigger1", 5));
scheduler.scheduleJob(reportJob2, createTrigger("trigger2", 7));
scheduler.scheduleJob(cleanupJob, createTrigger("trigger3", 3));

// Let them run for 15 seconds
Thread.sleep(15000);

// Pause all reporting jobs
scheduler.pauseJobs(GroupMatcher.groupEquals("reporting"));
System.out.println("Reporting jobs paused");

// Let cleanup job continue for 10 seconds
Thread.sleep(10000);

// Resume reporting jobs
scheduler.resumeJobs(GroupMatcher.groupEquals("reporting"));
System.out.println("Reporting jobs resumed");

private static Trigger createTrigger(String identity, int intervalSeconds) {
    return TriggerBuilder.newTrigger()
            .withIdentity(identity, "main")
            .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                    .withIntervalInSeconds(intervalSeconds)
                    .repeatForever())
            .build();
}
```

**ReportingJob.java:**
```java
public class ReportingJob implements Job {
    private static final Logger log = LoggerFactory.getLogger(ReportingJob.class);
    
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("Reporting Job {} running", context.getJobDetail().getKey().getName());
    }
}
```

**CleanupJob.java:**
```java
public class CleanupJob implements Job {
    private static final Logger log = LoggerFactory.getLogger(CleanupJob.class);
    
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("Cleanup job running");
    }
}
```
</details>

---

## Part 2 — Exercises

### Exercise 2.1 — CronTrigger

**Objective:** Replace SimpleTrigger with CronTrigger.

**Tasks:**
1. Schedule a job using a cron expression that runs every 5 seconds for testing
2. Change it to run every weekday at 9:15 AM (just log the schedule, don't wait for it)
3. Run the scheduler and verify the first schedule works

**Expected output:**
```
HelloWorldJob: Running at 2024-01-15T14:25:30
HelloWorldJob: Running at 2024-01-15T14:25:35
HelloWorldJob: Running at 2024-01-15T14:25:40
```

<details>
<summary><strong>Solution</strong></summary>

```java
// For testing - runs every 5 seconds
Trigger testTrigger = TriggerBuilder.newTrigger()
        .withIdentity("testTrigger", "test")
        .withSchedule(CronScheduleBuilder.cronSchedule("*/5 * * * * ?"))
        .build();

// For production - every weekday at 9:15 AM
Trigger productionTrigger = TriggerBuilder.newTrigger()
        .withIdentity("productionTrigger", "prod")
        .withSchedule(CronScheduleBuilder.cronSchedule("0 15 9 ? * MON-FRI"))
        .build();

// Use the test trigger for immediate verification
scheduler.scheduleJob(job, testTrigger);
```
</details>

---

### Exercise 2.2 — Misfire Simulation

**Objective:** Understand misfire behavior and how to configure it.

**Tasks:**
1. Create a job that sleeps for 10 seconds
2. Annotate it with `@DisallowConcurrentExecution`
3. Schedule it with a 5-second interval
4. Observe the misfire behavior with default settings
5. Change the misfire instruction to `withMisfireHandlingInstructionFireNow()`
6. Observe the difference

**Expected output:**
```
// Without misfire handling
Job started
Job finished (10s later)
// Misfire occurs, default behavior depends on trigger type

// With fireNow
Job started
Job finished
// Next firing happens immediately to catch up
```

<details>
<summary><strong>Solution</strong></summary>

**SlowJob.java:**
```java
@DisallowConcurrentExecution
public class SlowJob implements Job {
    private static final Logger log = LoggerFactory.getLogger(SlowJob.class);
    
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("Slow job started");
        try {
            Thread.sleep(10000); // 10 seconds
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        log.info("Slow job finished");
    }
}
```

**Main application with misfire handling:**
```java
JobDetail slowJob = JobBuilder.newJob(SlowJob.class)
        .withIdentity("slowJob", "test")
        .build();

// Without explicit misfire handling
Trigger trigger1 = TriggerBuilder.newTrigger()
        .withIdentity("trigger1", "test")
        .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                .withIntervalInSeconds(5)
                .repeatForever())
        .build();

// With fireNow misfire handling
Trigger trigger2 = TriggerBuilder.newTrigger()
        .withIdentity("trigger2", "test")
        .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                .withIntervalInSeconds(5)
                .repeatForever()
                .withMisfireHandlingInstructionFireNow())
        .build();

scheduler.scheduleJob(slowJob, trigger2);
```
</details>

---

### Exercise 2.3 — storeDurably

**Objective:** Understand the impact of `storeDurably()` on job lifecycle.

**Tasks:**
1. Create a `JobDetail` with `storeDurably(false)` (default)
2. Schedule it with a trigger, then delete the trigger
3. Check if the job still exists
4. Repeat with `storeDurably(true)` and verify the job persists

**Expected output:**
```
// With storeDurably(false)
Job exists: true
Deleted trigger
Job exists: false // Job automatically removed

// With storeDurably(true)
Job exists: true
Deleted trigger
Job exists: true // Job remains
```

<details>
<summary><strong>Solution</strong></summary>

```java
// Test 1: storeDurably(false) - default
JobDetail job1 = JobBuilder.newJob(HelloWorldJob.class)
        .withIdentity("job1", "test")
        .build();

Trigger trigger1 = TriggerBuilder.newTrigger()
        .withIdentity("trigger1", "test")
        .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                .withIntervalInSeconds(10)
                .repeatForever())
        .build();

scheduler.scheduleJob(job1, trigger1);
System.out.println("Job1 exists: " + scheduler.checkExists(new JobKey("job1", "test")));

// Unscheduling the job
scheduler.unscheduleJob(new TriggerKey("trigger1", "test"));
System.out.println("Job1 exists after unschedule: " + scheduler.checkExists(new JobKey("job1", "test")));

// Test 2: storeDurably(true)
JobDetail job2 = JobBuilder.newJob(HelloWorldJob.class)
        .withIdentity("job2", "test")
        .storeDurably(true)
        .build();

// Must add job separately when using storeDurably(true)
scheduler.addJob(job2, true);

Trigger trigger2 = TriggerBuilder.newTrigger()
        .withIdentity("trigger2", "test")
        .forJob(job2)
        .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                .withIntervalInSeconds(10)
                .repeatForever())
        .build();

scheduler.scheduleJob(trigger2);
System.out.println("Job2 exists: " + scheduler.checkExists(new JobKey("job2", "test")));

scheduler.unscheduleJob(new TriggerKey("trigger2", "test"));
System.out.println("Job2 exists after unschedule: " + scheduler.checkExists(new JobKey("job2", "test")));
```
</details>

---

### Exercise 2.4 — Request Recovery

**Objective:** Understand how `requestRecovery()` handles scheduler failures.

**Tasks:**
1. Create a job that logs start, sleeps for 30 seconds, then logs finish
2. Schedule it with `requestRecovery(true)`
3. Kill the scheduler mid-execution (Ctrl+C)
4. Restart the application and observe if the job recovers

**Note:** This exercise requires a persistent job store (JDBC). With `RAMJobStore`, recovery doesn't work across restarts.

<details>
<summary><strong>Solution</strong></summary>

**RecoverableJob.java:**
```java
public class RecoverableJob implements Job {
    private static final Logger log = LoggerFactory.getLogger(RecoverableJob.class);
    
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("Recoverable job started");
        try {
            Thread.sleep(30000); // 30 seconds
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        log.info("Recoverable job finished");
    }
}
```

**Main application:**
```java
JobDetail recoverableJob = JobBuilder.newJob(RecoverableJob.class)
        .withIdentity("recoverableJob", "test")
        .requestRecovery(true)  // Critical for recovery
        .storeDurably(true)     // Needed for JDBC store
        .build();

Trigger trigger = TriggerBuilder.newTrigger()
        .withIdentity("recoverableTrigger", "test")
        .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                .withIntervalInSeconds(60)
                .repeatForever())
        .build();

scheduler.scheduleJob(recoverableJob, trigger);
```
</details>
```

## Summary of New Key Concepts Covered

**Part 1:**
- Multiple jobs with different schedules
- Passing dynamic data via `JobDataMap`
- Using job groups for bulk operations

**Part 2:**
- Cron expressions for complex schedules
- Misfire handling and why it matters
- `storeDurably()` and its impact on job lifecycle
- `requestRecovery()` for handling scheduler failures
