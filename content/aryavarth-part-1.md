+++
title = "Aryavarth: Distributed Key Value Store Part 1"
date = 2025-02-05
[taxonomies]
tags = ["rust", "distributed-systems", "storage"]
categories = ["rust", "systems"]
+++

# Aryavarth: Distributed Key Value Store - Part 1

*"Just use Redis!" everyone tells you. "Why reinvent the wheel?"*

That's what everyone says when you mention building your own storage engine. The world is full of databases—Redis, PostgreSQL, MongoDB, Cassandra. Why would anyone in their right mind build yet another key-value store from scratch?

But honestly, they are WRONG. Understanding how storage engines work under the hood is fundamental to becoming a better engineer. You can't optimize what you don't understand.

NOTE: This is Part 1 of a technical deep-dive into Aryavarth, a distributed key-value store being built from first principles in Rust. This part covers the foundational single-node storage engine (KVS1). For the complete code implementation, see [src/kvs.rs](https://github.com/ygndotgg/kvs_store). Check the commit history for the actual implementation details. We will cover append-only logs, in-memory indexes, compaction strategies, and the architecture of persistent storage.

## What This Post Covers

* **[Part I: The Foundation](/aryavarth-part-1/#part-i-the-foundation)** - Understanding log-structured storage and why append-only is revolutionary
* **[Part II: The Index Problem](/aryavarth-part-1/#part-ii-the-index-problem)** - Building an in-memory index that makes reads fast
* **[Part III: The Compaction Challenge](/aryavarth-part-1/#part-iii-the-compaction-challenge)** - Managing disk space without losing data
* **[Part IV: The Complete Architecture](/aryavarth-part-1/#part-iv-the-complete-architecture)** - Putting it all together in Rust

---

## Prologue: Where Does it All Started

It started with a benchmark.

A few months ago, I took on the 1BRC (One Billion Row Challenge) in Rust. The goal was simple: process one billion temperature readings and compute statistics like mean, min, and max per station—as fast as possible.

Building that taught me something profound: **the foundation of every high-performance system is understanding your data layout and access patterns**.

The ceiling I hit was this: my 1BRC implementation could process data blazingly fast locally, but what if I wanted to distribute this across multiple machines? What if the dataset was too large for a single machine?

That's when it hit me—I needed distributed storage. And to understand distributed storage, I needed to understand local storage first.

So I started with the simplest possible storage mechanism: a log-structured key-value store called KVS1.

But here's the thing—I didn't want just any key-value store. I wanted one that taught me something. That's why I chose log-structured storage.

---

## PART I: The Foundation of Log-Structured Storage

### Chapter 1: The Append-Only Problem

Let me ask you a question: When you store data, what's the most fundamental operation?

It's not reading. Reading is easy—you just look up where you put something.

The hard part is writing. Specifically, writing efficiently while maintaining durability.

Here's the naive approach most people take: Open a file, seek to the right position, overwrite the data. This is called **random-write storage**, and it's deceptively problematic.

#### The Random-Write Problem

Imagine you're building a database. You have a file with existing data:

```
Position: 0    100   200   300   400   500
          ┌─────┬─────┬─────┬─────┬─────┐
Data:     │ A   │ B   │ C   │ D   │ E   │
          └─────┴─────┴─────┴─────┴─────┘
```

Now you want to update key "C" (at position 200) with new data that's larger than the original. What do you do?

```
Options:
1. Overwrite in place - OVERFLOW! Data spills into D's space
2. Seek to end - But now C is at wrong position
3. Rewrite entire file - Expensive!
```

This is the **update-in-place problem**. It's why traditional databases use complex structures like B-trees—they need to manage these overflows, which adds massive complexity.

#### The Solution: Append-Only Logs

Here's where log-structured storage changes everything. What if we never modified existing data? What if we only ever appended new data?

```
┌─────────────────────────────────────────────────────────┐
│         APPEND-ONLY LOG STRATEGY                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Time 1: SET key1="hello"                             │
│  ┌─────────────────────────────────────┐               │
│  │ {"cmd":"set","key":"key1","value":"hello"}         │
│  └─────────────────────────────────────┘               │
│                                                         │
│  Time 2: SET key2="world"                             │
│  ┌─────────────────────────────────────┐               │
│  │ {"cmd":"set","key":"key1","value":"hello"}         │
│  │ {"cmd":"set","key":"key2","value":"world"}         │
│  └─────────────────────────────────────┘               │
│                                                         │
│  Time 3: SET key1="updated" (UPDATE!)                  │
│  ┌─────────────────────────────────────┐               │
│  │ {"cmd":"set","key":"key1","value":"hello"}         │
│  │ {"cmd":"set","key":"key2","value":"world"}         │
│  │ {"cmd":"set","key":"key1","value":"updated"}       │
│  └─────────────────────────────────────┘               │
│                                                         │
│  Note: Old "key1" entry is NOT modified, just ignored  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

This is revolutionary. Here's why:

1. **Writes are always sequential** - No seeking, no random I/O. Just append to the end.
2. **No corruption from partial writes** - If the system crashes mid-write, the old data is still intact.
3. **Crash recovery is trivial** - Just replay the log from the beginning.

The tradeoff? Over time, your log grows forever with stale data. But that's a problem we'll solve with compaction.

---

### Chapter 2: The Language-Agnostic Core

Let me show you the core concepts in any language. Here's the pseudocode for a log-structured key-value store:

```
class LogStore:
    def __init__(self, directory):
        self.directory = directory
        self.in_memory_index = {}  # Maps key -> (file_id, offset, length)
        self.current_file = 0
        self.open_file()
    
    def open_file(self):
        # Open next available log file for appending
        self.file = open(f"{directory}/{current_file}.log", "a")
    
    def set(self, key, value):
        # Serialize the command as JSON
        entry = JSON.stringify({
            "cmd": "set",
            "key": key,
            "value": value
        })
        
        # Get current position (where we're appending)
        offset = self.file.tell()
        
        # Write the entry
        self.file.write(entry + "\n")
        self.file.flush()
        
        # Update our in-memory index
        self.in_memory_index[key] = {
            "file_id": self.current_file,
            "offset": offset,
            "length": len(entry)
        }
        
        # Check if we need to rotate files
        if self.file.size > MAX_SIZE:
            self.current_file += 1
            self.open_file()
    
    def get(self, key):
        # Look up where the data is
        location = self.in_memory_index.get(key)
        if not location:
            return None  # Key doesn't exist
        
        # Read the specific bytes from the file
        with open(f"{directory}/{location.file_id}.log") as f:
            f.seek(location.offset)
            data = f.read(location.length)
            entry = JSON.parse(data)
            return entry["value"]
```

This is language-agnostic. You could implement this in Python, Go, JavaScript, or any language. The concepts are universal.

Now let me show you how this translates to actual Rust code. For the complete implementation, see `src/kvs.rs`.

---

## PART II: The Index Problem

### Chapter 3: The Memory Index Architecture

Here's a critical insight: **the log stores data, but the index makes it fast**.

Without an index, to find a key, you'd have to scan through the entire log from the beginning. With millions of entries, that's unacceptable.

We need an in-memory index that maps keys to their locations in the log files.

```
┌─────────────────────────────────────────────────────────────────┐
│                    DUAL-LAYER STORAGE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   LAYER 1: In-Memory Index (HashMap)                          │
│   ┌─────────────────────────────────────────────────────┐      │
│   │  key1  ──▶ {file_id: 0, offset: 0,   len: 45}       │      │
│   │  key2  ──▶ {file_id: 0, offset: 45,  len: 38}       │      │
│   │  key3  ──▶ {file_id: 1, offset: 0,   len: 52}       │      │
│   │  key4  ──▶ {file_id: 1, offset: 52,  len: 29}       │      │
│   └─────────────────────────────────────────────────────┘      │
│                         │                                       │
│                         ▼                                        │
│   LAYER 2: Disk Storage (Log Files)                            │
│   ┌──────────────────────┐  ┌──────────────────────┐          │
│   │   0.log              │  │   1.log              │          │
│   │ ┌─────────────────┐ │  │ ┌─────────────────┐ │          │
│   │ │{set,key1,hello} │ │  │ │{set,key3,foo}   │ │          │
│   │ │{set,key2,world} │ │  │ │{set,key4,bar}   │ │          │
│   │ └─────────────────┘ │  │ └─────────────────┘ │          │
│   └──────────────────────┘  └──────────────────────┘          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

This architecture gives us:

* **O(1) lookups** - The HashMap provides instant access to any key
* **Sequential writes** - All writes go to the end of the current log file
* **Crash safety** - The log is the source of truth; the index is just a cache

### Chapter 4: Implementation in Rust

Now let's look at the actual Rust implementation. Here's how the index and log work together:

```
// The in-memory index entry - tracks where data lives on disk
pub struct LogPointer {
    offset: u32,    // Byte offset within the file
    len: u32,       // Length of the entry in bytes
    file_id: u32,   // Which log file contains this entry
}

// The main key-value store
pub struct KvStore {
    store: HashMap<String, LogPointer>,  // In-memory index
    dir: PathBuf,                         // Working directory
    current_file_id: u32,                 // Current log file ID
    writer: BufWriter<File>,              // Buffered writer for performance
    uncompacted_bytes: u32,               // Track stale data for compaction
}
```

> **Note on thresholds**: For testing purposes, I've used small threshold values in the implementation. The `MAX_FILE_SIZE` is set to 1004 bytes (~1KB) and compaction triggers at 2KB of stale data. In production, you'd use much larger values (e.g., 1GB for file rotation, 256MB for compaction).

The magic happens in the `set` method:

```
pub fn set(&mut self, key: String, value: String) -> Result<()> {
    // Check if current log file is too big - rotate if needed
    let file_path = self.dir.join("logs").join(format!("{}.log", self.current_file_id));
    let flen = OpenOptions::new()
        .read(true)
        .open(&file_path)?
        .metadata()?
        .len();

    if flen > MAX_FILE_SIZE {
        self.writer.flush()?;
        self.current_file_id += 1;
        // Create new log file and update writer
    }

    // Serialize the command as JSON
    let ser = serde_json::to_string(&Command::Set {
        key: key.clone(),
        value,
    }).unwrap();
    
    // Get current offset (where we'll write)
    let offset = self.writer.seek(std::io::SeekFrom::End(0))? as u32;
    
    // Actually write to the log
    writeln!(self.writer, "{}", ser)?;
    self.writer.flush()?;

    // Update the in-memory index with the new location
    self.store.insert(
        key,
        LogPointer {
            offset,
            len: (ser.len() + 1) as u32,  // +1 for newline
            file_id: self.current_file_id,
        },
    );

    Ok(())
}
```

This is elegant in its simplicity. Every write:

1. Checks if we need to rotate log files
2. Appends the JSON command to the log
3. Updates the in-memory index

The index is always in sync with the latest writes because we update it immediately after writing.

---

### Chapter 5: The GET Operation

Reading is where the index shines. Here's how it works:

```
pub fn get(&mut self, key: String) -> Result<Option<String>> {
    // O(1) lookup in the hash map
    let file_loc = self.store.get(&key).unwrap_or_else(|| {
        println!("Key not found");
        std::process::exit(0);
    });
    
    // Read only the specific bytes we need
    let mut buf = vec![0u8; file_loc.len as usize];
    let mut file = OpenOptions::new().read(true).open(
        self.dir.join("logs").join(format!("{}.log", file_loc.file_id)),
    )?;
    file.seek(std::io::SeekFrom::Start(file_loc.offset as u64))?;
    file.read_exact(&mut buf)?;
    
    // Parse and extract the value
    let cmd: Command = serde_json::from_slice(&buf).unwrap();
    match cmd {
        Command::Set { value, .. } => return Ok(Some(value)),
        _ => Ok(None),
    }
}
```

Notice we:

1. Look up the location in O(1) time
2. Only read the exact bytes we need (not the whole file)
3. Parse just the JSON we retrieved

This is dramatically faster than scanning the entire log.

---

## PART III: The Compaction Challenge

### Chapter 6: The Stale Data Problem

Here's the fundamental problem with append-only logs: they grow forever.

```
┌─────────────────────────────────────────────────────────────────┐
│                 LOG GROWTH OVER TIME                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  After 1 day:                                                  │
│  ┌─────────────────────────────────────┐                        │
│  │ {set,"user:1","Alice"}             │                        │
│  │ {set,"user:2","Bob"}               │                        │
│  │ {set,"user:3","Charlie"}           │  1 MB                │
│  └─────────────────────────────────────┘                        │
│                                                                 │
│  After 1 week (with updates):                                   │
│  ┌─────────────────────────────────────┐                        │
│  │ {set,"user:1","Alice"}             │ ◄── STALE             │
│  │ {set,"user:2","Bob"}               │                        │
│  │ {set,"user:3","Charlie"}           │ ◄── STALE             │
│  │ {set,"user:1","Alice Updated"}    │                        │
│  │ {set,"user:4","Diana"}             │                        │
│  │ {set,"user:3","New Charlie"}      │ ◄── STALE             │
│  │ {set,"user:5","Eve"}               │  5 MB                │
│  └─────────────────────────────────────┘                        │
│                                                                 │
│  Actual live data: only 3 keys!                                  │
│  Wasted space: 60%!                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

In the example above:

* We have 5 log entries
* But only 3 are "live" (the latest version of each key)
* 2 entries are stale (older versions that were updated)

As the system runs, stale data accumulates until most of your disk is wasted.

### Chapter 7: The Compaction Solution

Compaction solves this. The idea is simple: periodically create a new log file containing only the live (latest) data, then delete the old files.

Here's the algorithm:

```
┌─────────────────────────────────────────────────────────────────┐
│                      COMPACTION PROCESS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE COMPACTION:                                            │
│  ┌─────────────┐  ┌─────────────┐                              │
│  │ 0.log       │  │ 1.log       │                              │
│  │ A=1         │  │ A=2         │  ← Latest A is here          │
│  │ B=1         │  │ C=1         │  ← Latest C is here          │
│  │ D=1         │  │             │                              │
│  └─────────────┘  └─────────────┘                              │
│  In-memory index: A→1.log, B→0.log, C→1.log, D→0.log          │
│                                                                 │
│  STEP 1: Create new compact file                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ 0.log       │  │ 1.log       │  │ 2.log (NEW)│             │
│  │ A=1         │  │ A=2  ✓      │  │ A=2         │             │
│  │ B=1    ✓    │  │ C=1    ✓    │  │ B=1         │             │
│  │ D=1    ✓    │  │             │  │ C=1         │             │
│  └─────────────┘  └─────────────┘  │ D=1         │             │
│                                   └─────────────┘             │
│  STEP 2: Rewrite only live entries                            │
│                                                                 │
│  STEP 3: Update index                                         │
│  In-memory index: A→2.log, B→2.log, C→2.log, D→2.log         │
│                                                                 │
│  STEP 4: Delete old files                                     │
│  ┌─────────────┐                                               │
│  │ 2.log       │  1 MB → 0.5 MB (50% reduction!)             │
│  │ A=2         │                                               │
│  │ B=1         │                                               │
│  │ C=1         │                                               │
│  │ D=1         │                                               │
│  └─────────────┘                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Chapter 8: The Rust Compaction Implementation

Here's how KVS1 implements compaction:

```
pub fn compact(&mut self) -> Result<()> {
    // Create new compact file with ID after current
    let compact_file_id = self.current_file_id + 1;
    let next_file_id = self.current_file_id + 2;
    
    // Collect file IDs that are still referenced
    let old_fileids: HashSet<u32> = 
        self.store.iter().map(|entry| entry.1.file_id).collect();
    
    // Create the compact file
    let compact_file = OpenOptions::new()
        .create(true)
        .read(true)
        .write(true)
        .open(self.dir.join("logs").join(format!("{}.log", compact_file_id)))?;
    let mut compact_writer = BufWriter::new(compact_file);
    
    // Rewrite only live entries
    let mut offset = 0;
    for ptr in self.store.values_mut() {
        // Read the old entry
        let mut curr_file = File::open(
            self.dir.join("logs").join(format!("{}.log", ptr.file_id))
        )?;
        let mut buf = vec![0u8; ptr.len as usize];
        curr_file.seek(std::io::SeekFrom::Start(ptr.offset as u64))?;
        curr_file.read_exact(&mut buf)?;
        
        // Write to compact file
        compact_writer.write_all(&buf)?;
        compact_writer.flush()?;
        
        // Update the pointer to new location
        ptr.offset = offset;
        ptr.len = buf.len() as u32;
        ptr.file_id = compact_file_id;
        offset += buf.len() as u32;
    }
    
    // Delete old files that are no longer needed
    for file in old_fileids {
        let path = self.dir.join("logs").join(format!("{}.log", file));
        std::fs::remove_file(path)?;
    }
    
    // Set up writer for next file
    let next_file = OpenOptions::new()
        .create(true)
        .read(true)
        .write(true)
        .open(self.dir.join("logs").join(format!("{}.log", next_file_id)))?;
    self.writer = BufWriter::new(next_file);
    self.current_file_id = next_file_id;
    self.uncompacted_bytes = 0;
    
    Ok(())
}
```

The compaction is triggered automatically when too much stale data accumulates:

```
// After every write, check if we need compaction
self.uncompacted_bytes += ser.len() as u32;
if self.uncompacted_bytes > 1024 * 2 {  // 2KB threshold (small for testing)
    self.compact()?;
}
```

---

## PART IV: The Complete Architecture

### Chapter 9: The System Architecture

Here's how all the pieces fit together:

```
┌─────────────────────────────────────────────────────────────────┐
│                        KVS1 ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                         CLI Interface                          │
│                     ┌─────────────────┐                        │
│                     │ cargo run set   │                        │
│                     │    key value    │                        │
│                     └────────┬────────┘                        │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    KvStore Engine                        │  │
│   │                                                         │  │
│   │  ┌─────────────┐    ┌─────────────┐    ┌────────────┐  │  │
│   │  │    SET      │    │    GET      │    │    RM      │  │  │
│   │  │  (write)    │    │   (read)    │    │ (delete)   │  │  │
│   │  └──────┬──────┘    └──────┬──────┘    └──────┬─────┘  │  │
│   │         │                 │                 │        │  │
│   │         ▼                 │                 │        │  │
│   │  ┌─────────────────────────────────────────┐ │        │  │
│   │  │         In-Memory Index (HashMap)       │ │        │  │
│   │  │  key1 → {file: 0, offset: 0, len: 45}  │ │        │  │
│   │  │  key2 → {file: 0, offset: 45, len: 38} │◀┘        │  │
│   │  │  key3 → {file: 1, offset: 0, len: 52}  │◀┴────────┘  │  │
│   │  └─────────────────────────────────────────┘            │  │
│   │                         │                               │  │
│   │                         ▼                               │  │
│   │  ┌─────────────────────────────────────────────────┐    │  │
│   │  │            Log Manager                           │    │  │
│   │  │  • Append-only writes                           │    │  │
│   │  │  • File rotation (1KB chunks - testing)         │    │  │
│   │  │  • Compaction when stale > 2KB                  │    │  │
│   │  └─────────────────────────────────────────────────┘    │  │
│   │                         │                               │  │
│   └─────────────────────────┼───────────────────────────────┘  │
│                             │                                   │
│                             ▼                                   │
│                    ┌────────────────┐                          │
│                    │   ./logs/       │                          │
│                    │  ├── 0.log     │                          │
│                    │  ├── 1.log     │                          │
│                    │  └── 2.log     │                          │
│                    └────────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Chapter 10: Handling Deletes

What happens when you delete a key? We don't actually remove the data immediately—that would require random writes. Instead, we append a "tombstone" marker:

```
pub fn remove(&mut self, key: String) -> Result<()> {
    match self.store.get(&key) {
        Some(lg) => {
            // Write a "remove" command to the log
            let ser = serde_json::to_string(&Command::Rm { key: key.clone() }).unwrap();

            let mut file = OpenOptions::new()
                .append(true)
                .open(self.dir.join("logs").join(format!("{}.log", lg.file_id)))?;
            
            match writeln!(file, "{}", ser) {
                Ok(_) => {
                    // Remove from in-memory index
                    self.store.remove(&key);
                    println!("Removed {}", key);
                }
                Err(e) => {
                    eprintln!("{e}");
                }
            }
            
            // Track for compaction
            self.uncompacted_bytes += ser.len() as u32;
            if self.uncompacted_bytes > 1024 * 2 {
                self.compact()?;
            }
        }
        None => {
            println!("Key not found");
            process::exit(0);
        }
    }

    Ok(())
}
```

The delete marker tells compaction to exclude this key when rewriting the log. The old entry becomes "stale" just like an updated key.

### Chapter 11: The Opening Process (Bootstrapping)

When KVS1 starts up, it needs to rebuild the in-memory index from the log files. This is called "replay" or "recovery":

```
pub fn open(dir: PathBuf) -> Result<KvStore> {
    // Create logs directory if it doesn't exist
    std::fs::create_dir_all(dir.join("logs"))?;
    
    let mut files = Vec::new();
    let mut store = HashMap::new();
    let mut uncompacted_bytes: u32 = 0;
    
    // Scan existing log files
    if let Ok(entry) = read_dir(dir.join("logs")) {
        for i in entry {
            if let Ok(diry) = i {
                if let Some(pt) = diry.file_name().into_string().ok() {
                    // Parse file IDs from filenames like "0.log", "1.log"
                    let ck = pt.trim_end_matches(".log").parse::<u32>();
                    if let Ok(ck) = ck {
                        files.push(ck);
                    }
                }
            }
        }
        files.sort();

        // Replay each file in order
        for file in files.iter() {
            let d = OpenOptions::new()
                .create(true)
                .read(true)
                .append(true)
                .open(dir.join("logs").join(format!("{}.log", file)))?;
            
            let mut reader = BufReader::new(d);
            let mut offset = 0;
            
            loop {
                let mut line = String::new();
                let bytes_read = reader.read_line(&mut line)?;
                
                if bytes_read == 0 {
                    break;
                }
                
                // Parse and apply each command
                let cmd: Command = serde_json::from_str(&line).unwrap();
                match cmd {
                    Command::Rm { key } => {
                        // Delete removes from index
                        store.remove(&key);
                        uncompacted_bytes += bytes_read as u32;
                    }
                    Command::Get { .. } => {}  // Skip read commands
                    Command::Set { key, .. } => {
                        // Set updates the index to point to latest location
                        store.insert(
                            key,
                            LogPointer {
                                offset,
                                len: bytes_read as u32,
                                file_id: *file,
                            },
                        );
                        uncompacted_bytes += bytes_read as u32;
                    }
                }
                offset += bytes_read as u32;
            }
        }
    }
    
    // Open current log file for writing
    let current_file_id = files.last().copied().unwrap_or(0);
    let path = dir.join("logs").join(format!("{}.log", current_file_id));
    let wrifile = OpenOptions::new()
        .create(true)
        .write(true)
        .read(true)
        .append(true)
        .open(&path)?;
    
    Ok(KvStore {
        store,
        dir,
        writer: BufWriter::new(wrifile),
        current_file_id,
        uncompacted_bytes,
    })
}
```

This is the beauty of log-structured storage: **recovery is simple**. Just replay the log from beginning to end, and you get the exact same state you had before the crash. No complex recovery procedures needed.

---

### Chapter 12: The Command System

KVS1 uses a simple command enum that's serialized to JSON:

```
#[derive(Subcommand, Deserialize, Serialize, Debug)]
pub enum Command {
    Get { key: String },
    Set { key: String, value: String },
    Rm { key: String },
}
```

This makes the log human-readable and easy to debug. Each line in the log file is a valid JSON object:

```
{"key":"username","value":"john","cmd":"Set"}
{"key":"email","value":"john@example.com","cmd":"Set"}
{"key":"username","cmd":"Rm"}
{"key":"username","value":"jane","cmd":"Set"}
```

---

### Chapter 13: Project Structure and Usage

The KVS1 project structure:

```
dummy_kvs1/
├── Cargo.toml           # Rust package definition
├── src/
│   ├── main.rs         # CLI entry point with command parsing
│   ├── lib.rs          # Library root with error types
│   └── kvs.rs          # Core KvStore implementation
├── logs/               # Runtime log files (created at runtime)
│   ├── 0.log
│   ├── 1.log
│   └── ...
└── target/             # Compiled artifacts
```

The CLI uses `clap` for argument parsing:

```
#[derive(Parser)]
#[command(about, version)]
struct Cli {
    #[command(subcommand)]
    cmd: Command,
}

fn main() -> Result<()> {
    let cli = Cli::parse();
    let dir = env::current_dir()?;
    let mut kvs = KvStore::open(dir)?;
    
    match cli.cmd {
        Command::Get { key } => {
            let ans = kvs.get(key)?;
            if let Some(ans) = ans {
                println!("Found {}", ans);
            }
        }
        Command::Set { key, value } => {
            kvs.set(key, value)?;
        }
        Command::Rm { key } => {
            kvs.remove(key)?;
        }
    };
    Ok(())
}
```

Usage:

```
# Set a key-value pair
cargo run -- set username john

# Get a value
cargo run -- get username

# Delete a key
cargo run -- rm username
```

---

## Epilogue: The Road Ahead

Building KVS1 taught me things no tutorial could.

When I did the 1BRC challenge, I learned about SIMD, memory layouts, hash maps, and data processing. But I always felt like I was working with ephemeral data—everything disappeared when the program ended.

With KVS1, I built persistent storage from scratch. Now I understand:

1. **Why Redis is so fast** - It's essentially an in-memory index with optional persistence
2. **How PostgreSQL recovers from crashes** - It replays the WAL (write-ahead log)
3. **Why SSDs matter for databases** - Random writes were the bottleneck; now they're faster
4. **The true cost of "simple" operations** - Every feature has tradeoffs

But this is just Part 1.

The next part of Aryavarth will cover:

* **Network layer** - Adding RPC capabilities to communicate between nodes
* **Sharding** - Distributing keys across multiple machines
* **Replication** - Making data durable across failures
* **Consistency** - Handling concurrent writes and conflicts

The goal is to build a truly distributed key-value store that can scale across multiple machines while maintaining consistency and performance.

*Aryavarth: Building a distributed key-value store*  
[Part 1](/aryavarth1) | [Part 2: The Networking Layer & Lock-Free Concurrency](/aryavarth2)  
[GitHub](https://github.com/ygndotgg/kvs_store)