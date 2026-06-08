# Cronjob Scheduling — Scheduled Tasks in Spring Boot

## When Scheduled Tasks

Use scheduled tasks for periodic work: daily cleanup, report generation, cache warming, data sync. If it runs on a clock, use `@Scheduled`.

## Step 1: Enable Scheduling

```java
@Configuration
@EnableScheduling
public class SchedulingConfig {
}
```

## Step 2: Schedule Types

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class ScheduledTasks {
    private final CleanupService cleanupService;
    private final ReportService reportService;
    private final CacheService cacheService;

    @Scheduled(cron = "0 0 2 * * *")
    public void nightlyCleanup() {
        log.info("Starting nightly cleanup");
        cleanupService.removeExpiredSessions();
        cleanupService.archiveOldOrders();
    }

    @Scheduled(fixedRate = 30_000)
    public void refreshCache() {
        cacheService.warmProductCache();
    }

    @Scheduled(fixedDelay = 60_000)
    public void syncExternalData() {
        reportService.syncFromExternalApi();
    }

    @Scheduled(cron = "0 0 9 * * MON-FRI")
    public void weekdayReport() {
        reportService.generateDailySummary();
    }
}
```

| Annotation | Meaning |
|------------|---------|
| `cron = "0 0 2 * * *"` | Every day at 2:00 AM |
| `fixedRate = 30_000` | Every 30 seconds from start to start |
| `fixedDelay = 60_000` | 60 seconds after the previous run finishes |

`fixedRate` starts the next run regardless of whether the previous finished. `fixedDelay` waits for the previous run to complete.

## Cron Expression Reference

```
┌─────────── second (0-59)
│ ┌───────── minute (0-59)
│ │ ┌─────── hour (0-23)
│ │ │ ┌───── day of month (1-31)
│ │ │ │ ┌─── month (1-12)
│ │ │ │ │ ┌─ day of week (0-7, 0 and 7 = Sunday)
│ │ │ │ │ │
0 0 2 * * *
```

Common patterns:
- `0 0 * * * *` — every hour
- `0 */15 * * * *` — every 15 minutes
- `0 0 9 * * MON-FRI` — weekdays at 9 AM
- `0 0 0 1 * *` — first day of every month

## Step 3: Distributed Scheduling (Avoid Duplicate Execution)

When running multiple instances, every instance runs the scheduled task. You need a distributed lock.

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>5.10.0</version>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
    <version>5.10.0</version>
</dependency>
```

```java
@Configuration
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class SchedulingConfig {
    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(dataSource);
    }
}
```

```java
@Component
@RequiredArgsConstructor
public class DistributedScheduledTasks {
    private final ReportService reportService;

    @Scheduled(cron = "0 0 2 * * *")
    @SchedulerLock(name = "nightlyReport",
        lockAtLeastFor = "PT5M",
        lockAtMostFor = "PT30M")
    public void nightlyReport() {
        reportService.generateDailySummary();
    }
}
```

Only one instance acquires the lock and runs the task. `lockAtLeastFor` prevents another instance from running it too soon. `lockAtMostFor` is a safety net — if the instance crashes, the lock releases after 30 minutes.

## Step 4: Dynamic Scheduling

```java
@Component
@RequiredArgsConstructor
public class DynamicScheduler {
    private final TaskScheduler taskScheduler;
    private ScheduledFuture<?> currentTask;

    public void schedule(Runnable task, String cronExpression) {
        if (currentTask != null) {
            currentTask.cancel(false);
        }
        currentTask = taskScheduler.schedule(task,
            new CronTrigger(cronExpression));
    }
}
```

## Key Points

- Use `fixedDelay` for tasks that should not overlap; `fixedRate` for independent periodic checks
- In a multi-instance deployment, always use distributed locks (ShedLock)
- Keep scheduled tasks short — offload heavy work to a batch job or async processing
- Cron expressions are powerful but error-prone — test with a cron validator
- Log the start and end of every scheduled task for observability
