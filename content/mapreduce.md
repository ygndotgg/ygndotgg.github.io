+++
title = "MapReduce in Rust: A Hands-On Guide to Building Distributed Systems from Scratch"
date = 2025-02-15
[taxonomies]
tags = ["rust", "distributed-systems", "mapreduce"]
categories = ["rust", "systems"]
+++

# MapReduce in Rust: A Hands-On Guide to Building Distributed Systems from Scratch

*"You don't truly understand distributed systems until you've built one from scratch."*

That's what everyone tells you. Building a production-grade distributed system involves countless edge cases, failure scenarios, and subtle synchronization challenges that are impossible to appreciate until you've debugged a race condition at 3 AM.

But honestly, they are WRONG. You don't need production scale to understand distributed systems. You need a working implementation that forces you to confront every hard problem the field has to offer.

This is a technical deep-dive into building a MapReduce system in Rust, from first principles to a working implementation. We'll cover RPC communication, fault tolerance, task scheduling, and the elegant simplicity that makes MapReduce such a powerful abstraction.

---

## What This Post Covers

**[Part I: The Foundation](/mapreduce/#part-i-the-foundation)** - RPC, gRPC, and the communication layer that makes distributed systems possible

**[Part II: The Master](/mapreduce/#part-ii-the-master)** - Task scheduling, fault tolerance through backup tasks, and health checks for stuck workers

**[Part III: The Worker](/mapreduce/#part-iii-the-worker)** - Map and Reduce implementation, partitioning, and atomic file operations

**[Part IV: The Complete System](/mapreduce/#part-iv-the-complete-system)** - Putting it all together, code architecture, and running the system

---

## Prologue: Where Does it All Started

### The MapReduce Abstraction

In 2004, Google published a paper that would revolutionize how we process large-scale data: "MapReduce: Simplified Data Processing on Large Clusters." The idea was elegantly simple—any problem can be solved by breaking it into two phases:

1. **Map**: Transform raw input into key-value pairs
2. **Reduce**: Aggregate those pairs into final results

This abstraction is powerful because it hides all the complexity of distributed computing behind two function signatures. You don't need to think about network partitions, load balancing, or fault tolerance. The framework handles it all.

But here's the thing—reading about it and implementing it are completely different experiences.

### My Journey: From Curiosity to Implementation

I first encountered MapReduce in academic course MIT 6.824 Distributed Systems, I had understood, its motive but there are some small pitholes that only filled when i try to implement it on my own.

This is the story of building a MapReduce system in Rust, Althrough it was supposed to Finish the Lab1 of MIT 6.824 Distributed Systems

But more importantly, this is a manual on how to think about distributed systems—the mental models, the failure modes, and the elegant solutions that emerge when you build things from first principles.

---

## PART I: The Foundation

### Chapter 1: The Communication Problem

#### The Challenge of Distributed Communication

In a distributed system, processes don't share memory. They communicate over the network, which is unreliable, slow, and can fail at any moment. Before we can build a distributed MapReduce, we need a way for the Master to tell Workers what to do, and for Workers to report results back.

This is the RPC problem: how do we make a network call look like a local function call?

#### Attempt 1: Raw TCP Sockets

The first version of this project started with raw TCP sockets. Workers would connect to the Master and exchange serialized messages:

```
Worker: "I'd like a task, please"
Master: "Here's task #3 - process input/file1.txt"
Worker: "Done! Here are my results: {word1: 42, word2: 17, ...}"
```

This approach works but has serious limitations:

1. **No contract**: Every message format must be manually defined and validated
2. **No type safety**: You're serializing bytes and hoping both sides agree on the format
3. **No code generation**: Every new RPC method requires writing boilerplate

#### The Solution: gRPC with Protocol Buffers

gRPC solves these problems by using Protocol Buffers (proto3) for schema definition:

```
service MapReduce {
  rpc GetTask (Empty) returns (TaskResponse);
  rpc MapDone (MapDoneRequest) returns (Empty);
  rpc ReduceDone (ReduceDoneRequest) returns (Empty);
}

message TaskResponse {
  string task_type = 1;
  uint32 task_id = 2;
  repeated string input_files = 3;
  uint32 n_reduce = 4;
  string output_path = 5;
}
```

The key insight is that the `.proto` file is the **single source of truth**. From this definition, code is automatically generated for both client and server. This guarantees that both sides speak the same protocol—type errors become compile-time errors, not runtime surprises.

Rust's implementation of gRPC comes from the **tonic** crate, which provides an async runtime built on top of tokio. This is perfect for our use case: workers can process tasks concurrently while waiting for RPC responses.

```
// From src/client.rs
pub async fn get_task(&mut self) -> Result<TaskType, Box<dyn std::error::Error>> {
    let response = self.inner.get_task(mr::Empty {}).await?.into_inner();
    // Type-safe response parsing - the compiler catches mismatches
    match response.task_type.as_str() {
        "map" => TaskType::Map { ... },
        "reduce" => TaskType::Reduce { ... },
        "idle" => TaskType::Idle,
        "exit" => TaskType::Exit,
    }
}
```

### Chapter 2: The Architecture That Emerged

#### The Master-Worker Pattern

The commit history shows the natural evolution of this project:

1. **Initial commit** - Types and structure definitions
2. **TCP implementation** - Raw socket communication
3. **gRPC migration** - Using tonic for type-safe RPC

This pattern is universal in distributed systems: a central coordinator (Master) assigns work to multiple workers who execute in parallel. The Master doesn't do the heavy lifting—it orchestrates.

Here's how the architecture looks:

```
┌─────────────┐         gRPC         ┌─────────────┐
│   Master    │ ◄──────────────────► │   Worker 1  │
│  (Port 50051)│                     │  (Task: Map)│
└─────────────┘                     └─────────────┘
      │                                    │
      │                                    │
      ▼                                    ▼
┌─────────────┐                     ┌─────────────┐
│ Task Queue  │                     │   Worker 2  │
│  - Map 0-4  │                     │ (Task: Map) │
│  - Reduce 0-4                     └─────────────┘
└─────────────┘
```

The Master maintains two task queues: one for Map tasks and one for Reduce tasks. Workers poll the Master for work, execute the task, and report back when complete.

---

## PART II: The Master

### Chapter 3: The Task Scheduling Problem

#### The Naive Approach

A first attempt at task scheduling might look like this:

```
fn get_task(&mut self) -> Response {
    // Find any idle task
    if let Some((id, _)) = self.map_task.iter().find(|(_, s)| **s == Idle) {
        return Response::Task { task_id: id, ... };
    }
    Response::NoTask
}
```

This works—until it doesn't. The problem emerges when:

1. Worker A claims Task 1
2. Worker A crashes
3. Task 1 is now "InProgress" forever
4. All other workers are idle waiting for Task 1
5. The entire job is deadlocked

#### The Solution: Task Status Tracking

The implementation tracks detailed task state:

```
// From src/rpc.rs
pub enum TaskStatus {
    Idle,
    InProgress {
        start_time: std::time::Instant,
        backup_scheduled: bool,
    },
    Completed,
}
```

The Master knows exactly which tasks are:

* **Idle**: Available for workers to claim
* **InProgress**: Currently being processed (with timestamp!)
* **Completed**: Successfully finished

### Chapter 4: The Fault Tolerance Problem

#### The 10% Rule

Google's original MapReduce paper introduced a clever solution: when 5% of tasks remain, start scheduling **backup tasks** for in-progress work. If the original worker finishes, its results are ignored. If it fails, the backup task provides the result.

Here's the implementation:

```
// From src/master.rs
fn should_schedule_backup(&self) -> bool {
    let total = self.map_task.len();
    let completed = self.map_task.values()
        .filter(|s| matches!(s, TaskStatus::Completed))
        .count();
    
    let remaining = total - completed;
    remaining <= (total as f64 * 0.05) as usize
}
```

When should\_backup() returns true, the Master looks for tasks that have been running for more than 10 seconds without a backup scheduled:

```
let backup_threshold = Duration::from_secs(10);
let backup_task = self.map_task.iter()
    .find(|(_, status)| {
        matches!(status, TaskStatus::InProgress { start_time, backup_scheduled }
            if !backup_scheduled && start_time.elapsed() > backup_threshold)
    });
```

This is the **speculative execution** pattern—running the same task twice in parallel, using whichever finishes first.

### Chapter 5: The Health Check Problem

#### Detecting Stuck Tasks

What happens when a worker doesn't crash but just... stops responding? The backup mechanism won't help because no backup is scheduled until 10 seconds elapse.

The solution is periodic health checks:

```
// From src/master.rs
pub fn health_check(&mut self, timeout_secs: u64) {
    let timeout = Duration::from_secs(timeout_secs);
    
    for (task_id, status) in &mut self.map_task {
        if let TaskStatus::InProgress { start_time, .. } = status {
            if start_time.elapsed() > timeout {
                log::warn!("Map task {} timed out, resetting to Idle", task_id);
                *status = TaskStatus::Idle;
            }
        }
    }
}
```

The Master runs a background thread that checks every 10 seconds whether any tasks have been "InProgress" for more than 30 seconds. If so, it resets them to Idle, making them available for other workers.

**This is the core insight of fault tolerance**: assume things will fail, detect failures aggressively, and recover gracefully.

### Chapter 6: Phase Transitions

#### The Map → Reduce Handoff

One of the trickiest parts of MapReduce is knowing when to switch phases:

```
// From src/master.rs
fn handle_map_done(&mut self, task_id: u32, files: HashMap<u32, String>) {
    self.map_task.insert(task_id, TaskStatus::Completed);
    self.map_outputs.insert(task_id, files);
    
    // Check if ALL map tasks are completed
    let all_done = self.map_task.values()
        .all(|s| matches!(s, TaskStatus::Completed));
    
    if all_done && self.phase == Phase::Map {
        self.phase = Phase::Reduce;
        // Initialize reduce tasks
        for i in 0..self.n_reduce {
            self.reduce_task.insert(i, TaskStatus::Idle);
        }
    }
}
```

The Master won't assign Reduce tasks until every Map task is complete. This makes sense: you can't reduce until you have all the intermediate data from the Map phase.

---

## PART III: The Worker

### Chapter 7: The Map Phase

#### Input Processing

When a worker receives a Map task, it reads the input file and transforms it into key-value pairs:

```
// From src/worker.rs
fn map(_filename: &String, content: String) -> Vec<KeyValue> {
    // Split content into words, each with count 1
    let mut d: Vec<(&str, u32)> = content.split(" ").map(|x| (x, 1)).collect();
    d.sort_by_key(|f| f.0);
    
    // Group by key and reduce (count occurrences)
    let mut kvs = Vec::new();
    let mut i = 0;
    while i < d.len() {
        let key = d[i].0;
        let mut values = Vec::new();
        while i < d.len() && key == d[i].0 {
            values.push(d[i].1);
            i += 1;
        }
        let result = reduce(key.to_string(), values);
        let kv = result.split_once(" ")
            .map(|(k, v)| KeyValue { key: k.to_string(), value: v.to_string() })
            .expect("Unable to Convert to key Value form");
        kvs.push(kv);
    }
    kvs
}
```

For input "hello world hello", this produces:

* ("hello", "2")
* ("world", "1")

#### Partitioning: The Hash Function

The worker must now decide which Reduce partition should handle each key. This uses consistent hashing:

```
// From src/worker.rs
pub fn ihash(key: &str) -> u32 {
    let mut hasher = DefaultHasher::new();
    key.hash(&mut hasher);
    hasher.finish() as u32
}

// In the Map task execution:
let partition_id = ihash(&kv.key) % data.n_reduce;
```

All occurrences of "hello" will hash to the same partition, ensuring they're processed together in Reduce.

#### Atomic File Writing

Workers write intermediate files using atomic operations:

```
// Write to temp file first
let temp_filename = format!("{}/mr-{}-{}.tmp", output_path, task_id, partition_id);
let final_filename = format!("{}/mr-{}-{}", output_path, task_id, partition_id);

let mut file = File::create(&temp_filename)?;
for v in &values {
    writeln!(file, "{}", v)?;
}
file.flush()?;

// Atomic rename - either fully written or not visible
fs::rename(&temp_filename, &final_filename)?;
```

The key insight: writing to a temporary file and then renaming is atomic on most filesystems. If the worker crashes mid-write, the original file remains intact.

### Chapter 8: The Reduce Phase

#### Collecting Intermediate Files

Reduce workers receive a list of intermediate files from the Master:

```
// From src/master.rs - when assigning reduce task
let mut input_files = Vec::new();
for (_map_id, files) in &self.map_outputs {
    if let Some(file) = files.get(&id) {
        input_files.push(file.clone());
    }
}
```

This is the "shuffle" phase—each Reduce task gets ALL intermediate files for its partition.

#### Sorting and Aggregation

The Reduce worker reads all intermediate files, sorts by key, and aggregates:

```
// From src/worker.rs
// Read all input files into a single vector
let mut all_kv: Vec<(String, u32)> = Vec::new();
for file in &data.input_files {
    let content = read_to_string(file)?;
    for item in content.split(";") {
        if let Some((k, v)) = item.split_once(",") {
            if !k.is_empty() {
                all_kv.push((k.to_string(), v.parse().unwrap_or(0)));
            }
        }
    }
}

// Sort by key
all_kv.sort_by_key(|f| f.0.clone());

// Aggregate: group by key, sum values
let mut i = 0;
while i < all_kv.len() {
    let key = all_kv[i].0.clone();
    let mut values = Vec::new();
    while i < all_kv.len() && key == all_kv[i].0 {
        values.push(all_kv[i].1);
        i += 1;
    }
    let result = reduce(key, values);
    writeln!(file, "{}", result)?;
}
```

This is the classic "sort-merge" pattern: data is sorted by key, then a single pass groups and aggregates identical keys.

---

## PART IV: The Complete System

### Chapter 9: The Code Architecture

#### Module Structure

```
src/
├── lib.rs          # Public API - re-exports all modules
├── master.rs       # Master implementation - scheduling, fault tolerance
├── worker.rs       # Worker implementation - map/reduce logic
├── rpc.rs          # RPC types - requests, responses, task status
├── models.rs       # Domain models - KeyValue, Report
├── server.rs       # gRPC server - tonic service implementation
└── client.rs       # gRPC client - worker side RPC calls

bin/
├── master.rs       # Master binary entry point
└── worker.rs       # Worker binary entry point
```

#### Separation of Concerns

The architecture follows clean separation:

1. **rpc.rs** defines the communication contract
2. **master.rs** implements scheduling logic
3. **worker.rs** implements data processing
4. **server.rs** / **client.rs** handle gRPC boilerplate

This makes the code testable and maintainable. The Master logic can be tested without network calls; the Worker can run locally without a Master.

### Chapter 10: Running the System

#### Starting the Master

```
cargo run --release --bin master 50051
```

The Master starts on port 50051 with the default configuration:

* Input files: input/file1.txt through input/file5.txt
* Number of reduce tasks: 5
* Output directory: output/

#### Starting Workers

```
cargo run --release --bin worker http://127.0.0.1:50051
```

Workers connect to the Master and automatically pick up tasks. Start multiple workers to see parallel execution:

```
# Terminal 1
cargo run --release --bin master 50051

# Terminal 2
cargo run --release --bin worker http://127.0.0.1:50051

# Terminal 3
cargo run --release --bin worker http://127.0.0.1:50051
```

The log output shows the system in action:

```
# Master logs:
INFO mapreduce::server: gRPC Master listening on 127.0.0.1:50051
INFO mapreduce::master: All map tasks complete, switching to Reduce phase
INFO mapreduce::master: All reduce tasks complete, job finished!

# Worker logs:
INFO mapreduce::worker: Asking for task...
INFO mapreduce::worker: Map task 0 complete, sending MapDone...
INFO mapreduce::worker: Asking for task...
INFO mapreduce::worker: Reduce task 2 complete, sending ReduceDone...
```

### Chapter 11: The Final Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         Master                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────────┐│
│  │ Task Queue  │  │ Health Check │  │ Phase Transitions       ││
│  │ - Map 0-4   │  │ Every 10s    │  │ Map → Reduce → Done     ││
│  │ - Reduce 0-4│  │              │  │                         ││
│  └─────────────┘  └──────────────┘  └─────────────────────────┘│
│          │                │                      │              │
│          ▼                ▼                      ▼              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    gRPC Server (tonic)                       ││
│  │  - get_task() → TaskResponse                                ││
│  │  - map_done(MapDoneRequest) → Empty                        ││
│  │  - reduce_done(ReduceDoneRequest) → Empty                   ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│    Worker 1     │  │    Worker 2     │  │    Worker 3     │
│  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │
│  │ Map Phase │  │  │  │ Map Phase │  │  │  │   Idle    │  │
│  │ Read file │  │  │  │ Read file │  │  │  └───────────┘  │
│  │ Partition │  │  │  │ Partition │  │  │                  │
│  └───────────┘  │  │  └───────────┘  │  │                  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
    output/mr-0-*       output/mr-1-*          (waiting)
```

---

## Epilogue: The Rust Advantage

### Why Rust for Distributed Systems?

Building this in Rust provided several advantages:

1. **Memory Safety**: No segfaults or data races, even with concurrent workers
2. **Async/Await**: Tonic's async gRPC fits naturally with tokio's cooperative multitasking
3. **Type Safety**: The compiler catches mistakes that would slip through in other languages

### What This Journey Taught Me

The MapReduce paper is 20 years old, but the lessons remain:

1. **Abstraction simplifies complexity**: The Map/Reduce split hides enormous complexity behind simple function signatures
2. **Fault tolerance is an afterthought in design, a first-thought in implementation**: Every line of code assumes things will fail
3. **Small, focused components**: The Master doesn't process data; it only coordinates. Workers don't coordinate; they only process. This separation makes each piece understandable
4. **The power of streaming**: Both Map and Reduce are streaming transformations—they don't need to hold all data in memory

### The Code

This implementation is available on GitHub:

**Repository**: <https://github.com/ygndotgg/map_reduce->

The commit history tells the story of evolution:

* TCP sockets → gRPC migration
* Sequential → Concurrent workers
* Basic task scheduling → Backup task speculation
* Health checks for fault tolerance

### Final Thoughts

Building a distributed system from scratch is humbling. Every "simple" operation—reading a file, sending a message, updating state—becomes a source of potential failure. But that's exactly why it's worth doing.

You don't build distributed systems because they're easy. You build them because they force you to think clearly about failure, concurrency, and coordination. And you won't truly understand those concepts until you've implemented them yourself.

So go ahead—fork the repo, break things, add features. The best way to learn is by doing.

---

*MapReduce in Rust: A Hands-On Guide to Building Distributed Systems from Scratch | my thoughts*