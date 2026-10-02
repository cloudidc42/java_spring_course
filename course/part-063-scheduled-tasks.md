# Part 063: Scheduled Tasks and Background Jobs

## Overview

Every production system needs background jobs: sending emails, generating reports, cleaning up expired data, synchronizing with external systems. This part covers every scheduling mechanism available in Spring Boot — from simple `@Scheduled` to distributed ShedLock and enterprise-grade Quartz Scheduler.

---

## Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- ShedLock for distributed scheduling -->
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
    <!-- Quartz Scheduler -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-quartz</artifactId>
    </dependency>
    <!-- Spring Batch -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-batch</artifactId>
    </dependency>
</dependencies>
```

---

## 1. @Scheduled Annotation

```java
// Enable scheduling
@SpringBootApplication
@EnableScheduling
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

@Component
@Slf4j
public class BasicScheduledTasks {

    // Fixed rate: runs every 5 seconds, regardless of how long it takes
    // Multiple executions can overlap if task takes longer than interval
    @Scheduled(fixedRate = 5000)
    public void fixedRateTask() {
        log.info("Fixed rate task: {}", LocalDateTime.now());
    }

    // Fixed rate with initial delay: wait 10s before first execution
    @Scheduled(fixedRate = 5000, initialDelay = 10000)
    public void fixedRateWithDelay() {
        log.info("Fixed rate with delay: {}", LocalDateTime.now());
    }

    // Fixed delay: waits 5 seconds AFTER task completion before running again
    // Prevents overlapping executions
    @Scheduled(fixedDelay = 5000)
    public void fixedDelayTask() {
        log.info("Fixed delay task started: {}", LocalDateTime.now());
        // Do some work (may take variable time)
        processMessages();
        log.info("Fixed delay task finished: {}", LocalDateTime.now());
        // Next execution starts 5 seconds after this line
    }

    // Using properties for configurable timing
    @Scheduled(fixedRateString = "${app.tasks.polling-interval:5000}")
    public void configurableTask() {
        log.info("Configurable task: {}", LocalDateTime.now());
    }

    private void processMessages() {
        // Simulated work
    }
}
```

---

## 2. Cron Expressions in Detail

```
Cron format: second minute hour day-of-month month day-of-week

Field         Values
------        ------
second        0-59
minute        0-59
hour          0-23
day-of-month  1-31
month         1-12 or JAN-DEC
day-of-week   0-7 (0 and 7 = Sunday) or SUN-SAT

Special chars:
  *  = any value
  ?  = no specific value (day-of-month or day-of-week)
  -  = range (1-5 = 1,2,3,4,5)
  ,  = list (MON,WED,FRI)
  /  = step (0/15 = 0,15,30,45)
  L  = last (L in day-of-week = last Saturday)
  W  = weekday (15W = nearest weekday to 15th)
  #  = nth occurrence (2#3 = 3rd Tuesday)
```

```java
@Component
@Slf4j
public class CronScheduledTasks {

    // Every second
    @Scheduled(cron = "* * * * * *")
    public void everySecond() {}

    // Every 15 seconds
    @Scheduled(cron = "0/15 * * * * *")
    public void every15Seconds() {}

    // Every minute
    @Scheduled(cron = "0 * * * * *")
    public void everyMinute() {}

    // Every 5 minutes
    @Scheduled(cron = "0 0/5 * * * *")
    public void every5Minutes() {}

    // Every hour at minute 0
    @Scheduled(cron = "0 0 * * * *")
    public void hourly() {}

    // Every day at 2:30 AM
    @Scheduled(cron = "0 30 2 * * *")
    public void dailyAt2_30() {
        log.info("Daily cleanup at 2:30 AM");
        performCleanup();
    }

    // Every Monday at 8:00 AM
    @Scheduled(cron = "0 0 8 * * MON")
    public void weeklyMonday() {
        log.info("Weekly Monday report");
    }

    // First day of every month at midnight
    @Scheduled(cron = "0 0 0 1 * *")
    public void firstOfMonth() {
        log.info("Monthly billing run");
    }

    // Weekdays only at 9 AM
    @Scheduled(cron = "0 0 9 * * MON-FRI")
    public void weekdaysAt9() {
        log.info("Weekday morning task");
    }

    // Last day of month at 11:59 PM
    @Scheduled(cron = "0 59 23 L * *")
    public void lastDayOfMonth() {
        log.info("Month-end processing");
    }

    // Configurable cron via properties
    @Scheduled(cron = "${app.tasks.report-cron:0 0 6 * * *}")
    public void configurableReport() {
        log.info("Configurable report task");
    }

    // Timezone-aware scheduling
    @Scheduled(cron = "0 0 9 * * *", zone = "America/New_York")
    public void newYorkNineAm() {
        log.info("9 AM New York time");
    }

    private void performCleanup() {}
}
```

---

## 3. TaskScheduler and @EnableScheduling

```java
// Custom TaskScheduler configuration
@Configuration
@EnableScheduling
public class SchedulingConfig implements SchedulingConfigurer {

    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);                    // 10 concurrent scheduled tasks
        scheduler.setThreadNamePrefix("scheduled-");
        scheduler.setAwaitTerminationSeconds(30);     // Wait up to 30s on shutdown
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.setErrorHandler(t ->
            LoggerFactory.getLogger(SchedulingConfig.class)
                .error("Scheduled task error", t));
        scheduler.initialize();
        return scheduler;
    }

    // Configure task registrar (for programmatic scheduling)
    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        registrar.setTaskScheduler(taskScheduler());
    }
}
```

---

## 4. Dynamic Scheduling with ScheduledTaskRegistrar

```java
// Dynamic scheduling: change schedule at runtime
@Component
@Slf4j
public class DynamicSchedulingService implements SchedulingConfigurer {

    private final SchedulingConfigRepository configRepository;
    private ScheduledTask scheduledTask;
    private ScheduledTaskRegistrar taskRegistrar;

    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        this.taskRegistrar = registrar;

        // Add a trigger-based task that reads schedule from DB
        registrar.addTriggerTask(
            this::executeTask,
            triggerContext -> {
                // Read current schedule from DB
                String cronExpression = configRepository
                    .findByTaskName("data-sync")
                    .map(SchedulingConfig::getCronExpression)
                    .orElse("0 0/15 * * * *");  // Default: every 15 minutes

                CronTrigger trigger = new CronTrigger(cronExpression);
                return trigger.nextExecution(triggerContext);
            }
        );
    }

    public void executeTask() {
        log.info("Dynamic task executing at {}", LocalDateTime.now());
        performDataSync();
    }

    // Update schedule at runtime (e.g., via admin API)
    public void updateSchedule(String taskName, String newCronExpression) {
        // Validate cron expression
        try {
            new CronExpression(newCronExpression);
        } catch (IllegalArgumentException e) {
            throw new InvalidCronExpressionException("Invalid cron: " + newCronExpression);
        }

        // Save to DB - the trigger will pick it up on next execution
        configRepository.findByTaskName(taskName)
            .ifPresent(config -> {
                config.setCronExpression(newCronExpression);
                configRepository.save(config);
            });

        log.info("Updated schedule for '{}' to '{}'", taskName, newCronExpression);
    }

    private void performDataSync() {}
}

// Admin API to control tasks
@RestController
@RequestMapping("/api/admin/tasks")
@PreAuthorize("hasRole('ADMIN')")
public class TaskAdminController {

    private final DynamicSchedulingService schedulingService;
    private final TaskScheduler taskScheduler;

    @PutMapping("/{taskName}/schedule")
    public ResponseEntity<Void> updateSchedule(
            @PathVariable String taskName,
            @RequestBody UpdateScheduleRequest request) {
        schedulingService.updateSchedule(taskName, request.getCronExpression());
        return ResponseEntity.ok().build();
    }

    @PostMapping("/{taskName}/run-now")
    public ResponseEntity<Void> runNow(@PathVariable String taskName) {
        // Run task immediately (outside schedule)
        taskScheduler.schedule(
            () -> schedulingService.executeTask(),
            Instant.now()
        );
        return ResponseEntity.accepted().build();
    }
}
```

---

## 5. Distributed Scheduling with ShedLock

Without ShedLock, every pod in a cluster runs the same scheduled task simultaneously.

```sql
-- ShedLock table (must create manually)
CREATE TABLE shedlock (
    name        VARCHAR(64)  NOT NULL,
    lock_until  TIMESTAMP    NOT NULL,
    locked_at   TIMESTAMP    NOT NULL,
    locked_by   VARCHAR(255) NOT NULL,
    PRIMARY KEY (name)
);
```

```java
// ShedLock configuration
@Configuration
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class ShedLockConfig {

    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(new JdbcTemplate(dataSource))
                .usingDbTime()  // Use DB time for consistency across nodes
                .build()
        );
    }
}

// Scheduled tasks with distributed locks
@Component
@Slf4j
public class DistributedScheduledTasks {

    private final ReportService reportService;
    private final DataCleanupService cleanupService;
    private final EmailNotificationService emailService;

    // lockAtMostFor: maximum time lock is held (safety net for crashed nodes)
    // lockAtLeastFor: minimum time lock is held (prevents immediate re-run)
    @Scheduled(cron = "0 0 2 * * *")  // 2 AM daily
    @SchedulerLock(
        name = "daily-cleanup",
        lockAtMostFor = "30m",
        lockAtLeastFor = "10m"
    )
    public void dailyCleanup() {
        log.info("Starting daily cleanup (node: {})", getNodeId());

        int deleted = cleanupService.deleteExpiredSessions();
        int archived = cleanupService.archiveOldOrders(30);
        int purged = cleanupService.purgeExpiredTokens();

        log.info("Cleanup done: deleted={}, archived={}, purged={}",
            deleted, archived, purged);
    }

    @Scheduled(cron = "0 0 6 1 * *")  // 6 AM first of month
    @SchedulerLock(
        name = "monthly-report",
        lockAtMostFor = "2h",
        lockAtLeastFor = "30m"
    )
    public void monthlyReport() {
        log.info("Starting monthly report");
        YearMonth lastMonth = YearMonth.now().minusMonths(1);

        try {
            byte[] report = reportService.generateMonthlyReport(lastMonth);
            emailService.sendReportToManagement(report, lastMonth);
            log.info("Monthly report sent for {}", lastMonth);
        } catch (Exception e) {
            log.error("Monthly report failed for {}", lastMonth, e);
            emailService.sendAlert("Monthly report failed: " + e.getMessage());
            throw e;  // Re-throw to mark as failed
        }
    }

    @Scheduled(fixedDelay = 30000)  // Every 30 seconds
    @SchedulerLock(
        name = "outbox-relay",
        lockAtMostFor = "25s",  // Less than fixedDelay to prevent lock stacking
        lockAtLeastFor = "1s"
    )
    public void processOutboxEvents() {
        // Only one node processes outbox at a time
        outboxRelayService.processEvents();
    }

    private String getNodeId() {
        return System.getenv().getOrDefault("HOSTNAME", "unknown");
    }
}

// ShedLock with custom lock annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Scheduled(cron = "${app.tasks.${taskName}.cron}")
@SchedulerLock(name = "${taskName}", lockAtMostFor = "15m")
public @interface ScheduledJob {
    String taskName();
}
```

---

## 6. Spring Batch with Scheduling

```java
// Batch job for processing large datasets
@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Bean
    public Job orderExportJob(JobRepository jobRepository,
            Step exportStep) {
        return new JobBuilder("order-export-job", jobRepository)
            .incrementer(new RunIdIncrementer())
            .start(exportStep)
            .listener(new JobExecutionListener() {
                @Override
                public void afterJob(JobExecution jobExecution) {
                    if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
                        log.info("Order export completed: {} records",
                            jobExecution.getStepExecutions().iterator()
                                .next().getWriteCount());
                    } else {
                        log.error("Order export failed: {}", jobExecution.getStatus());
                    }
                }
            })
            .build();
    }

    @Bean
    public Step exportStep(JobRepository jobRepository,
            PlatformTransactionManager transactionManager,
            ItemReader<Order> orderReader,
            ItemProcessor<Order, OrderExportDto> orderProcessor,
            ItemWriter<OrderExportDto> csvWriter) {

        return new StepBuilder("export-step", jobRepository)
            .<Order, OrderExportDto>chunk(500, transactionManager)
            .reader(orderReader)
            .processor(orderProcessor)
            .writer(csvWriter)
            .faultTolerant()
            .skipLimit(10)
            .skip(DataIntegrityViolationException.class)
            .retryLimit(3)
            .retry(TransientDataAccessException.class)
            .build();
    }

    @Bean
    @StepScope
    public JdbcCursorItemReader<Order> orderReader(DataSource dataSource,
            @Value("#{jobParameters['startDate']}") String startDate,
            @Value("#{jobParameters['endDate']}") String endDate) {

        return new JdbcCursorItemReaderBuilder<Order>()
            .name("orderReader")
            .dataSource(dataSource)
            .sql("""
                SELECT o.*, c.email AS customer_email
                FROM orders o
                JOIN customers c ON o.customer_id = c.id
                WHERE o.created_at BETWEEN ? AND ?
                ORDER BY o.created_at
                """)
            .preparedStatementSetter(ps -> {
                ps.setString(1, startDate);
                ps.setString(2, endDate);
            })
            .rowMapper(new OrderRowMapper())
            .build();
    }

    @Bean
    public ItemProcessor<Order, OrderExportDto> orderProcessor() {
        return order -> {
            if (order.getStatus() == OrderStatus.CANCELLED) {
                return null;  // Filter out cancelled orders
            }
            return OrderExportDto.from(order);
        };
    }

    @Bean
    @StepScope
    public FlatFileItemWriter<OrderExportDto> csvWriter(
            @Value("#{jobParameters['outputFile']}") String outputFile) {

        return new FlatFileItemWriterBuilder<OrderExportDto>()
            .name("csvWriter")
            .resource(new FileSystemResource(outputFile))
            .delimited()
            .delimiter(",")
            .names("orderId", "customerEmail", "amount", "status", "createdAt")
            .headerCallback(writer -> writer.write(
                "Order ID,Customer Email,Amount,Status,Created At"))
            .build();
    }
}

// Schedule batch job with ShedLock
@Component
@Slf4j
public class BatchJobScheduler {

    private final JobLauncher jobLauncher;
    private final Job orderExportJob;

    @Scheduled(cron = "0 0 1 * * *")  // 1 AM daily
    @SchedulerLock(name = "order-export-batch", lockAtMostFor = "2h")
    public void runOrderExport() {
        try {
            LocalDate yesterday = LocalDate.now().minusDays(1);
            String outputFile = "/exports/orders-" + yesterday + ".csv";

            JobParameters params = new JobParametersBuilder()
                .addString("startDate", yesterday.atStartOfDay().toString())
                .addString("endDate", yesterday.plusDays(1).atStartOfDay().toString())
                .addString("outputFile", outputFile)
                .addLong("timestamp", System.currentTimeMillis())  // Makes params unique
                .toJobParameters();

            JobExecution execution = jobLauncher.run(orderExportJob, params);

            log.info("Batch job completed: status={}, records={}",
                execution.getStatus(),
                execution.getStepExecutions().stream()
                    .mapToLong(StepExecution::getWriteCount)
                    .sum());

        } catch (Exception e) {
            log.error("Batch job failed", e);
        }
    }
}
```

---

## 7. Quartz Scheduler Integration

```java
// Quartz configuration (persisted jobs survive restarts)
@Configuration
public class QuartzConfig {

    @Bean
    public SchedulerFactoryBean schedulerFactory(DataSource dataSource,
            ApplicationContext context) {

        SchedulerFactoryBean factory = new SchedulerFactoryBean();
        factory.setDataSource(dataSource);  // Persistent job store
        factory.setApplicationContextSchedulerContextKey("applicationContext");
        factory.setWaitForJobsToCompleteOnShutdown(true);
        factory.setOverwriteExistingJobs(false);

        Properties props = new Properties();
        props.setProperty("org.quartz.jobStore.class",
            "org.springframework.scheduling.quartz.LocalDataSourceJobStore");
        props.setProperty("org.quartz.jobStore.driverDelegateClass",
            "org.quartz.impl.jdbcjobstore.PostgreSQLDelegate");
        props.setProperty("org.quartz.jobStore.isClustered", "true");  // Clustered!
        props.setProperty("org.quartz.jobStore.clusterCheckinInterval", "20000");
        props.setProperty("org.quartz.scheduler.instanceId", "AUTO");
        props.setProperty("org.quartz.threadPool.threadCount", "10");

        factory.setQuartzProperties(props);
        return factory;
    }
}

// Quartz job implementation
@Component
@Slf4j
public class ReportGenerationJob implements Job {

    @Autowired
    private ReportService reportService;

    @Autowired
    private EmailService emailService;

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        JobDataMap data = context.getMergedJobDataMap();
        String reportType = data.getString("reportType");
        String recipientEmail = data.getString("recipientEmail");

        log.info("Generating {} report for {}", reportType, recipientEmail);

        try {
            byte[] report = reportService.generateReport(reportType);
            emailService.sendReport(recipientEmail, reportType, report);
            log.info("Report sent successfully");
        } catch (Exception e) {
            log.error("Report generation failed", e);
            throw new JobExecutionException(e, true);  // true = refire job
        }
    }
}

// Job scheduler service
@Service
@Slf4j
public class QuartzJobScheduler {

    private final Scheduler scheduler;

    public void scheduleRecurringReport(ScheduleReportRequest request) throws SchedulerException {
        String jobKey = "report-" + request.getRecipientId();
        String groupKey = "reports";

        // Check if job already exists
        if (scheduler.checkExists(new JobKey(jobKey, groupKey))) {
            scheduler.deleteJob(new JobKey(jobKey, groupKey));
        }

        JobDetail jobDetail = JobBuilder.newJob(ReportGenerationJob.class)
            .withIdentity(jobKey, groupKey)
            .withDescription("Recurring report for " + request.getRecipientEmail())
            .usingJobData("reportType", request.getReportType())
            .usingJobData("recipientEmail", request.getRecipientEmail())
            .storeDurably()  // Keep job even if no triggers
            .build();

        CronTrigger trigger = TriggerBuilder.newTrigger()
            .withIdentity("trigger-" + jobKey, groupKey)
            .withSchedule(CronScheduleBuilder.cronSchedule(request.getCronExpression())
                .inTimeZone(TimeZone.getTimeZone(request.getTimeZone()))
                .withMisfireHandlingInstructionFireAndProceed())
            .build();

        scheduler.scheduleJob(jobDetail, trigger);
        log.info("Scheduled report job '{}' with cron '{}'",
            jobKey, request.getCronExpression());
    }

    public void scheduleOneTimeJob(String jobKey, LocalDateTime runAt,
            Map<String, Object> data) throws SchedulerException {

        JobDataMap jobData = new JobDataMap(data);

        JobDetail job = JobBuilder.newJob(ReportGenerationJob.class)
            .withIdentity(jobKey, "one-time")
            .usingJobData(jobData)
            .build();

        Trigger trigger = TriggerBuilder.newTrigger()
            .withIdentity("trigger-" + jobKey, "one-time")
            .startAt(Date.from(runAt.atZone(ZoneId.systemDefault()).toInstant()))
            .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                .withMisfireHandlingInstructionFireNow())
            .build();

        scheduler.scheduleJob(job, trigger);
    }

    public void cancelJob(String jobKey, String groupKey) throws SchedulerException {
        scheduler.deleteJob(new JobKey(jobKey, groupKey));
        log.info("Cancelled job: {}/{}", groupKey, jobKey);
    }

    public List<JobInfo> listJobs(String groupKey) throws SchedulerException {
        GroupMatcher<JobKey> matcher = GroupMatcher.groupEquals(groupKey);

        return scheduler.getJobKeys(matcher).stream()
            .map(key -> {
                try {
                    JobDetail detail = scheduler.getJobDetail(key);
                    List<? extends Trigger> triggers = scheduler.getTriggersOfJob(key);
                    Date nextFire = triggers.isEmpty() ? null :
                        triggers.get(0).getNextFireTime();

                    return JobInfo.builder()
                        .name(key.getName())
                        .group(key.getGroup())
                        .description(detail.getDescription())
                        .nextFireTime(nextFire != null ?
                            nextFire.toInstant().atZone(ZoneId.systemDefault())
                                .toLocalDateTime() : null)
                        .build();
                } catch (SchedulerException e) {
                    return null;
                }
            })
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
}
```

---

## 8. Job Monitoring and Alerting

```java
// Job execution tracker
@Component
@Slf4j
public class ScheduledJobMonitor {

    private final JobExecutionRepository jobExecutionRepository;
    private final AlertService alertService;
    private final MeterRegistry meterRegistry;

    // Decorator for monitoring any scheduled task
    public <T> T executeWithMonitoring(String jobName, Supplier<T> task) {
        JobExecution execution = JobExecution.builder()
            .jobName(jobName)
            .startedAt(LocalDateTime.now())
            .status(JobExecutionStatus.RUNNING)
            .nodeId(getNodeId())
            .build();
        execution = jobExecutionRepository.save(execution);

        Timer.Sample timer = Timer.start(meterRegistry);
        try {
            T result = task.get();

            execution.setStatus(JobExecutionStatus.SUCCESS);
            execution.setFinishedAt(LocalDateTime.now());
            jobExecutionRepository.save(execution);

            timer.stop(meterRegistry.timer("job.duration",
                "name", jobName, "status", "success"));

            meterRegistry.counter("job.executions",
                "name", jobName, "status", "success").increment();

            return result;

        } catch (Exception e) {
            execution.setStatus(JobExecutionStatus.FAILED);
            execution.setFinishedAt(LocalDateTime.now());
            execution.setErrorMessage(e.getMessage());
            jobExecutionRepository.save(execution);

            timer.stop(meterRegistry.timer("job.duration",
                "name", jobName, "status", "failure"));

            meterRegistry.counter("job.executions",
                "name", jobName, "status", "failure").increment();

            alertService.sendAlert(Alert.builder()
                .severity(AlertSeverity.ERROR)
                .title("Job Failed: " + jobName)
                .message("Job '" + jobName + "' failed: " + e.getMessage())
                .build());

            throw e;
        }
    }

    // Check for stuck jobs (running longer than expected)
    @Scheduled(cron = "0 0/5 * * * *")  // Every 5 minutes
    public void checkStuckJobs() {
        LocalDateTime stuckThreshold = LocalDateTime.now().minusHours(1);

        List<JobExecution> stuckJobs = jobExecutionRepository
            .findByStatusAndStartedAtBefore(JobExecutionStatus.RUNNING, stuckThreshold);

        for (JobExecution stuck : stuckJobs) {
            log.error("Stuck job detected: {} started at {}",
                stuck.getJobName(), stuck.getStartedAt());

            alertService.sendAlert(Alert.builder()
                .severity(AlertSeverity.CRITICAL)
                .title("Stuck Job: " + stuck.getJobName())
                .message("Job has been running for over 1 hour")
                .build());
        }
    }

    private String getNodeId() {
        return System.getenv().getOrDefault("HOSTNAME", "localhost");
    }
}

// Monitored task wrapper
@Component
@Slf4j
public class MonitoredScheduledTasks {

    private final ScheduledJobMonitor monitor;
    private final CleanupService cleanupService;
    private final ReportService reportService;

    @Scheduled(cron = "0 0 3 * * *")
    @SchedulerLock(name = "cleanup-expired-data", lockAtMostFor = "45m")
    public void cleanupExpiredData() {
        monitor.executeWithMonitoring("cleanup-expired-data", () -> {
            int deleted = cleanupService.deleteExpiredData();
            log.info("Cleaned up {} expired records", deleted);
            return deleted;
        });
    }

    @Scheduled(cron = "0 30 5 * * MON-FRI")
    @SchedulerLock(name = "daily-report", lockAtMostFor = "1h")
    public void generateDailyReport() {
        monitor.executeWithMonitoring("daily-report", () -> {
            reportService.generateAndSendDailyReport();
            return null;
        });
    }
}
```

---

## 9. Distributed Lock Patterns

```java
// Redis-based distributed lock (without ShedLock)
@Component
@Slf4j
public class RedisDistributedLock {

    private final RedisTemplate<String, String> redisTemplate;
    private static final String LOCK_PREFIX = "lock:";

    public boolean tryLock(String lockName, Duration duration) {
        String key = LOCK_PREFIX + lockName;
        String value = UUID.randomUUID().toString();

        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(key, value, duration);

        return Boolean.TRUE.equals(acquired);
    }

    public void unlock(String lockName) {
        String key = LOCK_PREFIX + lockName;
        redisTemplate.delete(key);
    }

    public <T> T executeWithLock(String lockName, Duration lockDuration,
            Supplier<T> task) {

        if (!tryLock(lockName, lockDuration)) {
            log.info("Could not acquire lock '{}', skipping", lockName);
            return null;
        }

        try {
            return task.get();
        } finally {
            unlock(lockName);
        }
    }
}

// Distributed lock with owner tracking (prevents other instances from stealing)
@Component
@Slf4j
public class OwnerAwareDistributedLock {

    private final StringRedisTemplate redisTemplate;
    private final String instanceId;

    public OwnerAwareDistributedLock(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
        this.instanceId = UUID.randomUUID().toString();
    }

    public boolean acquireLock(String lockName, Duration duration) {
        String key = "dlock:" + lockName;
        Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(key, instanceId, duration);
        return Boolean.TRUE.equals(result);
    }

    public boolean releaseLock(String lockName) {
        String key = "dlock:" + lockName;
        String currentOwner = redisTemplate.opsForValue().get(key);

        if (instanceId.equals(currentOwner)) {
            redisTemplate.delete(key);
            return true;
        }

        log.warn("Cannot release lock '{}': owned by {}", lockName, currentOwner);
        return false;
    }

    public boolean extendLock(String lockName, Duration extension) {
        String key = "dlock:" + lockName;
        String currentOwner = redisTemplate.opsForValue().get(key);

        if (instanceId.equals(currentOwner)) {
            redisTemplate.expire(key, extension);
            return true;
        }

        return false;
    }
}
```

---

## 10. Real Example: Report Generation and Cleanup Jobs

```java
// Complete example: monthly report + daily cleanup with ShedLock

@Entity
@Table(name = "job_executions")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class JobExecution {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String jobName;
    private String nodeId;

    @Enumerated(EnumType.STRING)
    private JobExecutionStatus status;

    private LocalDateTime startedAt;
    private LocalDateTime finishedAt;
    private Long durationMs;

    @Column(length = 2000)
    private String errorMessage;

    private Long recordsProcessed;
}

public enum JobExecutionStatus { RUNNING, SUCCESS, FAILED, SKIPPED }

// Production-grade scheduled tasks
@Component
@Slf4j
public class ProductionScheduledJobs {

    private final ReportGenerationService reportService;
    private final DataCleanupService cleanupService;
    private final StorageService storageService;
    private final EmailService emailService;
    private final ScheduledJobMonitor monitor;

    // ======= CLEANUP JOBS =======

    @Scheduled(cron = "0 0 2 * * *")  // 2 AM daily
    @SchedulerLock(name = "cleanup-expired-sessions", lockAtMostFor = "30m", lockAtLeastFor = "1m")
    public void cleanupExpiredSessions() {
        monitor.executeWithMonitoring("cleanup-expired-sessions", () -> {
            log.info("Cleaning expired sessions");
            int count = cleanupService.deleteExpiredSessions(
                LocalDateTime.now().minusDays(30));
            log.info("Deleted {} expired sessions", count);
            return count;
        });
    }

    @Scheduled(cron = "0 30 2 * * *")  // 2:30 AM daily
    @SchedulerLock(name = "cleanup-temp-files", lockAtMostFor = "1h", lockAtLeastFor = "5m")
    public void cleanupTemporaryFiles() {
        monitor.executeWithMonitoring("cleanup-temp-files", () -> {
            log.info("Cleaning temporary files");
            List<String> deletedFiles = storageService.deleteTemporaryFilesOlderThan(
                Duration.ofDays(7));
            log.info("Deleted {} temporary files", deletedFiles.size());
            return deletedFiles.size();
        });
    }

    @Scheduled(cron = "0 0 3 * * 0")  // 3 AM every Sunday
    @SchedulerLock(name = "cleanup-audit-logs", lockAtMostFor = "2h", lockAtLeastFor = "10m")
    public void archiveOldAuditLogs() {
        monitor.executeWithMonitoring("archive-audit-logs", () -> {
            LocalDate archiveDate = LocalDate.now().minusMonths(6);
            int archived = cleanupService.archiveAuditLogs(archiveDate);
            log.info("Archived {} audit log entries", archived);
            return archived;
        });
    }

    // ======= REPORT GENERATION JOBS =======

    @Scheduled(cron = "0 0 6 * * MON-FRI")  // 6 AM weekdays
    @SchedulerLock(name = "daily-sales-report", lockAtMostFor = "1h", lockAtLeastFor = "5m")
    public void generateDailySalesReport() {
        monitor.executeWithMonitoring("daily-sales-report", () -> {
            LocalDate yesterday = LocalDate.now().minusDays(1);

            log.info("Generating daily sales report for {}", yesterday);
            ReportResult result = reportService.generateDailySalesReport(yesterday);

            // Upload to S3
            String s3Key = "reports/daily/sales-" + yesterday + ".xlsx";
            storageService.uploadReport(s3Key, result.getData());

            // Email to stakeholders
            emailService.sendDailyReport(result, s3Key);

            log.info("Daily sales report ready: {} records", result.getRecordCount());
            return result.getRecordCount();
        });
    }

    @Scheduled(cron = "0 0 7 1 * *")  // 7 AM first of month
    @SchedulerLock(name = "monthly-invoice-report", lockAtMostFor = "3h", lockAtLeastFor = "30m")
    public void generateMonthlyInvoiceReport() {
        monitor.executeWithMonitoring("monthly-invoice-report", () -> {
            YearMonth lastMonth = YearMonth.now().minusMonths(1);

            log.info("Generating monthly invoice report for {}", lastMonth);

            // Process in pages to handle large datasets
            int totalProcessed = 0;
            int page = 0;
            int pageSize = 1000;
            List<ReportPage> pages = new ArrayList<>();

            while (true) {
                ReportPage reportPage = reportService.generateInvoicePage(
                    lastMonth, page, pageSize);

                if (reportPage.isEmpty()) break;

                pages.add(reportPage);
                totalProcessed += reportPage.getRecordCount();
                page++;

                log.debug("Processed page {}, total: {}", page, totalProcessed);
            }

            // Merge pages and upload
            byte[] finalReport = reportService.mergePages(pages);
            String s3Key = "reports/monthly/invoices-" + lastMonth + ".pdf";
            storageService.uploadReport(s3Key, finalReport);

            emailService.sendMonthlyInvoiceReport(s3Key, lastMonth, totalProcessed);

            log.info("Monthly invoice report done: {} invoices for {}", totalProcessed, lastMonth);
            return totalProcessed;
        });
    }

    // ======= SYNC JOBS =======

    @Scheduled(fixedDelay = 60000)  // 1 minute after last completion
    @SchedulerLock(name = "sync-product-catalog", lockAtMostFor = "50s", lockAtLeastFor = "5s")
    public void syncProductCatalog() {
        monitor.executeWithMonitoring("sync-product-catalog", () -> {
            int synced = productSyncService.syncChangedProducts();
            if (synced > 0) {
                log.info("Synced {} products from external catalog", synced);
            }
            return synced;
        });
    }
}

// Job status API endpoint
@RestController
@RequestMapping("/api/admin/jobs")
@PreAuthorize("hasRole('ADMIN')")
public class JobStatusController {

    private final JobExecutionRepository jobExecutionRepository;
    private final QuartzJobScheduler quartzScheduler;

    @GetMapping
    public ResponseEntity<List<JobStatusDto>> getJobStatus() {
        // Get last 24h job executions
        LocalDateTime since = LocalDateTime.now().minusHours(24);
        List<JobExecution> executions = jobExecutionRepository
            .findByStartedAtAfterOrderByStartedAtDesc(since);

        Map<String, List<JobExecution>> byJob = executions.stream()
            .collect(Collectors.groupingBy(JobExecution::getJobName));

        List<JobStatusDto> status = byJob.entrySet().stream()
            .map(e -> {
                JobExecution latest = e.getValue().get(0);
                long successCount = e.getValue().stream()
                    .filter(j -> j.getStatus() == JobExecutionStatus.SUCCESS).count();
                long failureCount = e.getValue().stream()
                    .filter(j -> j.getStatus() == JobExecutionStatus.FAILED).count();

                return JobStatusDto.builder()
                    .jobName(e.getKey())
                    .lastStatus(latest.getStatus())
                    .lastRunAt(latest.getStartedAt())
                    .lastDurationMs(latest.getDurationMs())
                    .successCount24h((int) successCount)
                    .failureCount24h((int) failureCount)
                    .build();
            })
            .collect(Collectors.toList());

        return ResponseEntity.ok(status);
    }

    @PostMapping("/{jobName}/trigger")
    public ResponseEntity<String> triggerJob(@PathVariable String jobName) {
        // Manually trigger a job
        applicationEventPublisher.publishEvent(new ManualJobTriggerEvent(jobName));
        return ResponseEntity.accepted().body("Job triggered: " + jobName);
    }
}
```

### Configuration

```yaml
# application.yml for scheduled tasks
spring:
  quartz:
    job-store-type: jdbc       # Persistent storage
    jdbc:
      initialize-schema: always
    properties:
      org.quartz.jobStore.isClustered: true
      org.quartz.scheduler.instanceId: AUTO
      org.quartz.threadPool.threadCount: 10
  batch:
    job:
      enabled: false    # Don't run batch jobs on startup
    jdbc:
      initialize-schema: always

app:
  tasks:
    enabled: true
    report-cron: "0 0 6 * * MON-FRI"
    cleanup-cron: "0 0 2 * * *"
    sync-interval: 60000

# ShedLock table must be created manually:
# CREATE TABLE shedlock (
#   name VARCHAR(64) NOT NULL,
#   lock_until TIMESTAMP NOT NULL,
#   locked_at TIMESTAMP NOT NULL,
#   locked_by VARCHAR(255) NOT NULL,
#   PRIMARY KEY (name)
# );

logging:
  level:
    net.javacrumbs.shedlock: DEBUG
    org.springframework.scheduling: INFO
```

---

## Summary

| Tool | Use Case | Key Feature |
|------|---------|------------|
| `@Scheduled` | Simple recurring tasks | Easy, single-node |
| ShedLock | Distributed single-execution | Locks via DB/Redis |
| Spring Batch | Large dataset processing | Chunk-oriented, restartable |
| Quartz | Complex schedules, persistence | Survives restarts, clustered |
| Dynamic Scheduling | Runtime-configurable | DB-stored cron expressions |
| Distributed Lock | Custom coordination | Redis SET NX EX |

| Cron Example | Schedule |
|-------------|---------|
| `0 0 2 * * *` | 2 AM daily |
| `0 0 6 * * MON-FRI` | 6 AM weekdays |
| `0 0 1 * * 0` | 1 AM every Sunday |
| `0 0 0 1 * *` | Midnight first of month |
| `0 0/15 * * * *` | Every 15 minutes |
| `0 0 * * * *` | Every hour |

## Next Part Preview

**Part 064: File Handling and Storage** — Multipart upload, chunked upload, S3 integration, image processing with Thumbnailator, PDF generation, Excel with Apache POI, and a complete document management system.
