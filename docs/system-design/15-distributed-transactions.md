# Distributed Transactions

## What You'll Learn

- Two-phase commit (2PC) protocol
- Saga pattern for long-running transactions
- Compensation and rollback strategies
- Distributed transaction coordinators
- Trade-offs between consistency and availability

## Why This Matters

Distributed transactions are one of the hardest problems in distributed systems. When a business operation spans multiple services or databases, ensuring atomicity becomes complex. E-commerce orders involve inventory, payment, shipping - all must succeed or all must fail. Getting this wrong costs money and customer trust. Understanding when to use strong consistency (2PC) versus eventual consistency (Sagas) determines system reliability and performance.

## Two-Phase Commit (2PC)

Ensures atomicity across multiple participants. All nodes commit or all abort.

### Protocol Flow

**Phase 1: Prepare (Voting)**
1. Coordinator asks all participants to prepare
2. Participants lock resources and vote YES or NO
3. If any participant votes NO, abort

**Phase 2: Commit**
1. If all voted YES, coordinator sends COMMIT
2. If any voted NO, coordinator sends ABORT
3. Participants execute decision and release locks

```java
// Two-Phase Commit Coordinator
public class TwoPhaseCommitCoordinator {
    private final List<Participant> participants;
    private final TransactionLog log;
    
    public boolean executeTransaction(Transaction transaction) throws TransactionException {
        String transactionId = transaction.getId();
        log.write(transactionId, TransactionState.PREPARING);
        
        // Phase 1: Prepare
        List<Future<VoteResponse>> votes = new ArrayList<>();
        for (Participant participant : participants) {
            Future<VoteResponse> vote = executor.submit(() -> 
                participant.prepare(transactionId, transaction.getOperations())
            );
            votes.add(vote);
        }
        
        // Collect votes with timeout
        boolean allVotedYes = true;
        for (Future<VoteResponse> vote : votes) {
            try {
                VoteResponse response = vote.get(5, TimeUnit.SECONDS);
                if (response != VoteResponse.YES) {
                    allVotedYes = false;
                    break;
                }
            } catch (TimeoutException e) {
                allVotedYes = false;
                break;
            }
        }
        
        // Phase 2: Commit or Abort
        if (allVotedYes) {
            log.write(transactionId, TransactionState.COMMITTING);
            commitAll(transactionId);
            log.write(transactionId, TransactionState.COMMITTED);
            return true;
        } else {
            log.write(transactionId, TransactionState.ABORTING);
            abortAll(transactionId);
            log.write(transactionId, TransactionState.ABORTED);
            return false;
        }
    }
    
    private void commitAll(String transactionId) {
        for (Participant participant : participants) {
            try {
                participant.commit(transactionId);
            } catch (Exception e) {
                // Participant must eventually commit (crash recovery)
                log.error("Participant {} failed to commit {}", participant, transactionId);
                // Retry mechanism needed
            }
        }
    }
    
    private void abortAll(String transactionId) {
        for (Participant participant : participants) {
            try {
                participant.abort(transactionId);
            } catch (Exception e) {
                log.error("Participant {} failed to abort {}", participant, transactionId);
            }
        }
    }
}

// Participant implementation
public class DatabaseParticipant implements Participant {
    private final DataSource dataSource;
    private final Map<String, Connection> preparedTransactions = new ConcurrentHashMap<>();
    
    @Override
    public VoteResponse prepare(String transactionId, List<Operation> operations) {
        Connection conn = null;
        try {
            conn = dataSource.getConnection();
            conn.setAutoCommit(false);
            
            // Execute operations
            for (Operation op : operations) {
                executeOperation(conn, op);
            }
            
            // Lock acquired, ready to commit
            preparedTransactions.put(transactionId, conn);
            return VoteResponse.YES;
            
        } catch (SQLException e) {
            // Cannot prepare, vote NO
            if (conn != null) {
                try { conn.rollback(); } catch (SQLException ex) {}
                try { conn.close(); } catch (SQLException ex) {}
            }
            return VoteResponse.NO;
        }
    }
    
    @Override
    public void commit(String transactionId) {
        Connection conn = preparedTransactions.remove(transactionId);
        try {
            conn.commit();
        } catch (SQLException e) {
            throw new RuntimeException("Commit failed", e);
        } finally {
            try { conn.close(); } catch (SQLException e) {}
        }
    }
    
    @Override
    public void abort(String transactionId) {
        Connection conn = preparedTransactions.remove(transactionId);
        try {
            conn.rollback();
        } catch (SQLException e) {
            log.error("Rollback failed", e);
        } finally {
            try { conn.close(); } catch (SQLException e) {}
        }
    }
}
```

### Problems with 2PC

**Blocking**: Participants hold locks during both phases. If coordinator crashes, participants are blocked.

**Single Point of Failure**: Coordinator failure can leave system in uncertain state.

**Performance**: Synchronous protocol with multiple round trips causes high latency.

```python
# Demonstrating blocking problem
class BlockingScenario:
    def demonstrate_blocking(self):
        """Show how 2PC can block indefinitely"""
        
        # T1: Transaction starts
        coordinator.prepare(participants)
        
        # All participants vote YES and wait for decision
        # Locks held on inventory, payment, shipping
        
        # COORDINATOR CRASHES HERE - before sending commit/abort
        
        # Participants are stuck:
        # - Cannot commit (didn't receive COMMIT)
        # - Cannot abort (don't know if others committed)
        # - Must hold locks (atomicity requirement)
        # Result: System blocked until coordinator recovers
```

## Saga Pattern

Alternative to 2PC for long-running transactions. Series of local transactions with compensating actions.

### Choreography-Based Saga

Services communicate through events. No central coordinator.

```javascript
// Order saga with choreography
class OrderService {
    async createOrder(orderData) {
        const order = await this.db.createOrder({
            ...orderData,
            status: 'PENDING'
        });
        
        // Publish event
        await eventBus.publish('OrderCreated', {
            orderId: order.id,
            items: order.items,
            userId: order.userId,
            amount: order.total
        });
        
        return order;
    }
    
    // Compensation: Cancel order
    async cancelOrder(orderId, reason) {
        await this.db.updateOrder(orderId, {
            status: 'CANCELLED',
            cancelReason: reason
        });
        
        await eventBus.publish('OrderCancelled', {
            orderId,
            reason
        });
    }
}

class InventoryService {
    constructor() {
        // Listen for order events
        eventBus.subscribe('OrderCreated', this.reserveInventory.bind(this));
        eventBus.subscribe('PaymentFailed', this.releaseInventory.bind(this));
    }
    
    async reserveInventory(event) {
        const { orderId, items } = event;
        
        try {
            // Check stock
            const available = await this.checkStock(items);
            
            if (available) {
                await this.db.reserveItems(orderId, items);
                
                await eventBus.publish('InventoryReserved', {
                    orderId,
                    items
                });
            } else {
                await eventBus.publish('InventoryFailed', {
                    orderId,
                    reason: 'Out of stock'
                });
            }
        } catch (error) {
            await eventBus.publish('InventoryFailed', {
                orderId,
                reason: error.message
            });
        }
    }
    
    // Compensation: Release reserved inventory
    async releaseInventory(event) {
        const { orderId } = event;
        await this.db.releaseReservation(orderId);
        
        await eventBus.publish('InventoryReleased', { orderId });
    }
}

class PaymentService {
    constructor() {
        eventBus.subscribe('InventoryReserved', this.processPayment.bind(this));
        eventBus.subscribe('OrderCancelled', this.refundPayment.bind(this));
    }
    
    async processPayment(event) {
        const { orderId, amount } = event;
        
        try {
            const payment = await this.paymentGateway.charge(amount);
            
            await this.db.recordPayment({
                orderId,
                paymentId: payment.id,
                amount,
                status: 'SUCCESS'
            });
            
            await eventBus.publish('PaymentSucceeded', {
                orderId,
                paymentId: payment.id
            });
            
        } catch (error) {
            await eventBus.publish('PaymentFailed', {
                orderId,
                reason: error.message
            });
        }
    }
    
    // Compensation: Refund payment
    async refundPayment(event) {
        const { orderId } = event;
        const payment = await this.db.findPayment(orderId);
        
        if (payment) {
            await this.paymentGateway.refund(payment.paymentId);
            await this.db.updatePayment(payment.id, { status: 'REFUNDED' });
        }
    }
}
```

### Orchestration-Based Saga

Central orchestrator coordinates the saga.

```java
// Saga orchestrator
public class OrderSagaOrchestrator {
    private final OrderService orderService;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    
    public void executeOrderSaga(OrderRequest request) {
        String sagaId = UUID.randomUUID().toString();
        SagaState state = new SagaState(sagaId);
        
        try {
            // Step 1: Create order
            Order order = orderService.createOrder(request);
            state.addStep("createOrder", order.getId());
            
            // Step 2: Reserve inventory
            InventoryReservation reservation = inventoryService.reserve(
                order.getItems()
            );
            state.addStep("reserveInventory", reservation.getId());
            
            // Step 3: Process payment
            Payment payment = paymentService.charge(
                order.getUserId(),
                order.getTotal()
            );
            state.addStep("processPayment", payment.getId());
            
            // Step 4: Schedule shipping
            Shipment shipment = shippingService.schedule(
                order.getId(),
                order.getShippingAddress()
            );
            state.addStep("scheduleShipping", shipment.getId());
            
            // All steps succeeded
            state.setStatus(SagaStatus.COMPLETED);
            sagaRepository.save(state);
            
        } catch (Exception e) {
            // Compensation flow
            log.error("Saga {} failed at step {}", sagaId, state.getCurrentStep(), e);
            compensate(state);
        }
    }
    
    private void compensate(SagaState state) {
        // Execute compensating transactions in reverse order
        List<SagaStep> completedSteps = state.getCompletedSteps();
        Collections.reverse(completedSteps);
        
        for (SagaStep step : completedSteps) {
            try {
                switch (step.getName()) {
                    case "scheduleShipping":
                        shippingService.cancelShipment(step.getResourceId());
                        break;
                    case "processPayment":
                        paymentService.refund(step.getResourceId());
                        break;
                    case "reserveInventory":
                        inventoryService.release(step.getResourceId());
                        break;
                    case "createOrder":
                        orderService.cancel(step.getResourceId());
                        break;
                }
            } catch (Exception e) {
                // Log compensation failure for manual intervention
                log.error("Compensation failed for step {}", step.getName(), e);
                state.addFailedCompensation(step);
            }
        }
        
        state.setStatus(SagaStatus.COMPENSATED);
        sagaRepository.save(state);
    }
}

// Saga state management
public class SagaState {
    private String sagaId;
    private SagaStatus status;
    private List<SagaStep> steps = new ArrayList<>();
    private List<SagaStep> failedCompensations = new ArrayList<>();
    
    public void addStep(String name, String resourceId) {
        steps.add(new SagaStep(name, resourceId, Instant.now()));
    }
    
    public String getCurrentStep() {
        return steps.isEmpty() ? null : steps.get(steps.size() - 1).getName();
    }
    
    public List<SagaStep> getCompletedSteps() {
        return new ArrayList<>(steps);
    }
}
```

## Distributed Transaction Coordinators

### XA Transactions

Industry standard for distributed transactions.

```java
// XA transaction with multiple databases
public class XATransactionExample {
    
    public void transferMoneyAcrossBanks(
        String fromAccount, 
        String toAccount, 
        BigDecimal amount
    ) throws Exception {
        
        // XA transaction manager
        UserTransaction utx = (UserTransaction) new InitialContext()
            .lookup("java:comp/UserTransaction");
        
        try {
            utx.begin();
            
            // Database 1: Bank A
            Connection conn1 = xaDataSource1.getXAConnection().getConnection();
            PreparedStatement stmt1 = conn1.prepareStatement(
                "UPDATE accounts SET balance = balance - ? WHERE account_id = ?"
            );
            stmt1.setBigDecimal(1, amount);
            stmt1.setString(2, fromAccount);
            stmt1.executeUpdate();
            
            // Database 2: Bank B
            Connection conn2 = xaDataSource2.getXAConnection().getConnection();
            PreparedStatement stmt2 = conn2.prepareStatement(
                "UPDATE accounts SET balance = balance + ? WHERE account_id = ?"
            );
            stmt2.setBigDecimal(1, amount);
            stmt2.setString(2, toAccount);
            stmt2.executeUpdate();
            
            // Commit both databases atomically
            utx.commit();
            
        } catch (Exception e) {
            utx.rollback();
            throw e;
        }
    }
}
```

## Try-Cancel/Confirm (TCC) Pattern

Three-phase protocol optimized for microservices.

```python
# TCC pattern implementation
class TCCCoordinator:
    def __init__(self):
        self.participants = []
    
    def execute_transaction(self, transaction_id, operations):
        """Execute TCC transaction"""
        
        # Phase 1: Try (reserve resources)
        reserved = []
        for operation in operations:
            try:
                result = operation.try_reserve(transaction_id)
                reserved.append((operation, result))
            except Exception as e:
                # Try failed, cancel all reserved
                self.cancel_all(transaction_id, reserved)
                raise TransactionFailedException(f"Try failed: {e}")
        
        # Phase 2: Confirm or Cancel
        try:
            # Business validation
            if not self.validate_transaction(reserved):
                self.cancel_all(transaction_id, reserved)
                return False
            
            # Confirm all operations
            for operation, result in reserved:
                operation.confirm(transaction_id, result)
            
            return True
            
        except Exception as e:
            # Confirm failed, cancel
            self.cancel_all(transaction_id, reserved)
            raise
    
    def cancel_all(self, transaction_id, reserved):
        """Cancel all reserved operations"""
        for operation, result in reserved:
            try:
                operation.cancel(transaction_id, result)
            except Exception as e:
                # Log for manual intervention
                logger.error(f"Cancel failed: {e}")

# Example: Account service with TCC
class AccountServiceTCC:
    def try_reserve(self, transaction_id, account_id, amount):
        """Phase 1: Reserve funds"""
        # Check if sufficient balance
        account = self.db.get_account(account_id)
        if account.balance < amount:
            raise InsufficientFundsException()
        
        # Create reservation
        reservation = {
            'transaction_id': transaction_id,
            'account_id': account_id,
            'amount': amount,
            'status': 'RESERVED',
            'expires_at': datetime.now() + timedelta(minutes=5)
        }
        self.db.create_reservation(reservation)
        
        return reservation
    
    def confirm(self, transaction_id, reservation):
        """Phase 2: Confirm reservation"""
        # Deduct balance
        self.db.update_account(
            reservation['account_id'],
            balance=F('balance') - reservation['amount']
        )
        
        # Mark reservation as confirmed
        self.db.update_reservation(
            transaction_id,
            status='CONFIRMED'
        )
    
    def cancel(self, transaction_id, reservation):
        """Phase 3: Cancel reservation"""
        self.db.update_reservation(
            transaction_id,
            status='CANCELLED'
        )
```

## Comparison: 2PC vs Saga vs TCC

| Aspect | 2PC | Saga | TCC |
|--------|-----|------|-----|
| Consistency | Strong | Eventual | Strong |
| Availability | Low (blocking) | High | Medium |
| Performance | Slow | Fast | Medium |
| Complexity | Medium | High | High |
| Rollback | Automatic | Compensation | Automatic |
| Use Case | Short transactions | Long-running | Microservices |

## Best Practices

**Choose Sagas for Long-Running Transactions**: 2PC holds locks too long. Sagas release resources immediately.

**Idempotent Operations**: Compensations may execute multiple times. Ensure operations are idempotent.

**Saga State Persistence**: Store saga progress to recover from failures.

**Timeouts and Retries**: Handle network failures with exponential backoff.

**Monitoring**: Track saga success rates, compensation frequency, and duration.

## Anti-Patterns

❌ **Using 2PC for Long Transactions**: Locks held too long, poor availability

❌ **No Compensation Logic**: Sagas without compensations leave inconsistent state

❌ **Non-Idempotent Compensations**: Retry compensation can cause errors

❌ **Distributed Transactions Everywhere**: Most operations don't need strong consistency

❌ **Ignoring Saga Failures**: Failed compensations require manual intervention

## Real-World Examples

**Amazon**: Uses Saga pattern for order processing. Inventory reservation, payment, shipping coordinated via events.

**Uber**: Trip booking uses Saga. Driver matching, payment authorization, trip creation can compensate.

**Netflix**: Saga pattern for account management. Subscription changes, billing, content access.

**Google Spanner**: One of few systems with native distributed transactions using TrueTime for global consistency.

## Related Topics

- [Distributed Systems Fundamentals](11-distributed-systems-fundamentals.md)
- [Message Queues](06-message-queues.md)
- [CQRS & Event Sourcing](16-cqrs-event-sourcing.md)
- [Database Scaling Strategies](03-database-scaling-strategies.md)
