# ZooKeeper Architecture and ZAB Protocol

## Overview

Apache ZooKeeper is a distributed coordination service that provides a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. At its core, ZooKeeper uses the **ZAB (ZooKeeper Atomic Broadcast)** protocol to ensure consistency across its distributed ensemble.

## ZooKeeper Architecture

### Core Components

#### 1. **ZooKeeper Server** (`ZooKeeperServer.java`)
The main server class that handles client connections and request processing. It maintains:
- **DataTree**: An in-memory hierarchical data structure storing znodes
- **Session Management**: Tracking client sessions and their expiration
- **Request Processing Chain**: A pipeline of processors that handle different types of requests

#### 2. **Quorum Architecture** (`QuorumPeer.java`)
ZooKeeper operates in three modes:
- **Leader**: Processes write requests and coordinates state replication
- **Follower**: Processes read requests and forwards writes to the leader
- **Observer**: Like followers but don't participate in quorum decisions

```java
// From QuorumPeer.java lines 93-102
/**
 * This class manages the quorum protocol. There are three states this server
 * can be in:
 * <ol>
 * <li>Leader election - each server will elect a leader (proposing itself as a
 * leader initially).</li>
 * <li>Follower - the server will synchronize with the leader and replicate any
 * transactions.</li>
 * <li>Leader - the server will process requests and forward them to followers.
 * A majority of followers must log the request before it can be accepted.
 * </ol>
 */
```

#### 3. **Data Structure: DataTree** (`DataTree.java`)
ZooKeeper maintains a hierarchical namespace similar to a file system:
- **ZNodes**: Data nodes that can store data and have children
- **Persistent vs Ephemeral**: Persistent znodes survive client disconnections; ephemeral ones are deleted when the session ends
- **Sequential**: Znodes can be created with monotonically increasing sequence numbers

### Request Processing Pipeline

#### For Write Requests (Leader):
1. **PrepRequestProcessor** → Prepares the request and assigns a transaction ID (zxid)
2. **ProposalRequestProcessor** → Sends proposals to followers
3. **CommitProcessor** → Waits for quorum acknowledgment before committing
4. **FinalRequestProcessor** → Applies the transaction and responds to client

#### For Read Requests:
Reads can be served by any server from local state, providing eventual consistency.

## ZAB (ZooKeeper Atomic Broadcast) Protocol

ZAB is the atomic broadcast protocol that ensures all changes are propagated consistently across the ZooKeeper ensemble. It operates in phases:

### Phase Structure

#### **Phase 0: Leader Election** (`FastLeaderElection.java`)
- Servers elect a leader using a voting protocol
- The server with the highest zxid (transaction ID) typically becomes leader
- Requires majority consensus

#### **Phase 1: Discovery**
- New leader establishes its epoch and gathers follower states
- Leader determines what transactions to commit or discard
- Synchronizes with followers to ensure consistent starting state

#### **Phase 2: Synchronization**
- Leader sends `NEWLEADER` proposal to all followers
- Followers acknowledge when they've synchronized their state
- Leader commits the `NEWLEADER` proposal when quorum acknowledges

#### **Phase 3: Broadcast** (Active Messaging)
- Leader accepts write requests from clients
- Implements two-phase commit protocol:
  1. **Propose**: Leader sends `PROPOSE` messages to followers
  2. **Commit**: After quorum acknowledgment, leader sends `COMMIT` messages

### Key ZAB Properties

From the ZooKeeper internals documentation:

```
* Reliable delivery: If a message m is delivered by one server,
  message m will be eventually delivered by all servers.

* Total order: If a message a is delivered before message b by one server,
  message a will be delivered before b by all servers.

* Causal order: If a message b is sent after a message a has been delivered
  by the sender of b, message a must be ordered before b.
```

### Transaction IDs (zxid)
ZAB uses 64-bit zxids composed of:
- **High 32 bits**: Epoch number (changes with each new leader)
- **Low 32 bits**: Counter (monotonically increasing within epoch)

This ensures total ordering: `(epoch1, counter1) < (epoch2, counter2)` if `epoch1 < epoch2` or `(epoch1 == epoch2 && counter1 < counter2)`

### Commit Protocol Flow
```java
// From CommitProcessor.java lines 44-48
/**
 * This RequestProcessor matches the incoming committed requests with the
 * locally submitted requests. The trick is that locally submitted requests that
 * change the state of the system will come back as incoming committed requests,
 * so we need to match them up.
 */
```

## ZAB vs Raft Protocol Comparison

### Similarities

1. **Leader-based consensus**: Both use a leader to coordinate state replication
2. **Majority quorums**: Both require majority agreement for decisions
3. **Log replication**: Both maintain replicated logs of operations
4. **Leader election**: Both include mechanisms to elect new leaders

### Key Differences

| Aspect | ZAB | Raft |
|--------|-----|------|
| **Primary Use Case** | Coordination service, metadata management | General-purpose replicated state machines |
| **Read Semantics** | Reads from any server (eventual consistency) | Linearizable reads require leader contact or read index |
| **Phase Structure** | 4 explicit phases: Election → Discovery → Synchronization → Broadcast | 2 main states: Leader Election → Normal Operation |
| **Transaction Ordering** | Global ordering using epoch + counter (zxid) | Log index + term |
| **Synchronization** | Explicit Discovery & Synchronization phases for new leader | Leader sends heartbeats and overwrites inconsistent entries |

### Detailed Differences

#### **1. Consistency Guarantees**

**ZAB**:
- Write linearizability (writes appear to happen atomically)
- Read your writes (if you read after write from same client)
- Monotonic read consistency (within same session)
- Provides "ordered sequential consistency"

**Raft**:
- Full linearizability for both reads and writes (when properly implemented)
- All operations must go through leader for linearizability

#### **2. Leader Recovery Process**

**ZAB**:
```
1. Discovery Phase:
   - New leader gathers information about follower states
   - Determines which transactions to keep/discard

2. Synchronization Phase:
   - Leader synchronizes all followers to consistent state
   - NEWLEADER proposal must be committed before normal operations
```

**Raft**:
```
- Leader immediately starts sending heartbeats
- Overwrites follower log entries that conflict with leader's log
- No separate synchronization phase
```

#### **3. Transaction Processing**

**ZAB Two-Phase Protocol**:
```
Leader → PROPOSE(zxid, txn) → Followers
       ← ACK(zxid)         ← Followers (wait for quorum)
Leader → COMMIT(zxid)     → Followers
```

**Raft Log Replication**:
```
Leader → AppendEntries(term, index, entry) → Followers
       ← Success/Failure                   ← Followers
(Commit when replicated on majority, no separate commit message)
```

#### **4. Read Handling**

**ZAB**:
- Reads served by any server from local state
- May see stale data but within session consistency guarantees
- Faster reads but weaker consistency

**Raft**:
- Linearizable reads require leader interaction or read index mechanism
- Can implement lease-based reads for performance
- Stronger consistency but potentially higher latency

#### **5. Membership Changes**

**ZAB**:
- Dynamic reconfiguration supported through special transactions
- Complex process involving configuration versioning

**Raft**:
- Joint consensus for configuration changes
- Cleaner theoretical foundation for membership changes

### Performance Implications

**ZAB Advantages**:
- Better read performance (local reads)
- Lower read latency
- Better suited for read-heavy workloads

**Raft Advantages**:
- Simpler protocol with cleaner semantics
- Better write performance in some scenarios (no separate commit phase)
- Easier to reason about and implement correctly

### Use Case Suitability

**ZAB is better for**:
- Configuration management
- Service discovery
- Coordination primitives (locks, barriers)
- Scenarios requiring high read performance with relaxed consistency

**Raft is better for**:
- Replicated databases
- Strongly consistent key-value stores
- Applications requiring full linearizability
- General-purpose distributed systems

## Implementation Details in ZooKeeper

### Key Files and Their Roles

1. **`QuorumPeer.java`**: Main quorum management and state machine
2. **`Leader.java`**: Leader-specific logic for proposal management
3. **`Follower.java`**: Follower-specific logic for proposal handling
4. **`FastLeaderElection.java`**: Leader election algorithm
5. **`CommitProcessor.java`**: Manages commit ordering and processing
6. **`DataTree.java`**: In-memory data structure for znodes

### ZAB State Machine

```java
// Simplified state transitions from QuorumPeer
LOOKING → LEADING/FOLLOWING → LOOKING (on failure)
```

With ZAB states:
```
ELECTION → DISCOVERY → SYNCHRONIZATION → BROADCAST
```

## Conclusion

ZAB and Raft are both sophisticated consensus protocols, but they're optimized for different use cases. ZAB's design reflects ZooKeeper's specific requirements as a coordination service where read performance and session consistency matter more than strict linearizability. Raft provides a more general-purpose foundation for building consistent distributed systems where strong consistency is paramount.

The choice between them depends on your specific requirements:
- **Choose ZAB/ZooKeeper** for coordination services requiring high read performance
- **Choose Raft** for general-purpose distributed systems requiring strong consistency

Both protocols have proven themselves in production environments and continue to evolve based on real-world experience and formal verification efforts.