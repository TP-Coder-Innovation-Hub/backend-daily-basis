# Thread Pool — Managing Concurrent Work

## Why Thread Pools

Without a thread pool, every concurrent task spawns a new thread. 10,000 concurrent requests = 10,000 threads = OutOfMemoryError. A thread pool reuses a fixed set of threads, queuing excess work.

## Step 1: Configure a Thread Pool

```java
@Configuration
public class ThreadPoolConfig {
    @Bean("apiCallExecutor")
    public ThreadPoolTaskExecutor apiCallExecutor() {
        var executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("api-call-");
        executor.setRejectedExecutionHandler(
            new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.initialize();
        return executor;
    }

    @Bean("reportExecutor")
    public ThreadPoolTaskExecutor reportExecutor() {
        var executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("report-");
        executor.initialize();
        return executor;
    }
}
```

## How Pool Sizing Works

```
1. New task arrives
2. If active threads < corePoolSize → create new thread
3. If active threads >= corePoolSize → add to queue
4. If queue is full and active threads < maxPoolSize → create new thread
5. If queue is full and active threads >= maxPoolSize → rejection policy
```

## Rejection Policies

| Policy | Behavior |
|--------|----------|
| `CallerRunsPolicy` | The submitting thread runs the task (backpressure) |
| `AbortPolicy` | Throw `RejectedExecutionException` |
| `DiscardPolicy` | Silently drop the task |
| `DiscardOldestPolicy` | Drop the oldest queued task |

`CallerRunsPolicy` is usually best — it provides natural backpressure instead of failing.

## Step 2: Parallel API Calls

```java
@Service
@RequiredArgsConstructor
public class AggregationService {
    private final RestClient restClient;
    @Qualifier("apiCallExecutor")
    private final ThreadPoolTaskExecutor executor;

    public AggregatedData fetchAll(String userId) {
        var startTime = Instant.now();

        CompletableFuture<UserProfile> profileFuture =
            CompletableFuture.supplyAsync(
                () -> fetchProfile(userId), executor);
        CompletableFuture<List<Order>> ordersFuture =
            CompletableFuture.supplyAsync(
                () -> fetchOrders(userId), executor);
        CompletableFuture<List<Recommendation>> recsFuture =
            CompletableFuture.supplyAsync(
                () -> fetchRecommendations(userId), executor);

        CompletableFuture.allOf(profileFuture, ordersFuture, recsFuture)
            .join();

        return new AggregatedData(
            profileFuture.join(),
            ordersFuture.join(),
            recsFuture.join(),
            Duration.between(startTime, Instant.now()));
    }
}
```

Three API calls in parallel instead of sequentially. Total time = max(individual times) instead of sum.

## Step 3: @Async with Custom Executor

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ReportService {
    @Async("reportExecutor")
    public CompletableFuture<Report> generateReport(ReportRequest request) {
        log.info("Generating report on thread: {}",
            Thread.currentThread().getName());
        var report = buildReport(request);
        return CompletableFuture.completedFuture(report);
    }
}
```

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        var executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

## Tuning Guidelines

| Workload Type | CPU-bound | I/O-bound |
|---------------|-----------|-----------|
| Core pool size | CPU cores | 2x CPU cores |
| Max pool size | CPU cores + 1 | 4-8x CPU cores |
| Queue capacity | Small (10-50) | Larger (100-500) |

CPU-bound tasks (computations) need fewer threads. I/O-bound tasks (HTTP calls, DB queries) spend most time waiting, so more threads help.

## Key Points

- Always use thread pools for concurrent work — unbounded threads crash the JVM
- Size pools based on workload: small for CPU-bound, large for I/O-bound
- `CallerRunsPolicy` provides backpressure without losing work
- Use separate pools for different workload types (API calls vs. report generation)
- Name your threads (`setThreadNamePrefix`) for easier debugging in thread dumps
