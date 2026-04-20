# ZooKeeper Request Handling and Leader-Follower Coordination Deep Dive

## Table of Contents
1. [Request Lifecycle Overview](#request-lifecycle-overview)
2. [Request Processing Pipeline](#request-processing-pipeline)
3. [Leader-Follower Coordination Protocol](#leader-follower-coordination-protocol)
4. [Message Types and Communication](#message-types-and-communication)
5. [State Synchronization](#state-synchronization)
6. [Error Handling and Recovery](#error-handling-and-recovery)
7. [Performance Optimizations](#performance-optimizations)

## Request Lifecycle Overview

Every ZooKeeper request goes through a well-defined lifecycle that ensures consistency across the cluster. The flow depends on whether it's a read or write operation and whether the server is a leader, follower, or observer.

### Request Types
- **Read Requests**: `getData`, `getChildren`, `exists`, `getACL`
- **Write Requests**: `create`, `delete`, `setData`, `setACL`
- **Administrative Requests**: `sync`, `reconfig`, session management

## Request Processing Pipeline

### 1. Request Object Structure

```java
// From Request.java lines 47-100
public class Request {
    public final long sessionId;     // Client session identifier
    public final int cxid;           // Client transaction ID
    public final int type;           // Operation type (OpCode)
    public final ServerCnxn cnxn;    // Connection to client
    public TxnHeader hdr;            // Transaction header (for writes)
    public Record txn;               // Transaction record
    public long zxid;                // ZooKeeper transaction ID
    // ... additional fields
}
```

### 2. Leader Request Processing Chain

For **write requests**, the leader uses this processing chain:

```
Client Request → PrepRequestProcessor → ProposalRequestProcessor → CommitProcessor → FinalRequestProcessor
```

#### **PrepRequestProcessor** (`PrepRequestProcessor.java`)
- **Purpose**: Validates requests and creates transaction headers
- **Key Functions**:
  - Assigns unique `zxid` (transaction ID)
  - Creates `TxnHeader` for state-changing operations
  - Validates paths, ACLs, and session state
  - Handles multi-operations atomically

```java
// From PrepRequestProcessor.java lines 84-90
/**
 * This request processor is generally at the start of a RequestProcessor
 * change. It sets up any transactions associated with requests that change the
 * state of the system. It counts on ZooKeeperServer to update
 * outstandingRequests, so that it can take into account transactions that are
 * in the queue to be applied when generating a transaction.
 */
```

#### **ProposalRequestProcessor** (`ProposalRequestProcessor.java`)
- **Purpose**: Coordinates with followers for write consensus
- **Key Functions**:
  - Calls `zks.getLeader().propose(request)` to broadcast proposals
  - Forwards to `SyncRequestProcessor` for logging
  - Manages the two-phase commit protocol

```java
// From ProposalRequestProcessor.java lines 68-92
public void processRequest(Request request) throws RequestProcessorException {
    if (request instanceof LearnerSyncRequest) {
        zks.getLeader().processSync((LearnerSyncRequest) request);
    } else {
        if (shouldForwardToNextProcessor(request)) {
            nextProcessor.processRequest(request);
        }
        if (request.getHdr() != null) {
            // We need to sync and get consensus on any transactions
            try {
                zks.getLeader().propose(request);
            } catch (XidRolloverException e) {
                throw new RequestProcessorException(e.getMessage(), e);
            }
            syncProcessor.processRequest(request);
        }
    }
}
```

#### **CommitProcessor** (`CommitProcessor.java`)
- **Purpose**: Ensures proper ordering of committed transactions
- **Key Functions**:
  - Matches local proposals with committed transactions from leader
  - Maintains session-based ordering
  - Processes read requests while waiting for write commits

```java
// From CommitProcessor.java lines 44-76
/**
 * This RequestProcessor matches the incoming committed requests with the
 * locally submitted requests. The trick is that locally submitted requests that
 * change the state of the system will come back as incoming committed requests,
 * so we need to match them up. Instead of just waiting for the committed requests,
 * we process the uncommitted requests that belong to other sessions.
 *
 * Multi-threading constraints:
 *   - Each session's requests must be processed in order.
 *   - Write requests must be processed in zxid order
 *   - Must ensure no race condition between writes in one session that would
 *     trigger a watch being set by a read request in another session
 */
```

### 3. Follower Request Processing Chain

For **read requests**, followers use:
```
Client Request → FinalRequestProcessor
```

For **write requests**, followers forward to leader:
```
Client Request → FollowerRequestProcessor → Forward to Leader
```

## Leader-Follower Coordination Protocol

### 1. ZAB Two-Phase Commit Protocol

#### **Phase 1: Proposal** (`Leader.java:1295`)

```java
public Proposal propose(Request request) throws XidRolloverException {
    // Create QuorumPacket with PROPOSAL type
    byte[] data = request.getSerializeData();
    QuorumPacket pp = new QuorumPacket(Leader.PROPOSAL, request.zxid, data, null);

    Proposal p = new Proposal(request, pp);

    // Send to all followers
    sendPacket(pp);

    return p;
}
```

**What happens when leader proposes:**
1. **Leader creates proposal** with unique `zxid`
2. **Broadcasts PROPOSAL** message to all followers/observers
3. **Logs proposal locally** to transaction log
4. **Waits for quorum acknowledgments**

#### **Phase 2: Acknowledgment and Commit**

**Follower acknowledgment** (`Follower.java:156-198`):
```java
protected void processPacket(QuorumPacket qp) throws Exception {
    switch (qp.getType()) {
    case Leader.PROPOSAL:
        // Deserialize transaction
        TxnLogEntry logEntry = SerializeUtils.deserializeTxn(qp.getData());

        // Log the proposal locally
        fzk.logRequest(logEntry.toRequest());

        // Send ACK back to leader (implicit in logging process)
        break;
    case Leader.COMMIT:
        // Apply the committed transaction
        fzk.commit(qp.getZxid());
        break;
    }
}
```

**Leader processes ACKs** (`Leader.java:1054`):
```java
public synchronized void processAck(long sid, long zxid, SocketAddress followerAddr) {
    Proposal p = outstandingProposals.get(zxid);
    if (p == null) {
        LOG.warn("Trying to commit future proposal: zxid 0x{}", Long.toHexString(zxid));
        return;
    }

    p.addAck(sid);  // Add this follower's acknowledgment

    // Check if we have quorum
    boolean hasCommitted = tryToCommit(p, zxid, followerAddr);

    if (hasCommitted) {
        // Send COMMIT to all followers
        commit(zxid);
    }
}
```

### 2. Message Flow Diagram

```
Leader                                    Follower
  |                                         |
  | -------- PROPOSAL(zxid, txn) --------→ |
  |                                         | (logs locally)
  | ←-------- ACK(zxid) ------------------- |
  |                                         |
  | (waits for quorum)                      |
  |                                         |
  | -------- COMMIT(zxid) ---------------→ |
  |                                         | (applies to DataTree)
  | (applies locally)                       |
  |                                         |
```

### 3. LearnerHandler: Per-Follower Communication

Each follower has a dedicated `LearnerHandler` thread on the leader:

```java
// From LearnerHandler.java lines 61-66
/**
 * There will be an instance of this class created by the Leader for each
 * learner. All communication with a learner is handled by this
 * class.
 */
public class LearnerHandler extends ZooKeeperThread {
    protected final Socket sock;
    final LinkedBlockingQueue<QuorumPacket> queuedPackets = new LinkedBlockingQueue<>();
    // ...
}
```

**Key responsibilities:**
- **Packet queuing**: Manages outbound messages to follower
- **Heartbeat monitoring**: Tracks follower liveness
- **Synchronization**: Handles initial sync when follower connects
- **Flow control**: Manages packet queue size and backpressure

## Message Types and Communication

### Core ZAB Message Types

```java
// From Leader.java - Message type constants
static final int REQUEST = 1;        // Client request to leader
public static final int PROPOSAL = 2; // Leader proposes transaction
static final int ACK = 3;            // Follower acknowledges proposal
static final int COMMIT = 4;         // Leader commits transaction
static final int PING = 5;           // Heartbeat messages
public static final int REVALIDATE = 6; // Session revalidation
public static final int SYNC = 7;    // Sync operation
```

### Discovery and Synchronization Messages

```java
public static final int FOLLOWERINFO = 11;  // Follower registers with leader
public static final int LEADERINFO = 17;    // Leader responds with info
public static final int ACKEPOCH = 18;      // Follower acknowledges epoch
public static final int NEWEPOCH = 12;      // Leader announces new epoch
public static final int NEWLEADER = 10;     // Leader ready to serve
```

## State Synchronization

### 1. Follower Connection and Sync (`Follower.java:71`)

When a follower connects to a leader:

```java
void followLeader() throws InterruptedException {
    // Phase 1: Discovery
    self.setZabState(QuorumPeer.ZabState.DISCOVERY);
    QuorumServer leaderServer = findLeader();
    connectToLeader(leaderServer.addr, leaderServer.hostname);

    // Phase 2: Registration and Epoch Agreement
    long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);

    // Phase 3: Synchronization
    self.setZabState(QuorumPeer.ZabState.SYNCHRONIZATION);
    syncWithLeader(newEpochZxid);

    // Phase 4: Active Broadcast
    self.setZabState(QuorumPeer.ZabState.BROADCAST);

    // Main packet processing loop
    QuorumPacket qp = new QuorumPacket();
    while (this.isRunning()) {
        readPacket(qp);
        processPacket(qp);
    }
}
```

### 2. Synchronization Types

The leader synchronizes followers using different strategies:

1. **DIFF**: Send missing transactions since follower's last zxid
2. **TRUNC**: Follower has transactions leader doesn't - truncate and resync
3. **SNAP**: Send complete snapshot if diff is too large

### 3. Transaction Log Synchronization

```java
// Leader determines sync type based on follower's zxid
long peerLastZxid = qp.getZxid();
long minCommittedLog = zkDb.getminCommittedLog();
long maxCommittedLog = zkDb.getmaxCommittedLog();

if (peerLastZxid > maxCommittedLog) {
    // Follower is ahead - send TRUNC
} else if (peerLastZxid >= minCommittedLog) {
    // Send DIFF with missing transactions
} else {
    // Send SNAP - complete snapshot
}
```

## Error Handling and Recovery

### 1. Leader Failure Detection

**Heartbeat mechanism:**
- Leader sends `PING` messages periodically
- Followers respond with `PING` responses
- Missing pings trigger connection closure

**Quorum loss handling:**
```java
// From Leader.java - quorum monitoring
if (!syncedAckSet.hasAllQuorums()) {
    // Lost quorum - shutdown and trigger re-election
    shutdownMessage = "Not sufficient followers synced";
    break;
}
```

### 2. Follower Failure Recovery

When follower reconnects:
1. **Authentication**: Verify server credentials
2. **Epoch validation**: Ensure follower accepts current epoch
3. **State sync**: Bring follower up to current state
4. **Resume operation**: Add back to active quorum

### 3. Network Partition Handling

**Split-brain prevention:**
- Only majority partitions can process writes
- Minority partitions become read-only or unavailable
- FLE (Fast Leader Election) ensures single leader per epoch

## Performance Optimizations

### 1. Asynchronous Communication

```java
// From Learner.java - async packet sending
void writePacket(QuorumPacket pp, boolean flush) throws IOException {
    if (asyncSending) {
        sender.queuePacket(pp);  // Non-blocking queue
    } else {
        writePacketNow(pp, flush);  // Synchronous write
    }
}
```

### 2. Batch Processing

**CommitProcessor optimizations:**
- Configurable worker thread pools
- Batched read processing
- Session-based request ordering

### 3. Pipeline Parallelism

**Multi-threaded request processing:**
- Separate threads for different pipeline stages
- Concurrent read request processing
- Pipelined proposal acknowledgments

### 4. Flow Control

**Queue management:**
```java
// From LearnerHandler.java - packet queue monitoring
final LinkedBlockingQueue<QuorumPacket> queuedPackets = new LinkedBlockingQueue<>();
private final AtomicLong queuedPacketsSize = new AtomicLong();

// Monitor queue size for backpressure
if (queuedPacketsSize.get() > maxQueueSize) {
    // Apply flow control
}
```

## State Machine and Consistency Guarantees

### 1. ZAB State Transitions

```
ELECTION → DISCOVERY → SYNCHRONIZATION → BROADCAST
    ↑                                          ↓
    ←----------------ELECTION←-----------------
```

### 2. Consistency Properties

**From ZooKeeper internals documentation:**

1. **Sequential Consistency**: Operations from each client are executed in order
2. **Monotonic Read Consistency**: Read operations never go backwards in time
3. **Write Linearizability**: All write operations appear to happen atomically
4. **Read Your Writes**: A client always sees its own writes

### 3. Session Guarantees

**Session-level ordering:**
- All operations in a session are ordered
- Session timeouts ensure cleanup of ephemeral nodes
- Session migration during failover maintains consistency

## Implementation Details and Code References

### Key Files and Their Roles

1. **`Request.java`**: Request lifecycle and structure
2. **`PrepRequestProcessor.java`**: Request validation and transaction creation
3. **`ProposalRequestProcessor.java`**: Leader proposal coordination
4. **`CommitProcessor.java`**: Transaction ordering and commit logic
5. **`Leader.java`**: Leader-specific logic and proposal management
6. **`Follower.java`**: Follower behavior and packet processing
7. **`LearnerHandler.java`**: Per-follower communication management
8. **`Learner.java`**: Common learner (follower/observer) functionality

### Critical Code Paths

**Write Request Flow:**
1. `ZooKeeperServer.processPacket()` - Initial request handling
2. `PrepRequestProcessor.run()` - Transaction preparation
3. `Leader.propose()` - Proposal broadcast
4. `Follower.processPacket()` - Proposal handling
5. `Leader.processAck()` - Acknowledgment processing
6. `Leader.commit()` - Transaction commit
7. `FinalRequestProcessor.processRequest()` - Response to client

**Read Request Flow:**
1. `ZooKeeperServer.processPacket()` - Request validation
2. `FinalRequestProcessor.processRequest()` - Direct DataTree access
3. Response sent to client

## Conclusion

ZooKeeper's request handling and leader-follower coordination represent a sophisticated implementation of the ZAB protocol. The system achieves high performance through:

- **Pipelined request processing** with specialized processors
- **Asynchronous communication** between leader and followers
- **Efficient state synchronization** with multiple sync strategies
- **Robust failure detection and recovery** mechanisms
- **Strong consistency guarantees** while optimizing for read performance

The careful separation of concerns across different processor stages, combined with the ZAB protocol's phase-based approach, ensures that ZooKeeper maintains consistency even under network partitions and server failures while providing the performance characteristics needed for a coordination service.