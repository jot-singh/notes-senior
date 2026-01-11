# Distributed Systems Fundamentals

## What You'll Learn

- Distributed system characteristics and challenges
- Consistency models and their trade-offs
- Distributed consensus algorithms (Paxos, Raft)
- Clock synchronization and ordering of events
- Distributed system guarantees and theorems

## Why This Matters

Modern applications run across multiple machines, data centers, and geographic regions. Understanding distributed systems fundamentals is essential for building reliable, scalable services. Companies like Google, Amazon, and Netflix rely on distributed systems principles to serve billions of users. Knowing when to prioritize consistency over availability, how to handle network partitions, and how to achieve consensus across nodes determines whether your system can scale reliably.

## Distributed System Characteristics

A distributed system is a collection of independent computers that appears to users as a single coherent system. Key characteristics include:

**No Shared Clock**: Each node has its own clock, making it difficult to order events across the system. This creates challenges for determining causality and maintaining consistency.

**Independent Failures**: Any component can fail independently. A node crash, network partition, or disk failure affects only part of the system, but the system must continue operating.

**Concurrent Operations**: Multiple nodes process requests simultaneously, leading to race conditions, conflicts, and the need for coordination mechanisms.

**Network Unreliability**: Messages can be delayed, reordered, duplicated, or lost. The network itself can partition, splitting the system into isolated groups.

## The Eight Fallacies of Distributed Computing

Peter Deutsch and James Gosling identified assumptions developers incorrectly make:

1. **The network is reliable** - Networks fail, packets drop
2. **Latency is zero** - Network calls add significant delay
3. **Bandwidth is infinite** - Network capacity has limits
4. **The network is secure** - Security must be designed in
5. **Topology doesn't change** - Network configuration changes frequently
6. **There is one administrator** - Multiple teams manage components
7. **Transport cost is zero** - Serialization and network calls have overhead
8. **The network is homogeneous** - Mixed technologies and protocols exist

## Consistency Models

Consistency models define the guarantees a distributed system provides about the order and visibility of operations.

### Strong Consistency (Linearizability)

All operations appear to execute atomically at some point between invocation and response. Once a write completes, all subsequent reads see that write or a later value.

```java
// Example: Bank account with strong consistency
public class StronglyConsistentAccount {
    private final DistributedLock lock;
    private final Database database;
    
    public void transfer(String fromId, String toId, BigDecimal amount) {
        lock.acquire(fromId, toId); // Distributed lock ensures serialization
        try {
            BigDecimal fromBalance = database.read(fromId);
            BigDecimal toBalance = database.read(toId);
            
            if (fromBalance.compareTo(amount) >= 0) {
                database.write(fromId, fromBalance.subtract(amount));
                database.write(toId, toBalance.add(amount));
            }
        } finally {
            lock.release(fromId, toId);
        }
    }
}
```

**Use Cases**: Financial transactions, inventory management, leader election

**Trade-offs**: Lower availability during network partitions, higher latency due to coordination

### Sequential Consistency

Operations from each process appear in program order, but global ordering may differ from real-time order. All processes see the same order of operations.

```python
# Example: Sequential consistency with version vectors
class SequentiallyConsistentStore:
    def __init__(self):
        self.data = {}
        self.version_vector = {}
        
    def write(self, key, value, process_id):
        # Increment version for this process
        self.version_vector[process_id] = \
            self.version_vector.get(process_id, 0) + 1
        
        self.data[key] = {
            'value': value,
            'version': dict(self.version_vector)
        }
        
    def read(self, key):
        if key in self.data:
            return self.data[key]['value']
        return None
```

**Use Cases**: Collaborative editing, distributed caches

**Trade-offs**: Easier to implement than strong consistency, still requires coordination

### Causal Consistency

Operations that are causally related are seen in the same order by all processes. Concurrent operations may be seen in different orders.

```javascript
// Example: Causal consistency with Lamport timestamps
class CausallyConsistentStore {
    constructor() {
        this.data = new Map();
        this.lamportClock = 0;
    }
    
    write(key, value) {
        this.lamportClock++;
        this.data.set(key, {
            value: value,
            timestamp: this.lamportClock
        });
        return this.lamportClock;
    }
    
    read(key) {
        const entry = this.data.get(key);
        if (entry) {
            // Update clock on read to maintain causality
            this.lamportClock = Math.max(this.lamportClock, entry.timestamp) + 1;
            return entry.value;
        }
        return null;
    }
}
```

**Use Cases**: Social media feeds, comment threads

**Trade-offs**: More available than sequential consistency, complex to implement correctly

### Eventual Consistency

Given no new updates, all replicas eventually converge to the same state. No guarantees about when convergence happens.

```java
// Example: Eventual consistency with conflict resolution
public class EventuallyConsistentStore {
    private Map<String, VersionedValue> data = new ConcurrentHashMap<>();
    
    public void write(String key, String value, long timestamp) {
        data.compute(key, (k, existing) -> {
            if (existing == null || timestamp > existing.timestamp) {
                return new VersionedValue(value, timestamp);
            }
            // Keep existing value if it's newer (last-write-wins)
            return existing;
        });
    }
    
    public String read(String key) {
        VersionedValue value = data.get(key);
        return value != null ? value.value : null;
    }
    
    // Periodic sync with other nodes
    public void sync(EventuallyConsistentStore other) {
        for (Map.Entry<String, VersionedValue> entry : other.data.entrySet()) {
            write(entry.getKey(), entry.getValue().value, entry.getValue().timestamp);
        }
    }
    
    private static class VersionedValue {
        String value;
        long timestamp;
        
        VersionedValue(String value, long timestamp) {
            this.value = value;
            this.timestamp = timestamp;
        }
    }
}
```

**Use Cases**: DNS, caching, shopping carts, user profiles

**Trade-offs**: High availability, potential for temporary inconsistencies

## Consensus Algorithms

Consensus ensures multiple nodes agree on a single value despite failures. This is fundamental for leader election, distributed transactions, and maintaining replicated state.

### Paxos

Paxos is a family of protocols for achieving consensus in asynchronous networks. It guarantees safety (agreement on a single value) even with message delays and failures.

**Three Phases**:

1. **Prepare Phase**: Proposer selects proposal number and sends to acceptors
2. **Promise Phase**: Acceptors promise not to accept lower-numbered proposals
3. **Accept Phase**: Proposer sends value, acceptors accept if they haven't promised higher number

```python
# Simplified Paxos implementation
class PaxosNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers
        self.promised_proposal = None
        self.accepted_proposal = None
        self.accepted_value = None
        
    def prepare(self, proposal_number):
        """Phase 1: Prepare request"""
        if self.promised_proposal is None or proposal_number > self.promised_proposal:
            self.promised_proposal = proposal_number
            return {
                'promise': True,
                'accepted_proposal': self.accepted_proposal,
                'accepted_value': self.accepted_value
            }
        return {'promise': False}
    
    def accept(self, proposal_number, value):
        """Phase 2: Accept request"""
        if self.promised_proposal is None or proposal_number >= self.promised_proposal:
            self.promised_proposal = proposal_number
            self.accepted_proposal = proposal_number
            self.accepted_value = value
            return {'accepted': True}
        return {'accepted': False}
    
    def propose_value(self, value):
        """Full Paxos protocol to propose a value"""
        proposal_number = self.generate_proposal_number()
        
        # Phase 1: Send prepare to majority
        promises = []
        for peer in self.peers:
            response = peer.prepare(proposal_number)
            if response['promise']:
                promises.append(response)
        
        if len(promises) < len(self.peers) // 2 + 1:
            return None  # Failed to get majority
        
        # Use value from highest accepted proposal, or our value
        highest_proposal = max(
            (p['accepted_proposal'] for p in promises if p['accepted_proposal']),
            default=None
        )
        if highest_proposal:
            value = next(p['accepted_value'] for p in promises 
                        if p['accepted_proposal'] == highest_proposal)
        
        # Phase 2: Send accept to majority
        accepts = []
        for peer in self.peers:
            response = peer.accept(proposal_number, value)
            if response['accepted']:
                accepts.append(response)
        
        if len(accepts) >= len(self.peers) // 2 + 1:
            return value  # Consensus reached
        return None
    
    def generate_proposal_number(self):
        """Generate unique, increasing proposal number"""
        import time
        return int(time.time() * 1000) * 1000 + self.node_id
```

**Use Cases**: Google Chubby lock service, Apache ZooKeeper foundation

**Trade-offs**: Complex to implement correctly, requires majority for progress

### Raft

Raft achieves the same consensus guarantees as Paxos but is designed to be more understandable. It separates leader election, log replication, and safety.

**Key Components**:

- **Leader Election**: Nodes vote for leader using randomized timeouts
- **Log Replication**: Leader appends entries to followers' logs
- **Safety**: Committed entries never lost, all nodes execute same commands

```java
// Raft node implementation
public class RaftNode {
    enum State { FOLLOWER, CANDIDATE, LEADER }
    
    private State state = State.FOLLOWER;
    private int currentTerm = 0;
    private String votedFor = null;
    private List<LogEntry> log = new ArrayList<>();
    private int commitIndex = 0;
    private int lastApplied = 0;
    
    // Leader state
    private Map<String, Integer> nextIndex = new HashMap<>();
    private Map<String, Integer> matchIndex = new HashMap<>();
    
    private List<RaftNode> peers;
    private ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    private long electionTimeout;
    
    public RaftNode(String nodeId, List<RaftNode> peers) {
        this.votedFor = null;
        this.peers = peers;
        resetElectionTimeout();
        startElectionTimer();
    }
    
    private void startElectionTimer() {
        scheduler.schedule(() -> {
            if (state != State.LEADER) {
                startElection();
            }
        }, electionTimeout, TimeUnit.MILLISECONDS);
    }
    
    private void startElection() {
        state = State.CANDIDATE;
        currentTerm++;
        votedFor = "self";
        resetElectionTimeout();
        
        int votesReceived = 1; // Vote for self
        
        // Request votes from all peers
        for (RaftNode peer : peers) {
            VoteResponse response = peer.requestVote(
                currentTerm, 
                log.size() - 1, 
                log.isEmpty() ? 0 : log.get(log.size() - 1).term
            );
            
            if (response.voteGranted) {
                votesReceived++;
            }
        }
        
        // Become leader if received majority
        if (votesReceived > (peers.size() + 1) / 2) {
            becomeLeader();
        } else {
            startElectionTimer();
        }
    }
    
    private void becomeLeader() {
        state = State.LEADER;
        
        // Initialize leader state
        for (RaftNode peer : peers) {
            nextIndex.put(peer.getNodeId(), log.size());
            matchIndex.put(peer.getNodeId(), 0);
        }
        
        // Send heartbeats
        sendHeartbeats();
    }
    
    private void sendHeartbeats() {
        if (state != State.LEADER) return;
        
        for (RaftNode peer : peers) {
            int prevLogIndex = nextIndex.get(peer.getNodeId()) - 1;
            int prevLogTerm = prevLogIndex >= 0 ? log.get(prevLogIndex).term : 0;
            
            List<LogEntry> entries = log.subList(
                nextIndex.get(peer.getNodeId()), 
                log.size()
            );
            
            AppendEntriesResponse response = peer.appendEntries(
                currentTerm,
                prevLogIndex,
                prevLogTerm,
                entries,
                commitIndex
            );
            
            if (response.success) {
                nextIndex.put(peer.getNodeId(), log.size());
                matchIndex.put(peer.getNodeId(), log.size() - 1);
            } else {
                nextIndex.put(peer.getNodeId(), nextIndex.get(peer.getNodeId()) - 1);
            }
        }
        
        // Schedule next heartbeat
        scheduler.schedule(this::sendHeartbeats, 50, TimeUnit.MILLISECONDS);
    }
    
    public VoteResponse requestVote(int term, int lastLogIndex, int lastLogTerm) {
        if (term < currentTerm) {
            return new VoteResponse(currentTerm, false);
        }
        
        if (term > currentTerm) {
            currentTerm = term;
            votedFor = null;
            state = State.FOLLOWER;
        }
        
        boolean logIsUpToDate = lastLogTerm > getLastLogTerm() ||
            (lastLogTerm == getLastLogTerm() && lastLogIndex >= log.size() - 1);
        
        if ((votedFor == null) && logIsUpToDate) {
            votedFor = "candidate";
            resetElectionTimeout();
            return new VoteResponse(currentTerm, true);
        }
        
        return new VoteResponse(currentTerm, false);
    }
    
    public AppendEntriesResponse appendEntries(
        int term, 
        int prevLogIndex, 
        int prevLogTerm,
        List<LogEntry> entries, 
        int leaderCommit
    ) {
        if (term < currentTerm) {
            return new AppendEntriesResponse(currentTerm, false);
        }
        
        resetElectionTimeout();
        
        if (term > currentTerm) {
            currentTerm = term;
            votedFor = null;
        }
        state = State.FOLLOWER;
        
        // Check log consistency
        if (prevLogIndex >= 0 && 
            (prevLogIndex >= log.size() || log.get(prevLogIndex).term != prevLogTerm)) {
            return new AppendEntriesResponse(currentTerm, false);
        }
        
        // Append new entries
        int index = prevLogIndex + 1;
        for (LogEntry entry : entries) {
            if (index < log.size()) {
                if (log.get(index).term != entry.term) {
                    log = log.subList(0, index);
                    log.add(entry);
                }
            } else {
                log.add(entry);
            }
            index++;
        }
        
        // Update commit index
        if (leaderCommit > commitIndex) {
            commitIndex = Math.min(leaderCommit, log.size() - 1);
        }
        
        return new AppendEntriesResponse(currentTerm, true);
    }
    
    private void resetElectionTimeout() {
        // Randomized timeout between 150-300ms
        electionTimeout = 150 + (long)(Math.random() * 150);
    }
    
    private int getLastLogTerm() {
        return log.isEmpty() ? 0 : log.get(log.size() - 1).term;
    }
    
    private String getNodeId() {
        return "node-" + hashCode();
    }
    
    static class LogEntry {
        int term;
        String command;
        
        LogEntry(int term, String command) {
            this.term = term;
            this.command = command;
        }
    }
    
    static class VoteResponse {
        int term;
        boolean voteGranted;
        
        VoteResponse(int term, boolean voteGranted) {
            this.term = term;
            this.voteGranted = voteGranted;
        }
    }
    
    static class AppendEntriesResponse {
        int term;
        boolean success;
        
        AppendEntriesResponse(int term, boolean success) {
            this.term = term;
            this.success = success;
        }
    }
}
```

**Use Cases**: etcd, Consul, CockroachDB

**Trade-offs**: Easier to understand than Paxos, requires majority for progress

## Logical Clocks

Physical clocks drift and cannot be perfectly synchronized across distributed systems. Logical clocks provide ordering without relying on physical time.

### Lamport Timestamps

Each process maintains a counter that increments on every event. On message send, include timestamp. On receive, update timestamp to max(local, received) + 1.

```javascript
class LamportClock {
    constructor() {
        this.time = 0;
    }
    
    // Increment on local event
    tick() {
        this.time++;
        return this.time;
    }
    
    // Update on message receipt
    update(messageTime) {
        this.time = Math.max(this.time, messageTime) + 1;
        return this.time;
    }
    
    // Send message with timestamp
    sendMessage(message) {
        this.tick();
        return {
            ...message,
            timestamp: this.time
        };
    }
    
    // Receive message and update clock
    receiveMessage(message) {
        this.update(message.timestamp);
        return message;
    }
}

// Example usage
const process1 = new LamportClock();
const process2 = new LamportClock();

// Process 1 sends message
const msg = process1.sendMessage({ data: 'hello' });
console.log('Sent with timestamp:', msg.timestamp); // 1

// Process 2 receives and processes
process2.receiveMessage(msg);
console.log('Process 2 time after receive:', process2.time); // 2

// Process 2 does local work
process2.tick();
console.log('Process 2 time after local event:', process2.time); // 3
```

**Limitation**: Lamport timestamps provide partial ordering. If event A happened before event B, A's timestamp is less than B's, but the reverse isn't necessarily true.

### Vector Clocks

Each process maintains a vector of timestamps for all processes. This captures causality more precisely than Lamport clocks.

```python
class VectorClock:
    def __init__(self, process_id, num_processes):
        self.process_id = process_id
        self.clock = [0] * num_processes
    
    def tick(self):
        """Increment own clock on local event"""
        self.clock[self.process_id] += 1
        return list(self.clock)
    
    def update(self, other_clock):
        """Update clock on message receipt"""
        for i in range(len(self.clock)):
            self.clock[i] = max(self.clock[i], other_clock[i])
        self.clock[self.process_id] += 1
        return list(self.clock)
    
    def send_message(self, message):
        """Send message with vector clock"""
        self.tick()
        return {
            **message,
            'vector_clock': list(self.clock)
        }
    
    def receive_message(self, message):
        """Receive message and update vector clock"""
        self.update(message['vector_clock'])
        return message
    
    def happens_before(self, other_clock):
        """Check if this clock happened before another"""
        less_or_equal = all(
            self.clock[i] <= other_clock[i] 
            for i in range(len(self.clock))
        )
        strictly_less = any(
            self.clock[i] < other_clock[i] 
            for i in range(len(self.clock))
        )
        return less_or_equal and strictly_less
    
    def concurrent(self, other_clock):
        """Check if events are concurrent"""
        return not self.happens_before(other_clock) and \
               not VectorClock.happens_before_static(other_clock, self.clock)
    
    @staticmethod
    def happens_before_static(clock1, clock2):
        less_or_equal = all(clock1[i] <= clock2[i] for i in range(len(clock1)))
        strictly_less = any(clock1[i] < clock2[i] for i in range(len(clock1)))
        return less_or_equal and strictly_less

# Example: Detecting concurrent writes
p1 = VectorClock(0, 3)  # Process 0 of 3
p2 = VectorClock(1, 3)  # Process 1 of 3
p3 = VectorClock(2, 3)  # Process 2 of 3

# Process 1 does work
p1.tick()  # [1, 0, 0]

# Process 2 does work
p2.tick()  # [0, 1, 0]

# Check if concurrent
print(p1.concurrent(p2.clock))  # True - concurrent events

# Process 1 sends to Process 3
msg = p1.send_message({'data': 'hello'})  # [2, 0, 0]

# Process 3 receives
p3.receive_message(msg)  # [2, 0, 1]

# Now p3 happened after p1
print(p1.happens_before(p3.clock))  # True
```

**Use Cases**: Amazon Dynamo, Riak, detecting conflicts in distributed systems

## Distributed System Patterns

### Quorum-Based Replication

Instead of requiring all replicas to acknowledge, require only a quorum (majority). If there are N replicas, and we require W write acknowledgments and R read acknowledgments, as long as W + R > N, reads will see the most recent write.

```java
public class QuorumReplication {
    private List<StorageNode> replicas;
    private int writeQuorum; // W
    private int readQuorum;  // R
    
    public QuorumReplication(List<StorageNode> replicas, int w, int r) {
        this.replicas = replicas;
        this.writeQuorum = w;
        this.readQuorum = r;
        
        // Validate: W + R > N ensures consistency
        if (w + r <= replicas.size()) {
            throw new IllegalArgumentException("W + R must be > N");
        }
    }
    
    public void write(String key, String value, long timestamp) throws QuorumException {
        ExecutorService executor = Executors.newFixedThreadPool(replicas.size());
        List<Future<Boolean>> futures = new ArrayList<>();
        
        // Send write to all replicas in parallel
        for (StorageNode replica : replicas) {
            futures.add(executor.submit(() -> {
                try {
                    replica.write(key, value, timestamp);
                    return true;
                } catch (Exception e) {
                    return false;
                }
            }));
        }
        
        // Wait for write quorum
        int successCount = 0;
        for (Future<Boolean> future : futures) {
            try {
                if (future.get(1, TimeUnit.SECONDS)) {
                    successCount++;
                    if (successCount >= writeQuorum) {
                        executor.shutdownNow();
                        return; // Success
                    }
                }
            } catch (Exception e) {
                // Continue waiting for other replicas
            }
        }
        
        executor.shutdownNow();
        throw new QuorumException("Failed to achieve write quorum");
    }
    
    public String read(String key) throws QuorumException {
        ExecutorService executor = Executors.newFixedThreadPool(replicas.size());
        List<Future<VersionedValue>> futures = new ArrayList<>();
        
        // Read from all replicas in parallel
        for (StorageNode replica : replicas) {
            futures.add(executor.submit(() -> replica.read(key)));
        }
        
        // Collect read quorum responses
        List<VersionedValue> responses = new ArrayList<>();
        for (Future<VersionedValue> future : futures) {
            try {
                VersionedValue value = future.get(1, TimeUnit.SECONDS);
                if (value != null) {
                    responses.add(value);
                    if (responses.size() >= readQuorum) {
                        break;
                    }
                }
            } catch (Exception e) {
                // Continue waiting for other replicas
            }
        }
        
        executor.shutdownNow();
        
        if (responses.size() < readQuorum) {
            throw new QuorumException("Failed to achieve read quorum");
        }
        
        // Return value with highest timestamp (last write wins)
        return responses.stream()
            .max(Comparator.comparing(v -> v.timestamp))
            .map(v -> v.value)
            .orElse(null);
    }
    
    static class VersionedValue {
        String value;
        long timestamp;
        
        VersionedValue(String value, long timestamp) {
            this.value = value;
            this.timestamp = timestamp;
        }
    }
    
    static class QuorumException extends Exception {
        QuorumException(String message) {
            super(message);
        }
    }
}
```

**Common Configurations**:
- **Strong consistency**: W + R > N (e.g., W=2, R=2, N=3)
- **Read-optimized**: W=N, R=1 (all writes must succeed, any read is valid)
- **Write-optimized**: W=1, R=N (any write succeeds, read all replicas)

### Gossip Protocol

Nodes periodically exchange information with random peers, eventually propagating data throughout the cluster. Used for failure detection, membership, and state dissemination.

```python
import random
import time
from dataclasses import dataclass
from typing import Set, Dict

@dataclass
class NodeInfo:
    node_id: str
    address: str
    heartbeat: int
    timestamp: float

class GossipProtocol:
    def __init__(self, node_id, peers, gossip_interval=1.0):
        self.node_id = node_id
        self.peers = peers  # List of other GossipProtocol instances
        self.membership: Dict[str, NodeInfo] = {}
        self.heartbeat = 0
        self.gossip_interval = gossip_interval
        
        # Initialize with self
        self.membership[node_id] = NodeInfo(
            node_id=node_id,
            address=f"node-{node_id}",
            heartbeat=0,
            timestamp=time.time()
        )
    
    def start_gossip(self):
        """Periodically gossip with random peers"""
        while True:
            time.sleep(self.gossip_interval)
            self.heartbeat += 1
            self.membership[self.node_id].heartbeat = self.heartbeat
            self.membership[self.node_id].timestamp = time.time()
            
            # Select random peer to gossip with
            if self.peers:
                peer = random.choice(self.peers)
                self.gossip_with(peer)
            
            # Clean up failed nodes
            self.detect_failures()
    
    def gossip_with(self, peer):
        """Exchange membership information with peer"""
        # Send our membership to peer
        peer_membership = peer.receive_gossip(self.membership)
        
        # Merge peer's membership with ours
        self.merge_membership(peer_membership)
    
    def receive_gossip(self, remote_membership: Dict[str, NodeInfo]):
        """Receive gossip from another node"""
        self.merge_membership(remote_membership)
        return dict(self.membership)  # Return our membership
    
    def merge_membership(self, remote_membership: Dict[str, NodeInfo]):
        """Merge remote membership info with local"""
        for node_id, remote_info in remote_membership.items():
            if node_id not in self.membership:
                # New node discovered
                self.membership[node_id] = remote_info
            else:
                local_info = self.membership[node_id]
                # Keep info with higher heartbeat
                if remote_info.heartbeat > local_info.heartbeat:
                    self.membership[node_id] = remote_info
    
    def detect_failures(self, timeout=5.0):
        """Detect failed nodes based on heartbeat timeout"""
        current_time = time.time()
        failed_nodes = []
        
        for node_id, info in list(self.membership.items()):
            if node_id != self.node_id:
                if current_time - info.timestamp > timeout:
                    failed_nodes.append(node_id)
        
        # Remove failed nodes
        for node_id in failed_nodes:
            del self.membership[node_id]
            print(f"Node {self.node_id}: Detected failure of {node_id}")
    
    def get_alive_nodes(self) -> Set[str]:
        """Get set of currently alive nodes"""
        return set(self.membership.keys())
```

**Use Cases**: Cassandra failure detection, Consul membership, Redis Cluster gossip

## Comparison: Consistency Models

| Model | Ordering Guarantee | Availability | Latency | Complexity | Use Cases |
|-------|-------------------|--------------|---------|------------|-----------|
| Strong Consistency | Total order, real-time | Low during partitions | High (coordination) | High | Financial transactions, inventory |
| Sequential Consistency | Program order per process | Medium | Medium | Medium | Collaborative editing |
| Causal Consistency | Causal relationships preserved | High | Low | High | Social feeds, comments |
| Eventual Consistency | No ordering guarantees | Very high | Very low | Low | User profiles, caches, DNS |

## Best Practices

**Design for Failure**: Assume any component can fail at any time. Build redundancy, implement health checks, and use timeouts liberally.

**Embrace Asynchrony**: Blocking operations limit scalability. Use message queues, async processing, and eventual consistency where appropriate.

**Idempotent Operations**: Design operations that can be safely retried. Network failures mean requests may be duplicated.

**Versioning and Conflict Resolution**: Use vector clocks or timestamps to detect conflicts. Implement application-specific merge logic for concurrent updates.

**Monitor and Observe**: Distributed systems are complex. Implement comprehensive logging, metrics, and distributed tracing to understand behavior.

**Start Simple**: Begin with stronger consistency and relax as needed. It's easier to weaken guarantees than to strengthen them after launch.

## Anti-Patterns

❌ **Distributed Transactions Everywhere**: Two-phase commit is expensive and fragile. Use sagas or eventual consistency instead.

❌ **Ignoring Network Partitions**: "It won't happen to us" is wishful thinking. Design for partition tolerance from day one.

❌ **Assuming Clocks Are Synchronized**: Clock skew causes subtle bugs. Use logical clocks for ordering, physical clocks only for TTLs.

❌ **Building Your Own Consensus**: Paxos and Raft are notoriously difficult to implement correctly. Use proven systems like etcd or ZooKeeper.

❌ **No Backpressure**: Slow consumers can overwhelm systems. Implement rate limiting, circuit breakers, and queue bounds.

## Real-World Examples

### Google Spanner

Provides strong consistency across global datacenters using TrueTime API (synchronized clocks with bounded uncertainty) and two-phase commit.

### Amazon DynamoDB

Eventually consistent by default with optional strong consistency. Uses consistent hashing for partitioning and vector clocks for conflict resolution.

### Apache Cassandra

Tunable consistency with quorum-based replication. Supports multiple consistency levels from ONE to ALL, allowing per-query trade-offs.

### etcd

Uses Raft consensus for strong consistency. Provides linearizable reads and writes with leader-based replication.

## Related Topics

- [CAP Theorem](05-cap-theorem.md) - Understanding consistency, availability, and partition tolerance trade-offs
- [Message Queues](06-message-queues.md) - Asynchronous communication patterns
- [Database Scaling Strategies](03-database-scaling-strategies.md) - Replication and sharding
- [High Availability & Fault Tolerance](12-high-availability-fault-tolerance.md) - Building resilient systems
