# Part 59: Spring Batch Processing
## ขั้นตอนที่ 4031-4100: ETL, Batch Jobs, Large Data Processing

---

## 59.1 Spring Batch Concepts

```
Spring Batch Architecture:

Job
 └── Step (one or more)
      ├── ItemReader    = read data (DB, CSV, JSON, API)
      ├── ItemProcessor = transform/validate each item
      └── ItemWriter    = write results (DB, CSV, Kafka, Email)

Chunk-oriented Processing:
  Read N items → process each → write batch
  
  Example (chunk size = 100):
    1. Read 100 orders from DB
    2. Process each order (calculate discount, validate)
    3. Write 100 processed orders to DB
    4. Commit transaction
    5. Repeat until no more items

Job execution:
  JobInstance = one execution attempt (job name + parameters)
  JobExecution = one run of a JobInstance (can be multiple if restart)
  StepExecution = one run of a Step within a JobExecution

Features:
  ✓ Restart: failed jobs resume from last checkpoint
  ✓ Skip: skip bad records, continue processing
  ✓ Retry: retry failed items with backoff
  ✓ Partitioning: parallel processing
  ✓ Progress: track read/write/skip counts
```

---

## 59.2 Basic Job Configuration

```java
import org.springframework.batch.core.*;
import org.springframework.batch.core.job.builder.*;
import org.springframework.batch.core.step.builder.*;
import org.springframework.batch.item.*;
import org.springframework.batch.item.file.*;
import org.springframework.batch.item.database.*;
import org.springframework.context.annotation.*;

@Configuration
@EnableBatchProcessing
class OrderProcessingBatchConfig {
    
    // ====== Reader: read from CSV ======
    @Bean
    FlatFileItemReader<OrderCsvRow> orderCsvReader() {
        return new FlatFileItemReaderBuilder<OrderCsvRow>()
            .name("orderCsvReader")
            .resource(new org.springframework.core.io.FileSystemResource("/data/orders.csv"))
            .delimited()
            .names("orderId", "userId", "productId", "quantity", "price", "date")
            .targetType(OrderCsvRow.class)
            .linesToSkip(1)  // skip header
            .strict(false)   // don't fail on missing file
            .build();
    }
    
    // ====== Reader: read from DB ======
    @Bean
    JdbcCursorItemReader<Order> orderDbReader(javax.sql.DataSource dataSource) {
        return new JdbcCursorItemReaderBuilder<Order>()
            .name("orderDbReader")
            .dataSource(dataSource)
            .sql("SELECT * FROM orders WHERE status = 'PENDING' ORDER BY created_at")
            .rowMapper(new org.springframework.jdbc.core.BeanPropertyRowMapper<>(Order.class))
            .fetchSize(1000)
            .build();
    }
    
    // ====== Paging reader (better for large data) ======
    @Bean
    JdbcPagingItemReader<Order> orderPagingReader(javax.sql.DataSource dataSource) {
        return new JdbcPagingItemReaderBuilder<Order>()
            .name("orderPagingReader")
            .dataSource(dataSource)
            .selectClause("SELECT id, user_id, status, total, created_at")
            .fromClause("FROM orders")
            .whereClause("WHERE status = 'PENDING'")
            .sortKeys(java.util.Map.of("created_at", Order.class))
            .pageSize(100)
            .rowMapper(new org.springframework.jdbc.core.BeanPropertyRowMapper<>(Order.class))
            .build();
    }
    
    // ====== Processor ======
    @Bean
    ItemProcessor<Order, ProcessedOrder> orderProcessor() {
        return order -> {
            // Apply business rules
            double discount = order.getTotal() >= 10000 ? 0.1 : 0.0;
            double finalTotal = order.getTotal() * (1 - discount);
            
            return new ProcessedOrder(
                order.getId(),
                order.getUserId(),
                finalTotal,
                discount,
                java.time.Instant.now()
            );
        };
    }
    
    // ====== Writer: write to DB ======
    @Bean
    JdbcBatchItemWriter<ProcessedOrder> orderWriter(javax.sql.DataSource dataSource) {
        return new JdbcBatchItemWriterBuilder<ProcessedOrder>()
            .dataSource(dataSource)
            .sql("""
                INSERT INTO processed_orders(order_id, user_id, final_total, discount, processed_at)
                VALUES (:orderId, :userId, :finalTotal, :discount, :processedAt)
                ON CONFLICT (order_id) DO UPDATE SET
                  final_total = :finalTotal,
                  discount = :discount,
                  processed_at = :processedAt
                """)
            .beanMapped()
            .build();
    }
    
    // ====== Step ======
    @Bean
    Step processOrdersStep(
            JobRepository jobRepository,
            PlatformTransactionManager txManager,
            JdbcPagingItemReader<Order> reader,
            ItemProcessor<Order, ProcessedOrder> processor,
            JdbcBatchItemWriter<ProcessedOrder> writer) {
        
        return new StepBuilder("processOrdersStep", jobRepository)
            .<Order, ProcessedOrder>chunk(100, txManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .faultTolerant()
            .skipLimit(10)
            .skip(IllegalArgumentException.class)   // skip bad records
            .retryLimit(3)
            .retry(java.net.ConnectException.class) // retry on transient errors
            .listener(new StepExecutionListener() {
                @Override
                public void beforeStep(StepExecution execution) {
                    System.out.println("Starting step: " + execution.getStepName());
                }
                @Override
                public ExitStatus afterStep(StepExecution execution) {
                    System.out.printf("Step done. Read: %d, Written: %d, Skipped: %d%n",
                        execution.getReadCount(),
                        execution.getWriteCount(),
                        execution.getSkipCount());
                    return execution.getExitStatus();
                }
            })
            .build();
    }
    
    // ====== Job ======
    @Bean
    Job orderProcessingJob(JobRepository jobRepository, Step processOrdersStep) {
        return new JobBuilder("orderProcessingJob", jobRepository)
            .incrementer(new RunIdIncrementer())   // allows re-run with same params
            .flow(processOrdersStep)
            .end()
            .listener(new JobExecutionListener() {
                @Override
                public void afterJob(JobExecution execution) {
                    System.out.printf("Job %s finished with status: %s%n",
                        execution.getJobInstance().getJobName(),
                        execution.getStatus());
                }
            })
            .build();
    }
}

record OrderCsvRow(String orderId, String userId, String productId,
                   int quantity, double price, String date) {}
record ProcessedOrder(String orderId, String userId, double finalTotal,
                      double discount, java.time.Instant processedAt) {}
```

---

## 59.3 Partitioned Step (Parallel Processing)

```java
import org.springframework.batch.core.partition.*;
import org.springframework.batch.core.partition.support.*;

@Configuration
class PartitionedBatchConfig {
    
    // Partitioner: splits work into independent chunks
    @Bean
    Partitioner orderPartitioner(javax.sql.DataSource dataSource) {
        return gridSize -> {
            var jdbc = new org.springframework.jdbc.core.JdbcTemplate(dataSource);
            
            // Find min/max order ID for range partitioning
            Long minId = jdbc.queryForObject("SELECT MIN(id) FROM orders WHERE status = 'PENDING'", Long.class);
            Long maxId = jdbc.queryForObject("SELECT MAX(id) FROM orders WHERE status = 'PENDING'", Long.class);
            
            if (minId == null || maxId == null) return java.util.Map.of();
            
            long rangeSize = (maxId - minId) / gridSize + 1;
            var partitions = new java.util.LinkedHashMap<String, ExecutionContext>();
            
            for (int i = 0; i < gridSize; i++) {
                long start = minId + i * rangeSize;
                long end = Math.min(start + rangeSize - 1, maxId);
                
                var context = new ExecutionContext();
                context.putLong("minId", start);
                context.putLong("maxId", end);
                partitions.put("partition-" + i, context);
            }
            
            return partitions;
        };
    }
    
    // Range reader for each partition
    @Bean
    @StepScope
    JdbcPagingItemReader<Order> partitionedOrderReader(
            javax.sql.DataSource dataSource,
            @Value("#{stepExecutionContext['minId']}") Long minId,
            @Value("#{stepExecutionContext['maxId']}") Long maxId) {
        
        return new JdbcPagingItemReaderBuilder<Order>()
            .name("partitionedReader-" + minId)
            .dataSource(dataSource)
            .selectClause("SELECT *")
            .fromClause("FROM orders")
            .whereClause("WHERE id BETWEEN :minId AND :maxId AND status = 'PENDING'")
            .parameterValues(java.util.Map.of("minId", minId, "maxId", maxId))
            .sortKeys(java.util.Map.of("id", Order.class))
            .pageSize(500)
            .rowMapper(new org.springframework.jdbc.core.BeanPropertyRowMapper<>(Order.class))
            .build();
    }
    
    // Partitioned master step
    @Bean
    Step partitionedOrderStep(
            JobRepository jobRepository,
            Partitioner orderPartitioner,
            Step orderWorkerStep,
            org.springframework.core.task.TaskExecutor taskExecutor) {
        
        return new StepBuilder("partitionedOrderStep", jobRepository)
            .partitioner("orderWorkerStep", orderPartitioner)
            .step(orderWorkerStep)
            .gridSize(4)           // 4 partitions = 4 parallel workers
            .taskExecutor(taskExecutor)
            .build();
    }
    
    @Bean
    org.springframework.core.task.TaskExecutor batchTaskExecutor() {
        var exec = new org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor();
        exec.setCorePoolSize(4);
        exec.setMaxPoolSize(4);
        exec.setQueueCapacity(0);
        exec.setThreadNamePrefix("batch-");
        exec.initialize();
        return exec;
    }
}
```

---

## 59.4 Scheduled Batch Jobs

```java
import org.springframework.batch.core.launch.*;
import org.springframework.batch.core.*;
import org.springframework.scheduling.annotation.*;

@Service
class BatchScheduler {
    
    private final JobLauncher jobLauncher;
    private final Job orderProcessingJob;
    private final Job reportGenerationJob;
    
    BatchScheduler(JobLauncher jobLauncher, Job orderProcessingJob, Job reportGenerationJob) {
        this.jobLauncher = jobLauncher;
        this.orderProcessingJob = orderProcessingJob;
        this.reportGenerationJob = reportGenerationJob;
    }
    
    // Process orders every 5 minutes
    @Scheduled(fixedDelay = 5 * 60 * 1000)
    public void runOrderProcessing() throws Exception {
        var params = new JobParametersBuilder()
            .addLong("timestamp", System.currentTimeMillis())
            .toJobParameters();
        
        var execution = jobLauncher.run(orderProcessingJob, params);
        System.out.printf("Order processing: %s (read: %d, written: %d)%n",
            execution.getStatus(),
            execution.getStepExecutions().stream()
                .mapToLong(StepExecution::getReadCount).sum(),
            execution.getStepExecutions().stream()
                .mapToLong(StepExecution::getWriteCount).sum()
        );
    }
    
    // Generate daily report at 2 AM
    @Scheduled(cron = "0 0 2 * * *")
    public void runDailyReport() throws Exception {
        var yesterday = java.time.LocalDate.now().minusDays(1);
        var params = new JobParametersBuilder()
            .addString("reportDate", yesterday.toString())
            .toJobParameters();
        
        jobLauncher.run(reportGenerationJob, params);
    }
}

// Trigger job via REST API
@org.springframework.web.bind.annotation.RestController
@org.springframework.web.bind.annotation.RequestMapping("/api/admin/batch")
class BatchController {
    
    private final JobLauncher jobLauncher;
    private final Job orderProcessingJob;
    private final org.springframework.batch.core.explore.JobExplorer jobExplorer;
    
    BatchController(JobLauncher jobLauncher, Job orderProcessingJob,
                    org.springframework.batch.core.explore.JobExplorer jobExplorer) {
        this.jobLauncher = jobLauncher;
        this.orderProcessingJob = orderProcessingJob;
        this.jobExplorer = jobExplorer;
    }
    
    @org.springframework.web.bind.annotation.PostMapping("/orders/process")
    @org.springframework.security.access.prepost.PreAuthorize("hasRole('ADMIN')")
    public String triggerOrderProcessing() throws Exception {
        var params = new JobParametersBuilder()
            .addLong("timestamp", System.currentTimeMillis())
            .toJobParameters();
        
        var execution = jobLauncher.run(orderProcessingJob, params);
        return "Job started: " + execution.getId();
    }
    
    @org.springframework.web.bind.annotation.GetMapping("/jobs/{id}")
    public org.springframework.http.ResponseEntity<JobExecutionDTO> getJobStatus(
            @org.springframework.web.bind.annotation.PathVariable Long id) {
        
        var execution = jobExplorer.getJobExecution(id);
        if (execution == null) return org.springframework.http.ResponseEntity.notFound().build();
        
        return org.springframework.http.ResponseEntity.ok(new JobExecutionDTO(
            execution.getId(),
            execution.getStatus().name(),
            execution.getCreateTime(),
            execution.getEndTime()
        ));
    }
}

record JobExecutionDTO(Long id, String status, java.util.Date startTime, java.util.Date endTime) {}
```

---

## สรุป Part 59

```
Spring Batch Key Concepts:

Architecture:
  Job → Steps → [Reader → Processor → Writer]
  Chunk: read N → process each → write batch → commit

Readers:
  FlatFileItemReader  = CSV/fixed-width files
  JdbcCursorItemReader = DB cursor (memory efficient for sequential)
  JdbcPagingItemReader = DB paging (safe for large tables, restartable)

Processors:
  Transform, validate, filter (return null to skip item)
  Chain multiple with CompositeItemProcessor

Writers:
  JdbcBatchItemWriter = batch INSERT/UPDATE
  FlatFileItemWriter  = write to CSV
  KafkaItemWriter     = publish to Kafka

Fault Tolerance:
  skipLimit(N) + skip(ExceptionClass) = skip bad records
  retryLimit(N) + retry(ExceptionClass) = retry transient failures

Partitioning (parallel):
  Split by ID range, date range, or file
  gridSize = number of parallel workers
  Use ThreadPoolTaskExecutor for local, Kubernetes Jobs for remote

When to use Spring Batch:
  ✓ ETL: Extract → Transform → Load
  ✓ Report generation
  ✓ Data migration
  ✓ Monthly billing, statement generation
  ✓ > 100,000 records to process
  ✗ Real-time processing (use Kafka Streams instead)
```

➡️ [Part 60: Production Readiness Checklist](./Part-60-ProductionChecklist.md)
