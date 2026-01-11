# Asynchronous Operations & Long-Running Tasks

## What You'll Learn
How to design APIs that handle long-running operations without blocking clients. You'll understand the 202 Accepted pattern, status polling, webhooks for completion, and task lifecycle management.

## Why This Matters
Some operations take time: report generation (minutes), data exports (hours), batch processing (days). Synchronous APIs block clients, waste connection resources, and hit timeout limits. Async patterns enable responsive UIs, efficient resource usage, and scalable job processing. Users see immediate feedback ("Processing...") instead of frozen interfaces. Understanding async patterns is essential for reliable systems handling time-intensive work.

## The Problem with Synchronous Long-Running Tasks

**Bad: Synchronous**:
```http
POST /reports/generate
{"type": "annual-sales"}

... client waits 5 minutes ...

→ 200 OK (or timeout after 30s!)
{"reportUrl": "https://...pdf"}
```

**Problems**:
- Client connection hangs
- HTTP timeouts (30-60s typically)
- No progress feedback
- Can't cancel
- Retry on timeout = duplicate work

## Pattern: 202 Accepted with Status Polling

### Flow

```
1. Client initiates operation
   POST /reports
   → 202 Accepted
   Location: /reports/operations/op_abc123
   {"id": "op_abc123", "status": "pending"}

2. Client polls status
   GET /reports/operations/op_abc123
   → 200 OK
   {"id": "op_abc123", "status": "processing", "progress": 35}

3. Operation completes
   GET /reports/operations/op_abc123
   → 200 OK
   {
     "id": "op_abc123",
     "status": "completed",
     "result": {
       "reportUrl": "https://cdn.example.com/reports/abc123.pdf"
     }
   }
```

### Implementation

**Initiate Operation**:
```java
@PostMapping("/reports")
public ResponseEntity<OperationStatus> generateReport(@RequestBody ReportRequest request) {
    // Create operation record
    Operation operation = new Operation();
    operation.setId(UUID.randomUUID().toString());
    operation.setStatus(OperationStatus.PENDING);
    operation.setType("report_generation");
    operation.setCreatedAt(Instant.now());
    operationRepository.save(operation);
    
    // Submit to background queue
    reportQueue.submit(new ReportGenerationJob(operation.getId(), request));
    
    // Return 202 immediately
    return ResponseEntity
        .status(HttpStatus.ACCEPTED)
        .header("Location", "/reports/operations/" + operation.getId())
        .body(new OperationStatus(
            operation.getId(),
            "pending",
            null,
            null
        ));
}
```

**Check Status**:
```java
@GetMapping("/reports/operations/{id}")
public ResponseEntity<OperationResponse> getOperationStatus(@PathVariable String id) {
    Operation operation = operationRepository.findById(id)
        .orElseThrow(() -> new NotFoundException("Operation not found"));
    
    OperationResponse response = new OperationResponse();
    response.setId(operation.getId());
    response.setStatus(operation.getStatus());
    response.setProgress(operation.getProgress());
    response.setCreatedAt(operation.getCreatedAt());
    response.setUpdatedAt(operation.getUpdatedAt());
    
    if (operation.getStatus() == OperationStatus.COMPLETED) {
        response.setResult(operation.getResult());
        response.setCompletedAt(operation.getCompletedAt());
    } else if (operation.getStatus() == OperationStatus.FAILED) {
        response.setError(operation.getError());
    } else if (operation.getStatus() == OperationStatus.PROCESSING) {
        response.setEstimatedCompletion(operation.getEstimatedCompletion());
    }
    
    return ResponseEntity.ok(response);
}
```

**Background Worker**:
```java
@Component
public class ReportGenerationWorker {
    
    @RabbitListener(queues = "report-generation")
    public void processReportJob(ReportGenerationJob job) {
        Operation operation = operationRepository.findById(job.getOperationId())
            .orElseThrow();
        
        try {
            // Mark as processing
            operation.setStatus(OperationStatus.PROCESSING);
            operation.setStartedAt(Instant.now());
            operationRepository.save(operation);
            
            // Generate report (long-running)
            for (int i = 0; i <= 100; i += 10) {
                // Update progress
                operation.setProgress(i);
                operation.setEstimatedCompletion(
                    calculateEstimation(operation.getStartedAt(), i)
                );
                operationRepository.save(operation);
                
                // Do work
                processChunk(job, i);
            }
            
            // Upload result
            String reportUrl = uploadReport(job.getResult());
            
            // Mark completed
            operation.setStatus(OperationStatus.COMPLETED);
            operation.setCompletedAt(Instant.now());
            operation.setResult(Map.of("reportUrl", reportUrl));
            operationRepository.save(operation);
            
            // Optionally: send webhook
            webhookService.sendCompletion(operation);
            
        } catch (Exception e) {
            operation.setStatus(OperationStatus.FAILED);
            operation.setError(Map.of(
                "type", e.getClass().getSimpleName(),
                "message", e.getMessage()
            ));
            operationRepository.save(operation);
        }
    }
}
```

### Operation States

```
pending → Initial state
   ↓
processing → Work in progress
   │
   ├───→ completed (success)
   ├───→ failed (error)
   └───→ cancelled (user cancelled)
```

**States**:
- `pending`: Queued, not started
- `processing`: Currently executing
- `completed`: Successfully finished
- `failed`: Error occurred
- `cancelled`: User cancelled

### Response Examples

**Pending**:
```json
{
  "id": "op_abc123",
  "status": "pending",
  "created_at": "2026-01-11T10:30:00Z",
  "updated_at": "2026-01-11T10:30:00Z"
}
```

**Processing**:
```json
{
  "id": "op_abc123",
  "status": "processing",
  "progress": 65,
  "estimated_completion": "2026-01-11T10:45:00Z",
  "created_at": "2026-01-11T10:30:00Z",
  "started_at": "2026-01-11T10:32:00Z",
  "updated_at": "2026-01-11T10:40:00Z"
}
```

**Completed**:
```json
{
  "id": "op_abc123",
  "status": "completed",
  "progress": 100,
  "result": {
    "reportUrl": "https://cdn.example.com/reports/abc123.pdf",
    "fileSize": 2457600,
    "expiresAt": "2026-01-18T10:45:00Z"
  },
  "created_at": "2026-01-11T10:30:00Z",
  "started_at": "2026-01-11T10:32:00Z",
  "completed_at": "2026-01-11T10:45:30Z",
  "updated_at": "2026-01-11T10:45:30Z"
}
```

**Failed**:
```json
{
  "id": "op_abc123",
  "status": "failed",
  "error": {
    "type": "InsufficientDataException",
    "message": "No data available for specified date range",
    "code": "NO_DATA_IN_RANGE"
  },
  "created_at": "2026-01-11T10:30:00Z",
  "started_at": "2026-01-11T10:32:00Z",
  "failed_at": "2026-01-11T10:33:15Z",
  "updated_at": "2026-01-11T10:33:15Z"
}
```

## Cancellation

```java
@DeleteMapping("/reports/operations/{id}")
public ResponseEntity<Void> cancelOperation(@PathVariable String id) {
    Operation operation = operationRepository.findById(id)
        .orElseThrow(() -> new NotFoundException());
    
    if (operation.getStatus() == OperationStatus.COMPLETED ||
        operation.getStatus() == OperationStatus.FAILED) {
        // Already finished
        return ResponseEntity.status(HttpStatus.CONFLICT).build();
    }
    
    // Mark for cancellation
    operation.setStatus(OperationStatus.CANCELLING);
    operationRepository.save(operation);
    
    // Worker checks status periodically and stops
    
    return ResponseEntity.noContent().build();
}
```

**Worker Checks**:
```java
while (hasMoreWork()) {
    // Check if cancelled
    Operation current = operationRepository.findById(operation.getId()).orElseThrow();
    if (current.getStatus() == OperationStatus.CANCELLING) {
        operation.setStatus(OperationStatus.CANCELLED);
        operationRepository.save(operation);
        return;
    }
    
    // Continue work
    processNextChunk();
}
```

## Webhooks for Completion

**Alternative to Polling**: Client registers callback URL

```json
POST /reports
{
  "type": "annual-sales",
  "webhook_url": "https://myapp.example.com/webhooks/reports"
}

→ 202 Accepted
{"id": "op_abc123", "status": "pending"}

// When complete, server POSTs to webhook_url:
POST https://myapp.example.com/webhooks/reports
{
  "event": "operation.completed",
  "operation_id": "op_abc123",
  "status": "completed",
  "result": {
    "reportUrl": "https://cdn.example.com/reports/abc123.pdf"
  }
}
```

**Benefit**: No polling; immediate notification

## TTL & Cleanup

**Problem**: Operation records grow unbounded

**Solution**: TTL and cleanup

```java
@Scheduled(cron = "0 0 2 * * *")  // Daily at 2 AM
public void cleanupOldOperations() {
    Instant cutoff = Instant.now().minus(7, ChronoUnit.DAYS);
    
    // Delete completed operations older than 7 days
    operationRepository.deleteByStatusAndCompletedAtBefore(
        OperationStatus.COMPLETED,
        cutoff
    );
    
    // Delete failed operations older than 30 days
    Instant failedCutoff = Instant.now().minus(30, ChronoUnit.DAYS);
    operationRepository.deleteByStatusAndFailedAtBefore(
        OperationStatus.FAILED,
        failedCutoff
    );
    
    log.info("Cleaned up old operations");
}
```

**Document TTL**:
```
Operation records are retained for:
- Completed: 7 days
- Failed: 30 days
- Pending/Processing: Until completion or 24 hours timeout
```

## Idempotency for Async Operations

**Problem**: Client retries POST; creates duplicate jobs

**Solution**: Idempotency keys

```java
@PostMapping("/reports")
public ResponseEntity<OperationStatus> generateReport(
    @RequestHeader("Idempotency-Key") String idempotencyKey,
    @RequestBody ReportRequest request
) {
    // Check if operation already exists for this key
    Optional<Operation> existing = operationRepository
        .findByIdempotencyKey(idempotencyKey);
    
    if (existing.isPresent()) {
        // Return existing operation
        return ResponseEntity
            .status(HttpStatus.OK)  // or 202
            .header("Location", "/reports/operations/" + existing.get().getId())
            .body(toOperationStatus(existing.get()));
    }
    
    // Create new operation
    Operation operation = new Operation();
    operation.setId(UUID.randomUUID().toString());
    operation.setIdempotencyKey(idempotencyKey);
    operation.setStatus(OperationStatus.PENDING);
    operationRepository.save(operation);
    
    reportQueue.submit(new ReportGenerationJob(operation.getId(), request));
    
    return ResponseEntity
        .status(HttpStatus.ACCEPTED)
        .header("Location", "/reports/operations/" + operation.getId())
        .body(toOperationStatus(operation));
}
```

## Client Implementation

### Polling with Backoff

```javascript
async function generateReport(reportRequest) {
  // Initiate
  const initResponse = await fetch('/reports', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Idempotency-Key': crypto.randomUUID()
    },
    body: JSON.stringify(reportRequest)
  });
  
  const operation = await initResponse.json();
  const operationUrl = initResponse.headers.get('Location');
  
  // Poll with increasing intervals
  const pollIntervals = [1000, 2000, 5000, 10000, 30000];  // ms
  let intervalIndex = 0;
  
  while (true) {
    await sleep(pollIntervals[Math.min(intervalIndex, pollIntervals.length - 1)]);
    
    const statusResponse = await fetch(operationUrl);
    const status = await statusResponse.json();
    
    console.log(`Status: ${status.status}, Progress: ${status.progress}%`);
    
    if (status.status === 'completed') {
      return status.result;
    }
    
    if (status.status === 'failed') {
      throw new Error(status.error.message);
    }
    
    intervalIndex++;
  }
}
```

### React Hook Example

```javascript
function useAsyncOperation(operationUrl) {
  const [operation, setOperation] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    async function poll() {
      try {
        const response = await fetch(operationUrl);
        const data = await response.json();
        
        if (cancelled) return;
        
        setOperation(data);
        
        if (data.status === 'pending' || data.status === 'processing') {
          // Poll again after delay
          setTimeout(poll, 2000);
        } else {
          setLoading(false);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err);
          setLoading(false);
        }
      }
    }
    
    poll();
    
    return () => { cancelled = true; };
  }, [operationUrl]);
  
  return { operation, loading, error };
}

// Usage:
function ReportPage() {
  const { operation, loading, error } = useAsyncOperation('/reports/operations/op_abc123');
  
  if (loading) return <div>Generating report: {operation?.progress}%</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (operation.status === 'completed') {
    return <a href={operation.result.reportUrl}>Download Report</a>;
  }
}
```

## Monitoring

**Metrics**:
- Operation duration (p50, p95, p99)
- Success/failure rate
- Queue depth
- Time in each state
- Cancellation rate

**Alerts**:
- Queue backlog > 10,000
- Operations stuck in processing > 1 hour
- Failure rate > 5%

## Best Practices

1. **Return 202 Immediately**: Don't wait for processing
2. **Provide Location**: Header to status endpoint
3. **Progress Updates**: Update regularly during processing
4. **Idempotency**: Use idempotency keys
5. **TTL**: Clean up old operations
6. **Cancellation**: Support DELETE on operation
7. **Webhooks**: Offer alternative to polling
8. **Timeouts**: Abort stuck operations
9. **Error Details**: Include helpful error messages
10. **Documentation**: Clear polling recommendations

## Anti-Patterns

❌ **No progress feedback**: Client polls blindly
❌ **Synchronous disguised as async**: Returns 202 but blocks worker
❌ **Infinite polling**: No completion detection
❌ **Missing TTL**: Unbounded storage
❌ **No idempotency**: Duplicate operations on retry
