--- 
layout: post
title:  "Distributed System Consensus Algorithm - Deep Dive"
summary: "Distributed System Consensus Algorithm - Deep Dive"
author: anachary
date: '2024-11-30 14:35:23 +0530'
category: "cloud"
thumbnail: /assets/img/posts/distributed-sys-algo.png
keywords: logical programmer,consensus,paxos,raft,zab,Gossip
permalink: /blog/2024-11-30-distributed-consensus-algorithm-comparisons/
usemathjax: true
---

# The Origin: A Simple Problem Becomes a Complex Journey

It started innocently enough. As a senior software engineer, I was tasked with ensuring a singleton process in a containerized environment. "How hard could this be?" I thought, with the confident swagger of someone who's solved countless distributed system challenges.

Little did I know, this seemingly straightforward problem would lead me down a fascinating rabbit hole of distributed consensus algorithms, challenging everything I thought I knew about distributed systems.

### The Kubernetes Temptation

Kubernetes offered a built-in leader election mechanism, but the engineer in me wanted to understand the underlying principles. Why rely on a black box when I could potentially create a custom solution?

## Consensus Algorithms: A Comprehensive Exploration (C# custom implementation of each algorithm and workflow)

Here’s what I learned about **Paxos**, **Multi-Paxos**, **Raft**, **Zab**, and **Gossip Protocol**, including deeper insights and small C# implementations of their principles.

## Comparative Analysis of Consensus Algorithms

|&nbsp;**Algorithm**&nbsp;|&nbsp;**Complexity**&nbsp;|&nbsp;**Performance**&nbsp;|&nbsp;**Consistency**&nbsp;|&nbsp;**Real-World Examples**&nbsp;|
|---------------|------------------|-----------------|---------------------|----------------------------------------------------------------------------------------------------------------|
| **Paxos**     | 🔴 High          | 🟠 Med           | ✅ Strong           | [Google Chubby Lock Service](https://research.google/pubs/pub27897/), [Spanner Database](https://cloud.google.com/spanner) |
| **Multi-Paxos** | 🟠 Med        | 🟢 High         | ✅ Strong           | [Google's Replicated Log](https://raft.github.io/), [ZooKeeper Coordination](https://zookeeper.apache.org/)    |
| **Raft**      | 🟠 Med          | 🟢 High         | ✅ Strong           | [etcd](https://etcd.io/) (Kubernetes management), [Consul](https://www.consul.io/)                             |
| **Zab**       | 🟠 Med          | 🟢 High         | ✅ Strong           | [Apache ZooKeeper](https://zookeeper.apache.org/), [HDFS](https://hadoop.apache.org/)                          |
| **Gossip Protocol** | 🟢 Low   | 🟢 High         | ⏳ Eventually Consistent | [Apache Cassandra](https://cassandra.apache.org/), [Amazon DynamoDB](https://aws.amazon.com/dynamodb/), [Bitcoin](https://bitcoin.org/) |

---

### 1. Paxos: The Cornerstone of Consensus

#### What is Paxos?

Paxos, developed by Leslie Lamport in 1989, is the foundational algorithm for distributed consensus. It ensures safety (no two nodes disagree on the chosen value) and liveness (progress continues if a majority of nodes are functioning).

#### Detailed Workflow

1. **Prepare Phase**:
   - A proposer sends a proposal number (`n`) to acceptors.
   - Acceptors promise not to accept proposals with lower numbers.
2. **Accept Phase**:
   - The proposer sends the value associated with `n`.
   - Acceptors accept if it matches their promise.
3. **Learn Phase**:
   - Once a majority accepts, the value is chosen, and learners are updated.

#### Challenges

- **Complexity**: Paxos is notoriously difficult to implement correctly.
- **High Overhead**: Requires multiple message exchanges for each decision.

#### Use Cases

Ideal for systems that demand high consistency, like databases or coordination services.

#### C# Simple Implementation of Paxos 
```csharp
public class PaxosNode
{
    public string NodeId { get; private set; }
    public int PromisedProposal { get; private set; } = -1; // No proposal promised initially
    public (int Proposal, string Value)? AcceptedProposal { get; private set; } = null;

    public PaxosNode(string nodeId)
    {
        NodeId = nodeId;
    }

    // Prepare phase: Promise not to accept proposals with a lower number
    public bool Prepare(int proposalNumber)
    {
        if (proposalNumber > PromisedProposal)
        {
            PromisedProposal = proposalNumber;
            return true; // Node promises to accept proposals with higher numbers
        }
        return false; // Node rejects proposals with lower numbers
    }

    // Accept phase: Accept the proposal if it matches the promised number
    public bool Accept(int proposalNumber, string value)
    {
        if (proposalNumber >= PromisedProposal)
        {
            PromisedProposal = proposalNumber;
            AcceptedProposal = (proposalNumber, value);
            return true;
        }
        return false;
    }
}

public class PaxosAlgorithm
{
    private List<PaxosNode> nodes;
    private int majority;
    private int proposalNumber = 0;

    public PaxosAlgorithm(List<PaxosNode> clusterNodes)
    {
        nodes = clusterNodes;
        majority = nodes.Count / 2 + 1; // Majority needed to reach consensus
    }

    // Step 1: Prepare phase where nodes promise not to accept lower proposals
    public bool PreparePhase(int proposalNumber)
    {
        int prepareAcks = 0;

        foreach (var node in nodes)
        {
            if (node.Prepare(proposalNumber))
            {
                prepareAcks++;
            }
        }

        return prepareAcks >= majority; // Consensus reached if majority promises
    }

    // Step 2: Accept phase where nodes accept the proposal
    public bool AcceptPhase(int proposalNumber, string value)
    {
        int acceptAcks = 0;

        foreach (var node in nodes)
        {
            if (node.Accept(proposalNumber, value))
            {
                acceptAcks++;
            }
        }

        return acceptAcks >= majority; // Consensus reached if majority accepts
    }

    // Step 3: Learn phase where the consensus value is learned
    public void LearnPhase(string value)
    {
        Console.WriteLine($"Consensus reached on value: {value}");
    }

    // Run the complete Paxos protocol (Prepare -> Accept -> Learn)
    public void RunPaxos(string proposedValue)
    {
        proposalNumber++;

        // Prepare phase
        if (!PreparePhase(proposalNumber))
        {
            Console.WriteLine("Prepare phase failed: insufficient promises.");
            return;
        }

        // Accept phase
        if (!AcceptPhase(proposalNumber, proposedValue))
        {
            Console.WriteLine("Accept phase failed: insufficient accepts.");
            return;
        }

        // Learn phase: Consensus is reached
        LearnPhase(proposedValue);
    }
}

public class Program
{
    public static void Main(string[] args)
    {
        // Initialize a cluster of Paxos nodes
        var nodes = new List<PaxosNode>
        {
            new PaxosNode("Node1"),
            new PaxosNode("Node2"),
            new PaxosNode("Node3")
        };

        // Instantiate Paxos protocol with the list of nodes
        var paxos = new PaxosAlgorithm(nodes);

        // Propose a value
        Console.WriteLine("Proposing value: ConsensusValue1");
        paxos.RunPaxos("ConsensusValue1");

        // Simulate a second proposal (different value)
        Console.WriteLine("\nProposing value: ConsensusValue2");
        paxos.RunPaxos("ConsensusValue2");
    }
}
```
---

### 2. Multi-Paxos: Optimized Consensus

#### What is Multi-Paxos?

Multi-Paxos optimizes Paxos by introducing a long-lived leader, streamlining sequential decisions by avoiding repeated leader elections.

#### Detailed Workflow

1. **Leader Election**:
   - A leader is elected for a series of decisions.
2. **Proposal and Log Replication**:
   - The leader handles all proposals and replicates logs to followers.
3. **Commit Phase**:
   - Values are committed once acknowledged by a majority.

#### Advantages

- **Efficiency**: Reduced message complexity for consecutive operations.
- **Pipelining**: Supports concurrent proposals.

#### Use Cases

Widely used in distributed logging and storage systems, like Google’s Chubby.

#### C# Simple Implementation of Multi-Paxos

```csharp
public class MultiPaxosNode
{
    public string NodeId { get; private set; }
    public int CurrentTerm { get; set; } = 0;  // Keeps track of the current term of the node
    public bool IsLeader { get; set; } = false;  // Flag to determine if the node is the leader
    public List<string> Log { get; private set; } = new List<string>();  // Stores the log entries

    public MultiPaxosNode(string nodeId)
    {
        NodeId = nodeId;
    }

    // Request vote from the node to become the leader
    public bool RequestVote(int term)
    {
        if (term > CurrentTerm)
        {
            CurrentTerm = term;
            IsLeader = false;  // Ensure only one leader at a time
            return true;
        }
        return false;
    }

    // Append a log entry as the leader
    public void AppendLog(string logEntry)
    {
        if (IsLeader)
        {
            Log.Add(logEntry);
            Console.WriteLine($"{NodeId} added log entry: {logEntry}");
        }
    }

    // Commit the log entry once a majority has accepted it
    public void CommitLog(int majorityCount)
    {
        if (Log.Count >= majorityCount)
        {
            Console.WriteLine($"{NodeId} commits log entry: {Log.Last()}");
        }
    }
}

public class MultiPaxosAlgorithm
{
    private List<MultiPaxosNode> nodes;
    private int majority;

    public MultiPaxosAlgorithm(List<MultiPaxosNode> clusterNodes)
    {
        nodes = clusterNodes;
        majority = nodes.Count / 2 + 1;  // Majority threshold for committing logs
    }

    // Simulate leader election by having nodes request votes from each other
    public void ElectLeader()
    {
        int highestTerm = 0;
        MultiPaxosNode leader = null;

        // Simulate the leader election process by having nodes request votes
        foreach (var node in nodes)
        {
            if (node.RequestVote(highestTerm + 1))
            {
                highestTerm = node.CurrentTerm;
                leader = node;
            }
        }

        // Once a leader is elected, the leader can append logs
        if (leader != null)
        {
            leader.IsLeader = true;
            Console.WriteLine($"{leader.NodeId} is elected as leader (Term: {leader.CurrentTerm})");
        }
    }

    // Propose a value (the leader appends it to the log)
    public void ProposeValue(string value)
    {
        // The leader appends the proposed value to the log
        var leader = nodes.FirstOrDefault(node => node.IsLeader);
        if (leader != null)
        {
            leader.AppendLog(value);
            leader.CommitLog(majority);
        }
        else
        {
            Console.WriteLine("No leader elected yet.");
        }
    }
}

public class Program
{
    public static void Main(string[] args)
    {
        // Initialize a few nodes in the Paxos cluster
        var nodes = new List<MultiPaxosNode>
        {
            new MultiPaxosNode("Node1"),
            new MultiPaxosNode("Node2"),
            new MultiPaxosNode("Node3")
        };

        // Instantiate MultiPaxos protocol with the list of nodes
        var multiPaxosAlgorithm = new MultiPaxosAlgorithm(nodes);

        // Elect a leader
        multiPaxosAlgorithm.ElectLeader();

        // Propose a value
        multiPaxosAlgorithm.ProposeValue("ConsensusValue1");

        // Simulate another round of leader election
        Console.WriteLine("\nSimulating leader election again...");
        multiPaxosAlgorithm.ElectLeader();

        // Propose another value
        multiPaxosAlgorithm.ProposeValue("ConsensusValue2");
    }
}
```
---

### 3. Raft: Simplicity in Consensus

#### What is Raft?

Raft simplifies consensus with clear roles: leader, follower, and candidate. The leader handles log replication and state synchronization.

#### Key Features

- **Leader-Based Coordination**:
  - One leader manages log replication and state updates.
- **State Machine Replication**:
  - Ensures consistent state across all nodes.

#### Step-by-Step Workflow

The Raft consensus algorithm ensures that distributed systems maintain consistency and fault tolerance. Raft works by electing a leader, replicating logs, and committing entries to ensure that all nodes agree on the same sequence of operations. Below is the step-by-step workflow of the Raft algorithm.


##### **1. Initialization: Nodes Start as Followers**

- When the Raft cluster starts, **all nodes** are in the **Follower** state.
- **No leader** exists at the beginning, and nodes are waiting for a leader to be elected.
- Followers stay in this state until they hear from a leader. If they do not hear from a leader within an **election timeout**, they transition to the **Candidate** state and begin an election.


##### **2. Leader Election (Candidate Phase)**

- **Election Timeout**: When a follower does not hear from the leader within a specified timeout period, it **becomes a candidate** and starts a new election.
  - The **election timeout** is randomly chosen within a fixed range to prevent all followers from becoming candidates at the same time.

- **Requesting Votes**: The candidate increments its **term** number and sends a **RequestVote** message to all other nodes, asking for their vote.

- **Voting Process**:
  - Each node (follower) can vote for only **one candidate per term**.
  - A node votes for a candidate if it hasn't already voted in the current term.
  - If a candidate receives a **majority of votes**, it becomes the **leader** for the current term.
  
- **Leader Election Result**:
  - If the candidate gets the majority of votes, it **becomes the leader** for that term.
  - The new leader then starts replicating logs and processing client requests.


##### **3. Leader Announcement**

- Once a node becomes the leader, it starts sending **heartbeat messages** (empty **AppendEntries RPCs**) to followers to assert its leadership and keep them from starting another election.
- The leader handles all client requests and ensures that the system state remains consistent.

##### **4. Log Replication**

Once the leader is elected, it is responsible for handling **log replication**:

- **Client Request**: Clients send commands (e.g., write operations) to the leader.
  - The leader appends the received command to its log.
  
- **Replication to Followers**: The leader sends **AppendEntries RPCs** to the followers, containing the new log entry.
  
- **Follower Replication**: Each follower appends the log entry to its log once it receives the RPC. The follower sends an **acknowledgement** back to the leader.

- **Commitment**:
  - A log entry is **committed** once it has been replicated on a majority of the nodes.
  - Once a log entry is committed, it is **applied** to the system state and acknowledged back to the client.


##### **5. Heartbeat: Maintaining Leadership**

- The leader sends periodic **AppendEntries RPCs** (empty heartbeats) to its followers.
- These heartbeats serve two purposes:
  1. To **maintain authority** and ensure followers know who the leader is.
  2. To **prevent followers** from starting elections by resetting their election timeout.
  
- If a follower doesn't receive a heartbeat within the timeout period, it transitions to the **candidate** state and starts a new election.


##### **6. Leader Failure & Re-election**

- If the leader **fails** (e.g., crashes or becomes unresponsive), the system detects this when followers don't receive heartbeats within the election timeout.
  
- **Follower Transition to Candidate**:
  - If a follower does not hear from the leader for the election timeout, it transitions to the **Candidate** state and starts a new election.
  
- **Re-election Process**:
  - The candidate sends **RequestVote RPCs** to other nodes to solicit votes.
  - If the candidate receives a majority of votes, it becomes the new leader.
  
- **Log Consistency**:
  - Once a new leader is elected, it ensures **log consistency** across the cluster, even if previous leaders have failed or logs are inconsistent.


##### **7. Log Consistency & Safety**

Raft ensures that the logs remain consistent and that once a log entry is committed, it is guaranteed to be applied in the same order across all nodes:

- **Leader Log**: The leader’s log is **always replicated** to followers in the same order.
- **Log Matching**: Before a log entry is committed, Raft ensures that the logs of followers match the leader’s log for that term.
  - If there is a discrepancy (e.g., a follower has an outdated log), the leader forces the follower to **rollback** its log entries and sync with the leader’s log.

- **Commitment of Entries**:
  - A log entry is only considered committed once a **majority of nodes** (including the leader) have written the entry to their logs.


##### **8. Log Application**

Once a log entry is committed:

- **Applying Logs**: The leader and followers apply the committed log entry to their **state machine** (the actual system that processes the command).
- **Consistency Across Nodes**: Raft ensures that all nodes apply the log entries in the same order, thus maintaining consistency.

##### **Raft Workflow Diagram**

```plaintext
[Follower]        [Leader]       [Follower]      [Follower]
   |                 |                |                 |
   |-- Election ---->|                |                 |
   |                 |                |                 |
   |                 |---- Append --> |---- Append -->  | 
   |                 |     Log        |     Log         |
   |                 |                |                 |
   |                 |<-- Commit ---- |<-- Commit ----  |
   |                 |                |                 |
   |<--- Hearbeat ---|<--- Hearbeat --|<--- Hearbeat ---|

```

#### Use Cases

Powers tools like Kubernetes, etcd, and Consul for cluster management.

#### C# Simple Implementation of Raft

```csharp
public class RaftNode
{
    public string NodeId { get; private set; }
    public int CurrentTerm { get; set; } = 0;    // Term is a unique identifier for a leadership cycle
    public bool IsLeader { get; set; } = false;  // Whether this node is the leader
    public List<string> Log { get; private set; } = new List<string>();  // Log entries of the node
    public bool HasVoted { get; set; } = false;  // Tracks if the node has voted in the current term

    public RaftNode(string nodeId)
    {
        NodeId = nodeId;
    }

    // Request vote from other nodes during leader election
    public bool RequestVote(int term)
    {
        if (term > CurrentTerm)
        {
            CurrentTerm = term;
            HasVoted = true;
            IsLeader = false;  // If the node receives a vote, it is no longer a leader
            return true;
        }
        return false;
    }

    // Append a log entry as a leader
    public void AppendLog(string logEntry)
    {
        if (IsLeader)
        {
            Log.Add(logEntry);
            Console.WriteLine($"{NodeId} (Leader) appended log entry: {logEntry}");
        }
    }

    // Commit the log entry once a majority has accepted it
    public void CommitLog(int majority)
    {
        if (Log.Count >= majority)
        {
            Console.WriteLine($"{NodeId} (Leader) commits log entry: {Log.Last()}");
        }
    }

    // Reset vote flag at the start of a new term or leader election
    public void ResetVote()
    {
        HasVoted = false;
    }
}

public class RaftCluster
{
    private List<RaftNode> nodes;
    private int majority;
    private int electionTimeout;
    private Random random = new Random();

    public RaftCluster(List<RaftNode> clusterNodes)
    {
        nodes = clusterNodes;
        majority = nodes.Count / 2 + 1;
        electionTimeout = random.Next(150, 300);  // Randomized election timeout to avoid split votes
    }

    // Leader election process
    public void ElectLeader()
    {
        int highestTerm = 0;
        RaftNode leader = null;
        int votes = 0;

        // Candidate requests votes from all other nodes
        foreach (var node in nodes)
        {
            if (node.RequestVote(highestTerm + 1))  // Requesting vote for the next term
            {
                highestTerm = node.CurrentTerm;
                votes++;
            }
        }

        // If a majority votes for the candidate, it becomes the leader
        if (votes >= majority)
        {
            leader = nodes.FirstOrDefault(node => node.HasVoted);
            if (leader != null)
            {
                leader.IsLeader = true;
                Console.WriteLine($"{leader.NodeId} is elected as the leader.");
            }
        }
        else
        {
            Console.WriteLine("Leader election failed. Restarting election.");
        }
    }

    // Propose a value to be added to the log
    public void ProposeValue(string value)
    {
        var leader = nodes.FirstOrDefault(node => node.IsLeader);
        if (leader != null)
        {
            leader.AppendLog(value);
            leader.CommitLog(majority);
        }
        else
        {
            Console.WriteLine("No leader elected yet.");
        }
    }

    // Simulate leader failure and new leader election
    public void SimulateLeaderFailure()
    {
        var leader = nodes.FirstOrDefault(node => node.IsLeader);
        if (leader != null)
        {
            leader.IsLeader = false;  // Simulate leader failure
            Console.WriteLine($"{leader.NodeId} has failed, starting new leader election...");
            ElectLeader();  // Trigger new leader election
        }
    }

    // Run a simple loop simulating heartbeat and client requests
    public void RunCluster()
    {
        // Start by electing a leader
        ElectLeader();

        // Simulate client request
        ProposeValue("First Log Entry");

        // Simulate leader failure after some time
        Thread.Sleep(random.Next(1000, 2000)); // Simulate some random time before leader failure
        SimulateLeaderFailure();

        // Simulate new leader election and client request
        ProposeValue("Second Log Entry");
    }
}

public class Program
{
    public static void Main(string[] args)
    {
        // Initialize a few nodes in the Raft cluster
        var nodes = new List<RaftNode>
        {
            new RaftNode("Node1"),
            new RaftNode("Node2"),
            new RaftNode("Node3")
        };

        // Instantiate Raft protocol with the list of nodes
        var raftCluster = new RaftCluster(nodes);

        // Run the cluster simulation
        raftCluster.RunCluster();
    }
}
```

---

### 4. Zab (ZooKeeper Atomic Broadcast) Protocol: A Comprehensive Exploration

#### What is Zab?

Zab (ZooKeeper Atomic Broadcast) is a consensus protocol specifically designed for managing metadata and configurations in distributed systems. It's the heart of Apache ZooKeeper, providing a robust mechanism for maintaining consistency across distributed services. It ensures all actions are executed in the same order across all nodes.

#### Core Principles of Zab

1. **Atomic Broadcast**: Ensures all nodes receive messages in the same order
2. **Crash Recovery**: Ability to recover from node failures
3. **Leader-Based Consensus**: Uses a leader to coordinate transactions

#### Zab Protocol Workflow

1. **Leader Election**
   - Nodes enter "Looking" state
   - Elect a leader based on highest transaction ID
   - Leader transitions to "Leading" state
   - Followers transition to "Following" state

2. **Transaction Proposal**
   - Leader proposes a transaction
   - Generates a unique Zxid
   - Broadcasts to all followers
   - Waits for majority acknowledgment

3. **Transaction Commit**
   - Followers receive and log the transaction
   - Send acknowledgments to leader
   - Leader commits once majority agrees

#### C# Simple Implementation of Zab protocol

```csharp
public class ZabProtocol
{
    // Enhanced ZooKeeper Node with more details
    public class ZooKeeperNode
    {
        public string NodeId { get; set; }
        public string IpAddress { get; set; }
        public int Port { get; set; }
        public long LastZxid { get; set; }
        public NodeState State { get; set; }
        public DateTime LastHeartbeat { get; set; }
        public bool IsActive { get; set; }

        // Enhanced node joining mechanism
        public static ZooKeeperNode CreateNode(string nodeId, string ipAddress, int port)
        {
            return new ZooKeeperNode
            {
                NodeId = nodeId,
                IpAddress = ipAddress,
                Port = port,
                State = NodeState.Looking,
                LastHeartbeat = DateTime.UtcNow,
                IsActive = true
            };
        }
    }

    // Transaction representation
    public class ZabTransaction
    {
        public long Zxid { get; set; }
        public byte[] Data { get; set; }
        public TransactionType Type { get; set; }
        public DateTime Timestamp { get; set; }
    }

    // Possible transaction types
    public enum TransactionType
    {
        Create,
        Update,
        Delete,
        SetData,
        JoinCluster,
        LeaveCluster
    }

    // Node states in Zab protocol
    public enum NodeState
    {
        Looking,     // Searching for a leader
        Following,   // Following the current leader
        Leading      // Current leader
    }

    // Cluster Management
    public class ClusterManager
    {
        private ConcurrentDictionary<string, ZooKeeperNode> _clusterNodes 
            = new ConcurrentDictionary<string, ZooKeeperNode>();

        public bool AddNode(ZooKeeperNode node)
        {
            bool added = _clusterNodes.TryAdd(node.NodeId, node);
            
            if (added)
            {
                // Broadcast node addition
                BroadcastNodeJoin(node);
                return true;
            }
            
            return false;
        }

        public bool RemoveNode(string nodeId)
        {
            bool removed = _clusterNodes.TryRemove(nodeId, out var node);
            
            if (removed)
            {
                // Broadcast node removal
                BroadcastNodeLeave(node);
                return true;
            }
            
            return false;
        }

        public List<ZooKeeperNode> GetActiveNodes()
        {
            return _clusterNodes
                .Where(n => n.Value.IsActive)
                .Select(n => n.Value)
                .ToList();
        }

        private void BroadcastNodeJoin(ZooKeeperNode node)
        {
            Console.WriteLine($"Node {node.NodeId} joined the cluster at {node.IpAddress}:{node.Port}");
        }

        private void BroadcastNodeLeave(ZooKeeperNode node)
        {
            Console.WriteLine($"Node {node.NodeId} left the cluster");
        }
    }

    // Leader Election Mechanism
    public class LeaderElection
    {
        private ClusterManager _clusterManager;
        private ZooKeeperNode _currentLeader;

        public LeaderElection(ClusterManager clusterManager)
        {
            _clusterManager = clusterManager;
        }

        public ZooKeeperNode ElectLeader()
        {
            var activeNodes = _clusterManager.GetActiveNodes();
            
            if (activeNodes.Count == 0)
                throw new InvalidOperationException("No active nodes in cluster");

            // Election logic: Select node with highest transaction ID
            _currentLeader = activeNodes
                .OrderByDescending(n => n.LastZxid)
                .First();

            // Update node states
            foreach (var node in activeNodes)
            {
                node.State = node.NodeId == _currentLeader.NodeId 
                    ? NodeState.Leading 
                    : NodeState.Following;
            }

            Console.WriteLine($"Leader elected: {_currentLeader.NodeId}");
            return _currentLeader;
        }

        public ZooKeeperNode GetCurrentLeader()
        {
            return _currentLeader;
        }
    }

    // Atomic Broadcast Implementation
    public class AtomicBroadcast
    {
        private ConcurrentQueue<ZabTransaction> _transactionLog 
            = new ConcurrentQueue<ZabTransaction>();
        
        private long _currentZxid = 0;
        private ClusterManager _clusterManager;

        public AtomicBroadcast(ClusterManager clusterManager)
        {
            _clusterManager = clusterManager;
        }

        public bool ProposeTransaction(ZabTransaction transaction)
        {
            // Validate transaction
            if (!ValidateTransaction(transaction))
                return false;

            // Assign unique transaction ID
            transaction.Zxid = GenerateZxid();
            transaction.Timestamp = DateTime.UtcNow;

            // Broadcast to all active nodes
            bool broadcastResult = BroadcastTransaction(transaction);

            if (broadcastResult)
            {
                CommitTransaction(transaction);
                return true;
            }

            return false;
        }

        private long GenerateZxid()
        {
            return Interlocked.Increment(ref _currentZxid);
        }

        private bool BroadcastTransaction(ZabTransaction transaction)
        {
            var activeNodes = _clusterManager.GetActiveNodes();
            
            if (activeNodes.Count == 0)
                return false;

            // Simulate broadcast to followers
            int successfulBroadcasts = activeNodes
                .Count(node => SendTransactionToNode(node, transaction));

            // Require majority consensus
            return successfulBroadcasts > activeNodes.Count / 2;
        }

        private bool SendTransactionToNode(ZooKeeperNode node, ZabTransaction transaction)
        {
            try
            {
                // Simulate network transmission and processing
                node.LastZxid = transaction.Zxid;
                return true;
            }
            catch
            {
                return false;
            }
        }

        private void CommitTransaction(ZabTransaction transaction)
        {
            _transactionLog.Enqueue(transaction);
            Console.WriteLine($"Transaction committed: Zxid {transaction.Zxid}");
        }

        private bool ValidateTransaction(ZabTransaction transaction)
        {
            // Basic validation
            return transaction != null && transaction.Data != null;
        }
    }

    // Main Program to Demonstrate ZAB Protocol
    public class ZabProtocolDemo
    {
        public static void Main(string[] args)
        {
            // Create Cluster Manager
            var clusterManager = new ClusterManager();

            // Create Leader Election
            var leaderElection = new LeaderElection(clusterManager);

            // Create Atomic Broadcast
            var atomicBroadcast = new AtomicBroadcast(clusterManager);

            // Add initial nodes
            var node1 = ZooKeeperNode.CreateNode("node1", "192.168.1.100", 2181);
            var node2 = ZooKeeperNode.CreateNode("node2", "192.168.1.101", 2182);
            var node3 = ZooKeeperNode.CreateNode("node3", "192.168.1.102", 2183);

            // Add nodes to cluster
            clusterManager.AddNode(node1);
            clusterManager.AddNode(node2);
            clusterManager.AddNode(node3);

            // Elect initial leader
            var leader = leaderElection.ElectLeader();

            // Simulate transactions
            var transaction1 = new ZabTransaction
            {
                Data = System.Text.Encoding.UTF8.GetBytes("Initial data"),
                Type = TransactionType.Create
            };

            // Propose transaction
            bool transactionResult = atomicBroadcast.ProposeTransaction(transaction1);
            Console.WriteLine($"Transaction Proposed: {transactionResult}");

            // Simulate adding a new node
            var newNode = ZooKeeperNode.CreateNode("node4", "192.168.1.103", 2184);
            clusterManager.AddNode(newNode);

            // Re-elect leader after node addition
            leader = leaderElection.ElectLeader();
        }
    }
}
```

---

### 5. Gossip Protocol: Distributed Information Propagation

#### What is the Gossip Protocol?

The Gossip Protocol is a communication mechanism where nodes randomly share state information with each other, similar to how rumors spread in social networks.

#### Key Characteristics
- Decentralized
- Eventually consistent
- Highly fault-tolerant
- Low overhead communication

#### Gossip Protocol Workflow

1. **Initialization**
   - Each node maintains a local view of cluster state
   - Periodic gossip intervals
   - Random node selection for state sharing

2. **State Propagation**
   - Node selects random peers
   - Shares its current state
   - Peers merge received information
   - Timestamp-based conflict resolution

3. **Failure Detection**
   - Track last update time
   - Mark nodes as "suspected" if no recent updates
   - Self-healing mechanism

#### C# Simple Implementation of Gossip Protocol

```csharp
public class GossipProtocol
{
    // Represents a node in the gossip network
    public class GossipNode
    {
        public string NodeId { get; set; }
        public Dictionary<string, NodeInfo> ClusterState { get; set; }
        public int GossipInterval { get; set; } = 1000; // Milliseconds
    }

    // Detailed node information
    public class NodeInfo
    {
        public string Status { get; set; }
        public DateTime LastUpdated { get; set; }
        public Dictionary<string, object> Metadata { get; set; }
    }

    // Gossip communication manager
    public class GossipManager
    {
        private List<GossipNode> clusterNodes;
        private Random randomSelector = new Random();

        // Periodic gossip dissemination
        public void StartGossipDissemination(GossipNode currentNode)
        {
            while (true)
            {
                // Select random subset of nodes to gossip with
                var selectedNodes = SelectRandomNodes(3);
                
                foreach (var targetNode in selectedNodes)
                {
                    ShareNodeState(currentNode, targetNode);
                }

                // Wait before next gossip round
                Thread.Sleep(currentNode.GossipInterval);
            }
        }

        // Select random subset of nodes
        private List<GossipNode> SelectRandomNodes(int count)
        {
            return clusterNodes
                .OrderBy(x => randomSelector.Next())
                .Take(count)
                .ToList();
        }

        // Share node state with another node
        private void ShareNodeState(GossipNode sourceNode, GossipNode targetNode)
        {
            // Merge state information
            foreach (var stateEntry in sourceNode.ClusterState)
            {
                // Update if newer or not existing
                if (!targetNode.ClusterState.ContainsKey(stateEntry.Key) ||
                    targetNode.ClusterState[stateEntry.Key].LastUpdated < stateEntry.Value.LastUpdated)
                {
                    targetNode.ClusterState[stateEntry.Key] = stateEntry.Value;
                }
            }
        }

        // Failure detection mechanism
        public void DetectFailedNodes(GossipNode currentNode)
        {
            var currentTime = DateTime.UtcNow;
            var failedNodes = currentNode.ClusterState
                .Where(node => 
                    (currentTime - node.Value.LastUpdated).TotalSeconds > 30)
                .ToList();

            foreach (var failedNode in failedNodes)
            {
                // Mark node as potentially failed
                currentNode.ClusterState[failedNode.Key].Status = "SUSPECTED";
            }
        }
    }
}
```

---

## My Custom Implementation Journey

After exploring these algorithms, I realized:

- **Complex systems** like Paxos are best avoided unless absolutely necessary.
- **Raft** is a practical choice for most use cases due to its clarity.
- **Zab** is perfect for metadata-heavy tasks, while **Gossip** excels at large-scale, fault-tolerant systems.

---

## Key Takeaways

1. **No one-size-fits-all**: Choose an algorithm based on your system's consistency and scalability needs.
2. **Understand the trade-offs**: Balance complexity, performance, and fault tolerance.
3. **Design for failure**: Consensus is as much about recovery as agreement.

---

## Reflection

What began as a simple singleton challenge became a journey into distributed systems’ depths. These algorithms offer not just solutions, but a lens into the art of designing robust, scalable, and fault-tolerant systems.

**Which algorithm fits your project? Let’s discuss how to build consensus for your distributed system!**
