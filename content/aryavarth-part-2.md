+++
title = "Aryavarth: Distributed Key Value Store Part 2"
date = 2025-02-10
[taxonomies]
tags = ["rust", "distributed-systems", "networking", "concurrency"]
categories = ["rust", "systems", "networking"]
+++

# Aryavarth: Distributed Key Value Store - Part 2

## Part 2: The Networking Layer & Lock-Free Concurrency

---

### Prologue: From Library to Network Service

In [Part 1](/aryavarth1), we built a **single-process key-value store**. It worked beautifully within one application. But a database that lives only in one process is just a library—not a service.

For the complete code implementation, see [src/kvs.rs](https://github.com/ygndotgg/kvs_store). Check the commit history for the actual implementation details.

Real distributed systems need **networking**. Multiple clients, across multiple machines, talking to a shared data store. That's where the real engineering begins.

This is Part 2: where we transform our local KvStore into a **networked, concurrent, distributed key-value store**.

---

## What This Post Covers

**[Part I: The Networking Layer](/aryavarth-part-2/#part-i-the-networking-layer)** - TCP servers, Rust async, and turning a library into a service

**[Part II: Thread Safety](/aryavarth-part-2/#part-ii-thread-safety)** - Mutexes, locks, and the challenges of concurrent access

**[Part III: Lock-Free Concurrency](/aryavarth-part-2/#part-iii-lock-free-concurrency)** - Atomic operations, lock-free data structures, and performance

**[Part IV: The Complete Architecture](/aryavarth-part-2/#part-iv-the-complete-architecture)** - Putting it all together in Rust

---

## Section I: The Network Protocol

### Chapter 1: Designing the Communication Contract

Before writing a single line of networking code, we need a **protocol**. What do clients send? What do servers return?

#### The Request-Response Model

```
Client                                                  Server
   │                                                       │
   │───────── Request: Set("user:1", "Alice") ──────────►│
   │                                                       │
   │◄──────── Response: Ok(None) ─────────────────────────│
   │                                                       │
   │───────── Request: Get("user:1") ───────────────────►│
   │                                                       │
   │◄──────── Response: Ok(Some("Alice")) ───────────────│
   │                                                       │
   │───────── Request: Remove("user:1") ────────────────►│
   │                                                       │
   │◄──────── Response: Ok(None) ─────────────────────────│
```

#### The Wire Format

We use **JSON over TCP**—simple, debuggable, and human-readable:

```
#[derive(Serialize, Deserialize, Debug)]
pub enum Request {
    Set { key: String, value: String },
    Get { key: String },
    Remove { key: String },
}

#[derive(Serialize, Deserialize, Debug)]
pub enum Response {
    Ok(Option<String>),  // Success: returns value for Get, nothing for Set/Remove
    Err(String),         // Failure: error message
}
```

**Why this design?**

* `Option<String>` for success handles the "key not found" case elegantly
* `Err(String)` carries human-readable error messages
* Enum variants become JSON objects: `{"Set":{"key":"user:1","value":"Alice"}}`

---

### Chapter 2: The Client Implementation

The client is the \*\* ambassador\*\* between the application and the server. It must be:

1. **Connection-aware**: Establish and maintain TCP connections
2. **Protocol-compliant**: Serialize requests, deserialize responses
3. **Error-resistant**: Handle network failures gracefully

#### The KvsClient Structure

```
use std::net::TcpStream;
use std::io::{Read, Write};
use std::result::Result;

pub struct KvsClient {
    stream: TcpStream,
}

impl KvsClient {
    /// Connect to a KVS server at the given address
    pub fn connect(addr: &str) -> Result<Self> {
        let stream = TcpStream::connect(addr)?;
        stream.shutdown(Shutdown::Write)?;  // We'll signal "done sending" explicitly
        Ok(KvsClient { stream })
    }

    /// Send a Set request: key → value
    pub fn set(&mut self, key: String, value: String) -> Result<()> {
        let request = Request::Set { key, value };
        self.send_request(request)
    }

    /// Send a Get request: retrieve value by key
    pub fn get(&mut self, key: String) -> Result<Option<String>> {
        let request = Request::Get { key };
        self.send_request(request)
    }

    /// Send a Remove request: delete a key
    pub fn remove(&mut self, key: String) -> Result<()> {
        let request = Request::Remove { key };
        self.send_request(request)
    }

    /// Core: serialize request, send, deserialize response
    fn send_request<R: Serialize>(&mut self, request: R) -> Result<Response> {
        let mut serializer = serde_json::Serializer::new(&mut self.stream);
        request.serialize(&mut serializer)?;
        self.stream.shutdown(Shutdown::Write)?;  // Signal request complete
        
        let response: Response = serde_json::from_reader(&mut self.stream)?;
        Ok(response)
    }
}
```

#### The Client Flow in Action

```
// Usage example:
let mut client = KvsClient::connect("127.0.0.1:5000")?;

client.set("user:42".into(), "Aryavarth".into())?;
println!("Set user:42 = Aryavarth");

let value = client.get("user:42".into())?;
println!("Got: {:?}", value);  // Some("Aryavarth")

client.remove("user:42".into())?;
println!("Removed user:42");
```

**Key insight:** The `shutdown(Write)` call signals end-of-request. The server knows when to stop reading and start processing.

---

### Chapter 3: The Server Architecture

The server is where **complexity lives**. It must:

1. **Accept connections** from multiple clients simultaneously
2. **Route requests** to the appropriate handler
3. **Execute operations** on the underlying KvStore engine
4. **Return responses** back to clients

#### The KvServer Structure

```
use std::net::{TcpListener, TcpStream};
use std::sync::Arc;
use std::io::{Read, Write};

pub struct KvServer<E, P>
where
    E: KvsEngine,           // Any engine: KvStore or SledKvsEngine
    P: ThreadPool,         // Any pool: Naive, SharedQueue, or Rayon
{
    engine: Arc<E>,        // The data store (cloned for each request)
    pool: P,               // Thread pool for concurrency
    listener: TcpListener, // Network listener
}

impl<E, P> KvServer<E, P>
where
    E: KvsEngine + Clone,
    P: ThreadPool,
{
    /// Create a new server bound to the given address
    pub fn new(addr: &str, engine: E, pool: P) -> Result<Self> {
        let listener = TcpListener::bind(addr)?;
        listener.nonblocking()?;  // Non-blocking for better concurrency
        
        Ok(KvServer {
            engine: Arc::new(engine),
            pool,
            listener,
        })
    }

    /// Run the server: accept connections and process requests
    pub fn run(&self) -> ! {
        loop {
            match self.listener.accept() {
                Ok((stream, addr)) => {
                    let engine = self.engine.clone();
                    self.pool.spawn(move || {
                        if let Err(e) = handle_connection(stream, engine) {
                            eprintln!("Error handling {}: {}", addr, e);
                        }
                    });
                }
                Err(e) => {
                    // No connection waiting, do other work or yield
                    std::thread::sleep(std::time::Duration::from_millis(1));
                }
            }
        }
    }
}
```

#### Request Handler: The Request Processor

```
use std::io::{BufReader, BufWriter};

fn handle_connection<E: KvsEngine>(
    stream: TcpStream,
    engine: E,
) -> Result<()> {
    let reader = BufReader::new(&stream);
    let mut writer = BufWriter::new(&stream);
    
    // Read one request per connection (HTTP-style, not persistent)
    let request: Request = serde_json::from_reader(reader)?;
    
    let response = match request {
        Request::Set { key, value } => {
            engine.set(key, value)?;
            Response::Ok(None)
        }
        Request::Get { key } => {
            let value = engine.get(key)?;
            Response::Ok(value)
        }
        Request::Remove { key } => {
            engine.remove(key)?;
            Response::Ok(None)
        }
    };
    
    serde_json::to_writer(&mut writer, &response)?;
    writer.flush()?;
    
    Ok(())
}
```

---

### Chapter 4: The Thread Pool Problem

A server that handles **one request at a time** is useless. We need **concurrency**. But how?

#### The Thread Lifecycle

```
Traditional Model:
                    ┌─────────────┐
    Request 1 ─────►│   Thread    │────► Process ─────► Response
                    └─────────────┘
                    
    Request 2 ─────►│   Thread    │────► Process ─────► Response
                    └─────────────┘
                    
    Request 3 ─────►│   Thread    │────► Process ─────► Response
                    └─────────────┘
                    
Problem: Unlimited threads = memory explosion + context switching hell
```

#### Attempt 1: Naive Thread Pool

```
pub struct NaiveThreadPool;

impl ThreadPool for NaiveThreadPool {
    fn spawn<F>(&self, job: F)
    where
        F: FnOnce() + Send + 'static,
    {
        std::thread::spawn(job);  // Create a new thread for EVERY job!
    }
}
```

**Problem:** 10,000 concurrent requests = 10,000 threads = system collapse.

#### Attempt 2: Shared Queue Thread Pool

The solution: **one pool of worker threads** pulling from a **shared queue**.

```
use std::sync::mpsc;

pub struct SharedQueueThreadPool {
    sender: mpsc::Sender<Message>,
    workers: Vec<Worker>,
}

enum Message {
    Job(Box<dyn FnOnce() + Send + 'static>),
    Terminate,
}

impl SharedQueueThreadPool {
    pub fn new(threads: u32) -> Result<Self> {
        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver));
        
        let mut workers = Vec::with_capacity(threads as usize);
        for _ in 0..threads {
            workers.push(Worker::new(Arc::clone(&receiver)));
        }
        
        Ok(SharedQueueThreadPool { sender, workers })
    }
}

impl ThreadPool for SharedQueueThreadPool {
    fn spawn<F>(&self, job: F)
    where
        F: FnOnce() + Send + 'static,
    {
        self.sender.send(Message::Job(Box::new(job))).unwrap();
    }
}
```

**How it works:**

```
                          ┌──────────────────┐
    Job (FnOnce) ────────►│   mpsc::Sender   │
                          └────────┬─────────┘
                                   │
                                   ▼
                          ┌──────────────────┐
                          │   mpsc::Receiver │
                          │   (protected by  │
                          │      Mutex)       │
                          └────────┬─────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
       ┌────────────┐      ┌────────────┐      ┌────────────┐
       │  Worker 1   │      │  Worker 2  │      │  Worker N  │
       │  pop job   │      │  pop job   │      │  pop job   │
       │  execute   │      │  execute   │      │  execute   │
       └────────────┘      └────────────┘      └────────────┘
```

#### Attempt 3: Rayon Thread Pool (Work-Stealing)

For CPU-intensive workloads, Rayon provides **work-stealing**—workers steal jobs from each other when idle:

```
pub struct RayonThreadPool {
    pool: rayon::ThreadPool,
}

impl ThreadPool for RayonThreadPool {
    fn new(threads: u32) -> Result<Self> {
        let pool = rayon::ThreadPoolBuilder::new()
            .num_threads(threads as usize)
            .build()?;
        Ok(RayonThreadPool { pool })
    }

    fn spawn<F>(&self, job: F)
    where
        F: FnOnce() + Send + 'static,
    {
        self.pool.spawn(job);
    }
}
```

**When to use which:**

| Pool Type | Best For | Weakness |
| --- | --- | --- |
| Naive | Learning, trivial loads | Memory explosion |
| SharedQueue | I/O-bound workloads | Single channel bottleneck |
| Rayon | CPU-bound parallelism | Not ideal for waiting on I/O |

---

## Section II: The Engine Abstraction

### Chapter 5: Why One Engine Is Not Enough

Our KvStore is custom-built with append-only logs. But what if we want to compare against **industry-standard engines** like Sled?

#### The Problem: Tight Coupling

```
// Before: Server only knows about KvStore
pub struct KvServer {
    store: KvStore,  // Hardcoded!
    // ...
}
```

#### The Solution: Trait-Based Abstraction

```
pub trait KvsEngine: Clone + Send + 'static {
    /// Set a key-value pair
    fn set(&self, key: String, value: String) -> Result<()>;

    /// Get a value by key
    fn get(&self, key: String) -> Result<Option<String>>;

    /// Remove a key
    fn remove(&self, key: String) -> Result<()>;
}
```

Now the server is **generic**:

```
pub struct KvServer<E, P>
where
    E: KvsEngine,    // Any engine that implements our trait
    P: ThreadPool,
{
    engine: Arc<E>,
    pool: P,
    listener: TcpListener,
}
```

#### Implementing for Custom KvStore

```
impl KvsEngine for KvStore {
    fn set(&self, key: String, value: String) -> Result<()> {
        self.inner.lock().unwrap().set(key, value)
    }

    fn get(&self, key: String) -> Result<Option<String>> {
        self.inner.lock().unwrap().get(key)
    }

    fn remove(&self, key: String) -> Result<()> {
        self.inner.lock().unwrap().remove(key)
    }
}
```

#### Implementing for Sled

```
use sled::Db;

pub struct SledKvsEngine {
    db: Db,
}

impl KvsEngine for SledKvsEngine {
    fn set(&self, key: String, value: String) -> Result<()> {
        self.db.insert(key.as_bytes(), value.as_bytes())?;
        self.db.flush()?;  // Ensure durability
        Ok(())
    }

    fn get(&self, key: String) -> Result<Option<String>> {
        Ok(self.db
            .get(key.as_bytes())?
            .map(|v| String::from_utf8_lossy(&v).into_owned()))
    }

    fn remove(&self, key: String) -> Result<()> {
        self.db.remove(key.as_bytes())?;
        self.db.flush()?;
        Ok(())
    }
}
```

**The beauty:** The server code never changes. Only the engine instantiation changes:

```
// Using custom KvStore
let engine = KvStore::open("./data")?;
let server = KvServer::new("127.0.0.1:5000", engine, pool);

// Using Sled
let engine = SledKvsEngine::new("./sled_data")?;
let server = KvServer::new("127.0.0.1:5000", engine, pool);
```

---

## Section III: The Lock-Free Revolution

### Chapter 6: The Concurrency Crisis

We've solved **network concurrency** with thread pools. But there's another concurrency problem: **data access concurrency**.

#### The Mutex Bottleneck

```
pub struct KvStore {
    inner: Mutex<KvStoreInner>,  // One lock to rule them all
}
```

**What happens:**

```
Thread 1 (Reader):  [=========LOCK=========.....get().....]
Thread 2 (Reader):                    [=========LOCK=========.....get().....]
Thread 3 (Writer):                                        [=========LOCK=========.....set().....]

Problem: Even READERS block each other!
```

#### The RwLock Attempt

```
pub struct KvStore {
    inner: RwLock<KvStoreInner>,  // Multiple readers allowed
}
```

**Better, but still problematic:**

```
Thread 1 (Reader):  [===READ===.....get().....]
Thread 2 (Reader):  [===READ===.....get().....]  ✓ Concurrent reads!
Thread 3 (Writer):              [===WRITE===.....set().....]
Thread 4 (Reader):                          [---BLOCKED---]

Problem: BufReader needs &mut for seek(), but RwLock gives &T!
```

**The fundamental trap:** You get shared access (`&T`) but file operations need exclusive access (`&mut T`).

---

### Chapter 7: Breaking the Monolith

The insight: **don't share everything**. Some things should be shared; others should be **instantiated fresh per-operation**.

#### Before: One Giant Lock

```
┌─────────────────────────────────────────────────────────────┐
│                     Mutex<KvStoreInner>                      │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  path: PathBuf                                         │ │
│  │  index: HashMap<String, LogPointer>                    │ │
│  │  writer: BufWriter                                     │ │
│  │  reader: HashMap<u64, BufReader>                       │ │
│  │  current_file_id: u64                                  │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

All synchronized together = maximum contention
```

#### After: Field-Level Synchronization

```
┌─────────────────────────────────────────────────────────────┐
│                       KvStore (Clone-safe)                   │
├─────────────────────────────────────────────────────────────┤
│  path:              Arc<PathBuf>      ← immutable, shared   │
│  index:             Arc<SkipMap<...>>← lock-free, shared    │
│  writer:            Arc<Mutex<...>>  ← write-only, shared   │
│  current_file_id:   Arc<AtomicU64>   ← lock-free counter    │
│  reader:            REMOVED          ← open fresh each time │
└─────────────────────────────────────────────────────────────┘
```

#### The New Lock-Free Structure

```
use std::sync::atomic::{AtomicU64, Ordering};
use crossbeam::sync::SkipMap;

#[derive(Clone)]
pub struct KvStore {
    path: Arc<PathBuf>,                              // Immutable after creation
    index: Arc<SkipMap<String, LogPointer>>,        // Lock-free concurrent map
    writer: Arc<Mutex<WriterState>>,                // Serialize writes only
    current_file_id: Arc<AtomicU64>,                // Lock-free counter
}
```

**Why each field is different:**

| Field | Type | Why |
| --- | --- | --- |
| `path` | `Arc<PathBuf>` | Never changes after init |
| `index` | `Arc<SkipMap>` | Lock-free reads & writes |
| `writer` | `Arc<Mutex<...>>` | Only one writer at a time |
| `file_id` | `Arc<AtomicU64>` | Lock-free increment |

---

### Chapter 8: The Lock-Free Read Path

This is the **crown jewel**—reads that never block.

#### The Problem with Shared Readers

```
OLD: Shared BufReader
     
     ┌──────────┐      ┌─────────────┐
     │ Reader 1 │─────►│  BufReader  │◄── Needs &mut for seek()
     └──────────┘      └──────┬──────┘
     ┌──────────┐            │
     │ Reader 2 │────────────┘
     └──────────┘
     
     Problem: One seek() blocks everyone!
```

#### The Solution: Own Your Own Handle

```
NEW: Fresh file handle per read
     
     ┌──────────┐      ┌─────────────┐
     │ Reader 1 │─────►│ File (own)  │◄── Independent seek()
     └──────────┘      └─────────────┘
     ┌──────────┐      ┌─────────────┐
     │ Reader 2 │─────►│ File (own)  │◄── Independent seek()
     └──────────┘      └─────────────┘
     
     ✓ No sharing = No blocking
```

#### Lock-Free Get Implementation

```
impl KvStore {
    pub fn get(&self, key: String) -> Result<Option<String>> {
        // Step 1: Get pointer from SkipMap (lock-free!)
        let log_ptr = match self.index.get(&key) {
            Some(ptr) => ptr,
            None => return Ok(None),  // Key doesn't exist
        };
        
        // Step 2: Open fresh file handle (no sharing!)
        let path = self.path.join(format!("{}.log", log_ptr.file_id));
        let file = OpenOptions::new().read(true).open(path)?;
        let mut reader = BufReader::new(file);
        
        // Step 3: Seek to position and read
        reader.seek(SeekFrom::Start(log_ptr.offset))?;
        
        let cmd: Command = serde_json::from_reader(reader)?;
        
        match cmd {
            Command::Set { value, .. } => Ok(Some(value)),
            Command::Remove { .. } => Ok(None),
        }
    }
}
```

**The Timeline:**

```
Lock-Free Reads in Action:

Thread 1 (Reader):  [===get(key_a)===........]
Thread 2 (Reader):  [===get(key_b)===........]  ✓ Never blocked!
Thread 3 (Writer):  [===set(key_c)===........]
Thread 4 (Reader):  [===get(key_a)===........]  ✓ Still works!

No locks held during reads = maximum parallelism
```

---

### Chapter 9: The SkipMap Deep Dive

HashMap is great, but **not thread-safe**. We need a concurrent map that supports **lock-free reads**.

#### Why SkipMap?

```
┌────────────────────┬─────────────────────┐
│      HashMap       │      SkipMap        │
├────────────────────┼─────────────────────┤
│ Not thread-safe    │ Lock-free reads     │
│ Needs external lock│ Lock-free writes    │
│ O(1) average       │ O(log n)            │
│ No ordering        │ Range queries       │
└────────────────────┴─────────────────────┘
```

#### The SkipMap API Quirk

```
// HashMap style:
let old = hashmap.insert(key, value);  // Returns old value
let old = hashmap.get(&key);            // Returns &value

// SkipMap style:
let entry = skipmap.insert(key, value); // Returns Entry (the NEW value!)
let old = skipmap.get(&key);            // But we need the OLD value...

// Workaround: get first, then insert
let old_value = skipmap.get(&key);
skipmap.insert(key, new_pointer);
if let Some(old) = old_value {
    // Use old pointer for cleanup
}
```

---

### Chapter 10: Compaction with Lock-Free Readers

Here's the tricky part: **compaction changes file pointers while readers are accessing them**.

#### The Race Condition

```
Scenario:
┌──────────────────────────────────────────────────────────────┐
│ Reader:                                                       │
│   1. index.get("key") → LogPointer { file_id: 1, offset: 0 }│
│                              ▲                               │
│                              │ Pointer points to file:1      │
│                                                              │
│ Compactor:                                                   │
│   2. Write compacted data to file:2                         │
│   3. index.insert("key", LogPointer { file_id: 2, ... })    │
│   4. delete("1.log")  ← OOPS! Reader was about to read this │
└──────────────────────────────────────────────────────────────┘
```

#### The Fix: Collect Before Updating

```
impl KvStore {
    pub fn compact(&mut self) -> Result<()> {
        // Step 1: Collect OLD file IDs BEFORE modifying index
        let mut old_files: Vec<u64> = Vec::new();
        {
            let index = self.index.read().unwrap();
            for (_key, ptr) in index.iter() {
                if !old_files.contains(&ptr.file_id) {
                    old_files.push(ptr.file_id);
                }
            }
        }
        
        // Step 2: Write compacted file
        let compaction_file_id = self.current_file_id.fetch_add(1);
        let compacted_path = self.path.join(format!("{}.log", compaction_file_id));
        
        // ... write only live entries to new file ...
        
        // Step 3: Update all pointers to new file
        for (_key, ptr) in self.index.iter() {
            ptr.file_id = compaction_file_id;
            // update offsets too...
        }
        
        // Step 4: NOW delete old files (safe!)
        for file_id in old_files {
            let path = self.path.join(format!("{}.log", file_id));
            fs::remove_file(path)?;
        }
        
        Ok(())
    }
}
```

**Why this works:** We collected file IDs that **existed when compaction started**. New files created during compaction aren't collected.

---

### Chapter 11: Write Serialization

Even though reads are lock-free, **writes must be serialized**. Only one writer at a time.

#### The Writer State

```
struct WriterState {
    writer: BufWriter<File>,
    current_file_id: u64,
    bytes_in_current_file: u64,
}

impl KvStore {
    pub fn set(&self, key: String, value: String) -> Result<()> {
        // Serialize access to writer
        let mut writer = self.writer.lock().unwrap();
        
        // Prepare command
        let cmd = Command::Set { key: key.clone(), value: value.clone() };
        let serialized = serde_json::to_string(&cmd)?;
        
        // Write to log
        let offset = writer.bytes_in_current_file;
        writeln!(writer.writer, "{}", serialized)?;
        writer.writer.flush()?;
        
        // Update index with new pointer
        let ptr = LogPointer {
            file_id: writer.current_file_id,
            offset,
            length: serialized.len() as u64,
        };
        self.index.insert(key, ptr);
        
        Ok(())
    }
}
```

---

## Section IV: The Complete Architecture

### Final: How It All Fits Together

```
┌─────────────────────────────────────────────────────────────────┐
│                      Client Applications                         │
│   ┌──────────┐       ┌──────────┐       ┌──────────┐          │
│   │  App 1   │       │  App 2   │       │  App N   │          │
│   └────┬─────┘       └────┬─────┘       └────┬─────┘          │
│        │                  │                  │                │
│        ▼                  ▼                  ▼                │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                   Network Layer (TCP)                    │  │
│   │  Request: {Set|Get|Remove} ────► Response: {Ok|Err}   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │              Thread Pool (P)                             │  │
│   │   ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐      │  │
│   │   │Worker 1│  │Worker 2│  │Worker 3│  │Worker N│      │  │
│   │   └────────┘  └────────┘  └────────┘  └────────┘      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │            KvsEngine Trait (E)                           │  │
│   │   fn set(key, value) │ fn get(key) │ fn remove(key)    │  │
│   └─────────────────────────────────────────────────────────┘  │
│                    │                         │                  │
│                    ▼                         ▼                  │
│   ┌─────────────────────────┐  ┌────────────────────────────┐  │
│   │      KvStore           │  │     SledKvsEngine          │  │
│   │  ┌─────────────────┐   │  │                            │  │
│   │  │ path: Arc<PathBuf>   │  │                            │  │
│   │  ├─────────────────┤   │  │                            │  │
│   │  │ index: Arc<SkipMap> │  │    (sled::Db)              │  │
│   │  ├─────────────────┤   │  │                            │  │
│   │  │ writer: Arc<Mutex<>>│  │                            │  │
│   │  ├─────────────────┤   │  │                            │  │
│   │  │ file_id: AtomicU64 │  │                            │  │
│   │  └─────────────────┘   │  │                            │  │
│   │  (Lock-Free Reads!)     │  │                            │  │
│   └─────────────────────────┘  └────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

### Epilogue: What We've Built

**Part 1 recap (local store):**

* Append-only log for durability
* In-memory index for fast lookups
* Log compaction for space reclamation

**Part 2 addition (networked, concurrent):**

* TCP client-server protocol with JSON serialization
* Thread pool abstraction for concurrent request handling
* Engine trait for swapping implementations (KvStore ↔ Sled)
* Lock-free reads using SkipMap and per-read file handles
* Field-level synchronization instead of global locks
* Atomic counters for lock-free file ID management

**The result:** A distributed key-value store that:

* ✓ Serves multiple concurrent clients
* ✓ Supports multiple storage engines
* ✓ Reads never block other reads
* ✓ Writes are properly serialized
* ✓ Compacts without disrupting readers

---

**Next in Part 3:** We'll explore **distributed consensus**, **replication**, and **the Raft protocol** for building truly fault-tolerant distributed systems.

---

*Building a Distributed Key Value Store - Part 2*  
[Part 1: The Persistence Journey](/aryavarth1) | Part 2  
[GitHub](https://github.com/ygndotgg/kvs_store)