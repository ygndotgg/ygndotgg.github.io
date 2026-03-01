+++
title = "brrrc: Processing One Billion Rows in 3.29 Seconds"
date = 2025-01-23
[taxonomies]
tags = ["rust", "performance", "optimization"]
categories = ["programming"]
+++

# brrrc: Processing One Billion Rows in 3.29 Seconds

"Just use BufferedReader and parse line by line - it'll be fast enough."

That is what everyone tells you. Read the file, split by lines, parse temperatures, done.

But honestly, they are WRONG. When you're processing **one billion rows** - that's 13+ GB of data - "fast enough" translates to minutes, not seconds. The difference between a naive solution and an optimized one isn't 2x or 3x - it's **100x or more**.

NOTE: This is a technical deep-dive into **1brc**, a solution to the One Billion Row Challenge. We will cover memory-mapped I/O, SIMD vectorization, custom hashing, and the incremental optimization journey from ~2 minutes to ~3 seconds.

---

## What This Post Covers

**[Part I: The Naive Foundation](https://ygndotgg.github.io/1brc-blog/#part-i-the-naive-foundation)** - BufferedReader, HashMap, and why the obvious approach fails at scale

**[Part II: The Memory Revolution](https://ygndotgg.github.io/1brc-blog/#part-ii-the-memory-revolution)** - Memory-mapped I/O, kernel bypass, and the art of zero-copy

**[Part III: The Parsing Pipeline](https://ygndotgg.github.io/1brc-blog/#part-iii-the-parsing-pipeline)** - Integer arithmetic, memchr, SIMD vectorization

**[Part IV: The Architecture](https://ygndotgg.github.io/1brc-blog/#part-iv-the-architecture)** - Multi-threading, custom data structures, and the final assembly

---

## Performance Snapshot

**Hardware:**

* CPU: Intel Core Ultra 5
* RAM: 16GB
* OS: Omarchy (Arch Linux-based)

**Benchmark Results:**

* Final execution time: **~3.29 seconds**
* Input: 1 billion rows (~13GB)
* Output: Min/Max/Average for ~400 weather stations
* Flame graph: Available in repository

```
{Abha=-5.0/18.0/39.9, Abidjan=-5.0/26.0/40.0, ...}
```

---

## Prologue: The One Billion Row Problem

I've been writing Rust for about a year now. When I first heard about the **One Billion Row Challenge** - originally a Java competition to process temperature measurements at extreme scale - I was intrigued. Many senior developers participated, sharing breakthrough techniques and optimization strategies.

The challenge is deceptively hard:

* Input: A text file with **1 billion rows**
* Format: `station_name;temperature` (with duplicates)
* Output: Min, Max, and Average temperature for each unique station

```
Vijayawada;12.3
Bezawada;21.9
Vijayawada;15.1
...
```

The ceiling I hit was predictable: **naive parsing doesn't scale**. Reading 13B with buffered readers, parsing floats, allocating strings - every operation multiplied by a billion becomes a bottleneck.

Existing solutions in the challenge used sophisticated techniques: memory mapping, SIMD, custom hashers. But I wanted to **discover these optimizations myself**, starting from nothing and iterating toward performance.

This is the story of **1brc**.

More importantly, this is a manual on how to **transform naive code into high-performance systems software**.

---

## PART I: THE NAIVE FOUNDATION

### Chapter 1: The BufferedReader Trap

#### The Problem Every Beginner Makes

At billion-row scale, "readable code" and "performant code" diverge dramatically. Let's start with the most obvious solution - the one most developers would write first.

#### The Solution: BufferedReader with BTreeMap

```
// Version 1: The Initial Attempt
use std::{
    collections::BTreeMap,
    fs::File,
    io::{BufRead, BufReader},
};

pub fn main() {
    let f = File::open("measurements.txt").unwrap();
    let f = BufReader::new(f);
    let mut bmap = BTreeMap::new();
    
    for k in f.lines() {
        let k = k.unwrap();
        let (city, temp) = k.split_once(";").unwrap();
        let stats = bmap
            .entry(city.to_string())
            .or_insert((f64::MAX, 0 as usize, 0 as f64, f64::MIN));
        let temp: f64 = temp.parse().unwrap();
        stats.0 = stats.0.min(temp);
        stats.1 += 1;
        stats.2 += temp;
        stats.3 = stats.3.max(temp);
    }
    // ... print results
}
```

**What this does:**

1. Opens file with `BufReader` - allocates kernel buffer, user buffer
2. `.lines()` - allocates a new `String` for every line
3. `.to_string()` - allocates another String for the station name
4. `.parse::<f64>()` - parses to floating point, expensive operation
5. `BTreeMap` - maintains sorted order on every insert

**Estimated time:** Multiple minutes for 1 billion rows.

**Why it's slow:**

* **Allocation storm**: Billions of allocations for strings
* **Copy overhead**: `BufReader` copies from kernel buffer → user buffer
* **Float parsing**: `f64` parsing is expensive compared to integer operations
* **Sorting overhead**: BTreeMap maintains order during insertion (O(log n) per insert)

---

### Chapter 2: The HashMap Optimization

#### The Sorting Problem

The first optimization is obvious: **stop sorting on every insert**.

`BTreeMap` maintains sorted order, requiring O(log n) comparisons per insertion. At a billion rows with ~400 unique stations, that's billions of unnecessary comparisons.

#### The Solution: Switch to HashMap

```
// Version 2: HashMap for O(1) lookups
let mut hmap: HashMap<String, (f64, usize, f64, f64)> = HashMap::new();

for k in f.lines() {
    let k = k.unwrap();
    let (city, temp) = k.split_once(";").unwrap();
    let stats = match hmap.get_mut(city) {
        Some(k) => k,
        None => hmap.entry(city.to_string())
            .or_insert((f64::MAX, 0, 0.0, f64::MIN))
    };
    let temp: f64 = temp.parse().unwrap();
    stats.0 = stats.0.min(temp);
    stats.1 += 1;
    stats.2 += temp;
    stats.3 = stats.3.max(temp);
}

// Sort only once at the end
let stats = BTreeMap::from_iter(hmap);
```

**Key insight:**

* HashMap gives O(1) amortized lookup
* Sort once at the end instead of maintaining order throughout
* **Significant improvement**, but still bottlenecked by allocations

```
┌─────────────────────────────────────────────────────────────┐
│                    DATA FLOW (Naive)                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Disk ──► Kernel Buffer ──► User Buffer ──► String Alloc   │
│                    │              │               │         │
│                    ▼              ▼               ▼         │
│               memcpy()      .lines()      .to_string()      │
│                                                             │
│  Each line: 2 allocations + 1 float parse + 1 hash lookup  │
│  1 billion lines = Billions of allocations                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### Chapter 3: The String Problem

#### The Allocation Ceiling

Profiling revealed the next bottleneck: **String allocations**.

Every `.to_string()` allocates heap memory. At a billion rows, even with only ~400 unique stations, we're allocating billions of Strings, then immediately discarding most of them.

#### The Solution: Work with Bytes

```
// Version 3: Avoid String, work with &[u8]
let mut hmap: HashMap<Vec<u8>, (f64, usize, f64, f64)> = HashMap::new();

for k in f.split(b'\n') {
    let k = k.unwrap();
    // Split from the right - semicolon is near the end
    let mut k = k.rsplitn(2, |k| *k == b';');
    let temp = k.next().unwrap();
    let city = k.next().unwrap();
    
    let stats = match hmap.get_mut(city) {
        Some(k) => k,
        None => hmap.entry(city.to_vec())
            .or_insert((f64::MAX, 0, 0.0, f64::MIN))
    };
    // ... update stats
}
```

**Improvements:**

* `.split(b'\n')` - splits on bytes, no UTF-8 validation
* `.rsplitn(2, ...)` - splits from right (semicolon is near end)
* `Vec<u8>` instead of `String` - no UTF-8 checks

**Still slow because:**

* `Vec<u8>` still allocates on the heap
* `f64` parsing is still expensive
* `BufReader` still copies data

---

## PART II: THE MEMORY REVOLUTION

### Chapter 4: The Fallacy of "Just Read the File"

#### Attempt 1: BufferedReader is "Good Enough"

The `BufReader` pattern is standard:

1. Kernel reads file into kernel buffer
2. `BufReader` copies from kernel buffer to user buffer
3. `.lines()` creates String slices

**The hidden cost:** Every `memcpy` at billion-row scale is measurable.

```
┌────────────────────────────────────────────────────────────────┐
│                  TRADITIONAL I/O PATH                          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   ┌──────┐      ┌──────────┐      ┌──────────┐                │
│   │ Disk │─────►│  Kernel  │─────►│   User   │                │
│   └──────┘      │  Buffer  │      │  Buffer  │                │
│                 └──────────┘      └──────────┘                │
│                      │                 │                       │
│                      │    memcpy()     │                       │
│                      └─────────────────┘                       │
│                                                                │
│   For 100GB file: 100GB copied from kernel to user space     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

#### Attempt 2: Memory-Mapped I/O

**The insight:** What if we could read directly from kernel memory?

Memory-mapped I/O tells the operating system: "Map this file directly into my process's virtual address space."

```
fn mmap(f: &File) -> &[u8] {
    let len = f.metadata().unwrap().len() as usize;
    unsafe {
        let ptr = libc::mmap(
            ptr::null_mut(),  // Let kernel choose address
            len,              // Size of mapping
            PROT_READ,        // Read-only access
            MAP_PRIVATE,      // Copy-on-write (we're only reading)
            f.as_raw_fd(),    // File descriptor
            0,                // Start at offset 0
        );
        
        if ptr == libc::MAP_FAILED {
            panic!("{}", std::io::Error::last_os_error());
        }
        
        std::slice::from_raw_parts(ptr as *const u8, len)
    }
}
```

**What happens under the hood:**

```
┌────────────────────────────────────────────────────────────────┐
│                  MEMORY-MAPPED I/O PATH                        │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   ┌──────┐      ┌──────────────────────────────────┐           │
│   │ Disk │─────►│        Virtual Memory            │           │
│   └──────┘      │  ┌────────────────────────────┐ │           │
│                 │  │   Page Cache (Kernel)      │ │           │
│                 │  │   ┌────┐ ┌────┐ ┌────┐    │ │           │
│                 │  │   │Page│ │Page│ │Page│    │ │           │
│                 │  │   └────┘ └────┘ └────┘    │ │           │
│                 │  └────────────────────────────┘ │           │
│                 └──────────────────────────────────┘           │
│                              ▲                                 │
│                              │                                 │
│                    Direct CPU Access                          │
│                    (No memcpy!)                               │
│                              │                                 │
│                              ▼                                 │
│                 ┌──────────────────┐                           │
│                 │  Application    │                           │
│                 │  &[u8] slice    │                           │
│                 └──────────────────┘                           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**Benefits:**

1. **Zero-copy**: No `memcpy` from kernel to user buffer
2. **Lazy loading**: Pages loaded on-demand, not all at once
3. **Kernel-managed**: OS handles caching, prefetching

---

### Chapter 5: The Kernel Whisperer (madvise)

#### The Sequential Access Hint

Memory mapping is good, but we can make it **better**.

We know our access pattern: **sequential read from start to finish**. The kernel doesn't know this - it might use a random-access caching strategy.

`madvise()` tells the kernel our intentions:

```
use libc::{MADV_SEQUENTIAL, madvise};

// After mmap succeeds:
if madvise(ptr, len, MADV_SEQUENTIAL) != 0 {
    panic!("{}", std::io::Error::last_os_error());
}
```

**What MADV\_SEQUENTIAL does:**

1. **Aggressive readahead**: Kernel prefetches pages ahead
2. **Cache eviction**: Drop pages after reading (we won't revisit)
3. **Reduced memory pressure**: Don't pollute cache with old data

```
┌───────────────────────────────────────────────────────────────┐
│                  MADV_SEQUENTIAL Effect                        │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│   Without MADV_SEQUENTIAL:     With MADV_SEQUENTIAL:         │
│   ─────────────────────        ─────────────────────         │
│                                                               │
│   [1][2][3][4][5][6]           [1][2][3][4][5][6]            │
│    ▲  ▲  ▲  ▲                   ▲  ▲  ▲  ▲                   │
│    │  │  │  │                   │  │  │  │                   │
│   Keep all in cache            Read, process, evict          │
│   (wastes memory)              (efficient cache use)        │
│                                                               │
│   Cache hit on revisit         Cache hit on next chunk       │
│   (but we never revisit)       (kernel prefetches ahead)     │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

**Performance gain:** Significant. The kernel now works *with* us, not against us.

---

### Chapter 6: The Result So Far

```
// Version 4: Memory-mapped I/O
pub fn main() {
    let f = File::open("measurements.txt").unwrap();
    let map = mmap(&f);  // Zero-copy access
    
    let mut hmap: HashMap<Vec<u8>, (f64, usize, f64, f64)> = HashMap::new();
    
    for k in map.split(|a| *a == b'\n') {
        if k.is_empty() { break; }
        let mut k = k.rsplitn(2, |k| *k == b';');
        let temp = k.next().unwrap();
        let city = k.next().unwrap();
        // ... update stats
    }
}
```

**Still bottlenecked by:**

* Float parsing (`f64`)
* Heap allocations (`Vec<u8>`)
* Hash function overhead

---

## PART III: THE PARSING PIPELINE

### Chapter 7: The Float Problem

#### The Expensive f64

Parsing `12.3` to `f64` involves:

1. Validate UTF-8
2. Scan for decimal point
3. Convert each digit
4. Handle exponent, sign
5. Compute floating-point representation

**The invariant:** Challenge temperatures always have **one decimal place**.

```
12.3  → valid
-5.0  → valid
123.4 → valid
```

#### The Solution: Fixed-Point Arithmetic

Instead of `f64`, store as `i16` (multiplied by 10):

```
// 12.3  → 123
// -5.0  → -50
// 123.4 → 1234

fn parse_temperature(temp: &[u8]) -> i16 {
    let mut neg = false;
    let mut t = 0;
    let mut mul = 1;
    
    for i in temp.iter().rev() {
        match i {
            b'-' => { neg = true; break; }
            b'.' => { continue; }  // Skip decimal point
            _ => {
                t += i16::from(i - b'0') * mul;
                mul *= 10;
            }
        }
    }
    if neg { t = -t; }
    t
}
```

**Why this is faster:**

* No floating-point conversion
* Integer arithmetic is CPU-native
* No heap allocation

**Storage savings:**

* `f64`: 8 bytes per temperature
* `i16`: 2 bytes per temperature (4x smaller)

```
┌──────────────────────────────────────────────────────────────┐
│              FLOAT vs FIXED-POINT COMPARISON                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   Float Parsing:           Fixed-Point Parsing:             │
│   ─────────────            ───────────────────              │
│                                                              │
│   "12.3"                   "12.3"                           │
│      │                        │                             │
│      ▼                        ▼                             │
│   UTF-8 validate          Read byte                         │
│      │                        │                             │
│      ▼                        ▼                             │
│   Find decimal            Skip '.'                          │
│      │                        │                             │
│      ▼                        ▼                             │
│   Convert digits          Convert digits: 3*1 + 2*10 + 1*100
│      │                        │                             │
│      ▼                        ▼                             │
│   IEEE-754 encode         Store: 123                        │
│      │                                                       │
│      ▼                                                       │
│   Store: 12.300000...                                        │
│                                                              │
│   ~100+ CPU cycles         ~20 CPU cycles                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Chapter 8: The Split Problem

#### The memchr Solution

Splitting on `\n` is expensive if done byte-by-byte. Enter `memchr`:

```
use std::ffi::{c_int, c_void};

// libc::memchr is vectorized by the C library
let next_line = unsafe { 
    libc::memchr(
        rest.as_ptr() as *const c_void, 
        '\n' as c_int, 
        rest.len()
    ) 
};

let k = if next_line.is_null() {
    rest
} else {
    let len = unsafe { 
        next_line.offset_from(rest.as_ptr() as *const c_void) as usize 
    };
    &rest[..len]
};
```

**How memchr works:**

* Uses **SIMD** internally (vectorized search)
* Scans 16-64 bytes per instruction
* Much faster than byte-by-byte loop

```
┌──────────────────────────────────────────────────────────────┐
│                    memchr vs Naive Loop                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   Naive (byte-by-byte):                                      │
│   ┌─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐                         │
│   │H│e│l│l│o│\n│B│y│e│\n│...│ │ │ │ │  1 byte per check     │
│   └─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘                         │
│         ↑  6 checks to find \n                               │
│                                                              │
│   memchr (SIMD):                                             │
│   ┌────────────────┬────────────────┬────────────────┐       │
│   │ H e l l o \n B │ y e \n ...     │                │       │
│   └────────────────┴────────────────┴────────────────┘       │
│         16-64 bytes checked in ONE instruction               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Chapter 9: The SIMD Revolution

#### Why SIMD Matters

**SIMD** = Single Instruction, Multiple Data

Instead of processing 1 byte at a time, process 32 bytes at once.

```
#![feature(portable_simd)]
use std::simd::{Simd, cmp::SimdPartialEq};

fn find_newline(map: &[u8]) -> Option<usize> {
    const LANES: usize = 32;
    const SPLAT: Simd<u8, LANES> = Simd::splat(b'\n');
    
    let mut i = 0;
    while let Some((chunk, rem)) = map.split_first_chunk::<32>() {
        let bytes = Simd::<u8, LANES>::from_array(*chunk);
        
        // Compare 32 bytes to '\n' in ONE instruction
        let mask = bytes.simd_eq(SPLAT);
        
        // Find first match
        let index = mask.first_set().map(|k| k + i);
        if index.is_some() {
            return index;
        }
        i += LANES;
        map = rem;
    }
    
    // Handle remaining bytes
    let k = Simd::<u8, LANES>::load_or_default(map);
    k.simd_eq(SPLAT).first_set().map(|k| k + i)
}
```

**The key insight:**

* CPU has 128-bit, 256-bit, or 512-bit vector registers
* `u8x32` = 32 bytes in one register
* One `simd_eq` compares all 32 bytes simultaneously

```
┌──────────────────────────────────────────────────────────────┐
│                    SIMD NEWLINE DETECTION                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   Input: "Vijayawada;12.3\nBezawada;21.9\n..."              │
│                                                              │
│   Step 1: Load 32 bytes into vector register                 │
│   ┌────────────────────────────────────────┐                 │
│   │ Vijayawada;12.3\nBezawada;21.9\n...   │                 │
│   └────────────────────────────────────────┘                 │
│                      │                                       │
│                      ▼                                       │
│   Step 2: SIMD compare with '\n'                             │
│   ┌────────────────────────────────────────┐                 │
│   │ 0 0 0 0 0 0 0 0 0 0 1 0 0 0 0 0 0 0 1 │  mask          │
│   └────────────────────────────────────────┘                 │
│                      │                                       │
│                      ▼                                       │
│   Step 3: Find first set bit = index of first \n            │
│                                                              │
│   Result: Index 10 (found in 1 instruction, not 10)          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Chapter 10: The Semicolon Split

#### Optimized Backwards Search

Since temperatures are short (4-5 bytes), semicolon is near the end:

```
fn split_semi(k: &[u8]) -> (&[u8], &[u8]) {
    // Start from position len-4 (temp is ~4 chars)
    let mut pos = k.len() - 4;
    
    unsafe {
        // Search backwards for ';'
        while *k.get_unchecked(pos) != b';' {
            pos -= 1;
        }
        let (before, after) = k.split_at_unchecked(pos + 1);
        (&before[..before.len() - 1], after)
    }
}
```

**Why backwards?**

* Temperature is always 3-5 characters (e.g., `12.3`, `-5.0`, `123.4`)
* Semicolon is near the end
* Average 1-2 comparisons vs scanning entire line

---

## PART IV: THE ARCHITECTURE

### Chapter 11: The Multi-Threading Dimension

#### Parallel Processing Strategy

With ~17 CPU threads available, parallelization is essential.

**Challenge:** How do you split a file for parallel processing when lines have variable lengths?

**Solution:** Chunk at approximately equal byte offsets, then find nearest newline:

```
std::thread::scope(|scope| {
    let map = mmap(&f);
    let nthreads = available_parallelism().unwrap();
    let (tx, rx) = std::sync::mpsc::sync_channel(nthreads.get());
    
    let chunk_len = map.len() / nthreads;
    let mut at = 0;
    
    for _ in 0..nthreads.get() {
        let start = at;
        let end = (at + chunk_len).min(map.len());
        
        // Find newline to avoid splitting mid-line
        let end = if end == map.len() {
            map.len()
        } else {
            let newline_at = find_newline(&map[end..]).unwrap();
            end + newline_at + 1
        };
        
        let chunk = &map[start..end];
        at = end;
        
        let tx = tx.clone();
        scope.spawn(move || tx.send(process_chunk(chunk)));
    }
    
    drop(tx);
    // Aggregate results from all threads
    for thread_stats in rx {
        merge_stats(&mut final_stats, thread_stats);
    }
});
```

```
┌──────────────────────────────────────────────────────────────┐
│                    PARALLEL CHUNKING                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   File (13GB):                                              │
│   ┌──────────────────────────────────────────────────────┐   │
│   │ Line 1 │ Line 2 │ Line 3 │ ... │ Line 1,000,000,000 │   │
│   └──────────────────────────────────────────────────────┘   │
│        │         │         │              │                  │
│        ▼         ▼         ▼              ▼                  │
│   ┌────────┬────────┬────────┬────────┬────────┐              │
│   │Thread 1│Thread 2│Thread 3│  ...   │Thread N│              │
│   │        │        │        │        │        │              │
│   │ Find \n│ Find \n│ Find \n│        │ Find \n│              │
│   │ align  │ align  │ align  │        │ align  │              │
│   └────────┴────────┴────────┴────────┴────────┘              │
│        │         │         │              │                  │
│        ▼         ▼         ▼              ▼                  │
│   ┌─────────────────────────────────────────────────────┐    │
│   │              Aggregate Results                       │    │
│   │         (Single-threaded merge)                     │    │
│   └─────────────────────────────────────────────────────┘    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Chapter 12: The Vault (StrVec - Custom String Storage)

#### The Problem with Strings

Even `Vec<u8>` allocates heap memory for each unique station. With ~400 stations, that's 400 heap allocations.

But most station names are **short** (< 32 bytes). Can we store them inline?

#### The Solution: StrVec Union

```
const INLINE: usize = std::mem::size_of::<AllocatedStrVec>(); // 32 bytes
const LAST: usize = INLINE - 1;

#[repr(C)]
union StrVec {
    inlined: [u8; INLINE],  // Small strings: store inline
    heap: ManuallyDrop<AllocatedStrVec>,  // Large strings: heap allocate
}

#[repr(C)]
struct AllocatedStrVec {
    ptr: *mut u8,
    len: usize,
}

impl StrVec {
    pub fn new(s: &[u8]) -> Self {
        if s.len() < INLINE {
            // Inline storage: copy into union
            let mut combined = [0u8; INLINE];
            combined[..s.len()].copy_from_slice(s);
            combined[LAST] = s.len() as u8 + 1;  // Length marker
            Self { inlined: combined }
        } else {
            // Heap allocation for large strings
            let ptr = Box::into_raw(s.to_vec().into_boxed_slice());
            Self {
                heap: ManuallyDrop::new(AllocatedStrVec {
                    ptr: ptr.cast(),
                    len: ptr.len().to_le(),
                }),
            }
        }
    }
    
    fn as_ref(&self) -> &[u8] {
        unsafe {
            if self.inlined[LAST] != 0x00 {
                // Inline: extract length from last byte
                let len = self.inlined[LAST] as usize - 1;
                std::slice::from_raw_parts(self.inlined.as_ptr(), len)
            } else {
                // Heap: read from allocated memory
                std::hint::cold_path();
                let len = usize::from_le(self.heap.len);
                std::slice::from_raw_parts(self.heap.ptr, len)
            }
        }
    }
}
```

**Memory layout:**

```
┌──────────────────────────────────────────────────────────────┐
│                    StrVec MEMORY LAYOUT                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   Short String (< 32 bytes):                                  │
│   ┌────────────────────────────────────────────────┬────┐    │
│   │  V  i  j  a  y  a  w  a  d  a  0  0  ...  0   │ 11 │    │
│   └────────────────────────────────────────────────┴────┘    │
│   └─────────────────────────────────────────────┘  └──┘      │
│                    Data (31 bytes max)        Length marker   │
│                                                              │
│   No heap allocation! 32 bytes total on stack                │
│                                                              │
│   Long String (>= 32 bytes):                                 │
│   ┌────────────────────────────────────────────────────┐    │
│   │  ptr (8 bytes)          │  len (8 bytes)           │    │
│   └────────────────────────────────────────────────────┘    │
│            │                                                 │
│            ▼                                                 │
│   ┌────────────────────────────────────────────────────┐    │
│   │  Heap allocated string bytes...                    │    │
│   └────────────────────────────────────────────────────┘    │
│                                                              │
│   16 bytes on stack + heap allocation                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Chapter 13: The Hash Function

#### Why Default Hashing is Slow

Rust's default `HashMap` uses SipHash - cryptographically secure but slow.

For station names (short, non-malicious), we can use a faster hash.

#### FastHasher Implementation

```
const HASH_K: u64 = 0xf1357aea2e62a9c5;
const HASH_SEED: u64 = 0x13198a2e03707344;

struct Fasthasher(u64);

impl Hasher for Fasthasher {
    fn finish(&self) -> u64 {
        self.0.rotate_left(26)  // Final mixing
    }
    
    fn write(&mut self, bytes: &[u8]) {
        let len = bytes.len();
        let mut acc = HASH_SEED;
        
        match len {
            0..4 => {
                // For very short strings: use first, middle, last bytes
                let low = bytes[0];
                let mid = bytes[len / 2];
                let high = bytes[len - 1];
                acc ^= (low as u64) | ((mid as u64) << 8) | ((high as u64) << 16);
            }
            4.. => {
                // For longer strings: use first 4 bytes
                acc ^= u32::from_le_bytes(bytes[0..4].try_into().unwrap()) as u64;
            }
        }
        
        self.0 = self.0.wrapping_add(acc).wrapping_mul(HASH_K);
    }
}
```

**Key optimizations:**

* No cryptographic security needed
* Use first bytes for quick differentiation
* Multiplication-based mixing (fast on modern CPUs)

---

### Chapter 14: The Temperature Parser (Optimized)

#### Final Optimized Parser

```
#[inline]
fn parse_temperature(k: &[u8]) -> i16 {
    let tlen = k.len();
    
    // Compiler hint: temperature is always >= 3 chars
    unsafe { std::hint::assert_unchecked(tlen >= 3); }
    
    // Branchless sign detection
    let isneg = std::hint::select_unpredictable(k[0] == b'-', true, false);
    let sign = i16::from(!isneg) * 2 - 1;  // -1 or 1
    let skip = usize::from(isneg);         // 0 or 1
    
    // Branchless digit count detection
    let isdd = std::hint::select_unpredictable(k.len() - skip == 4, true, false);
    let mul = i16::from(isdd) * 90 + 10;   // 100 or 10
    
    let t1 = mul * i16::from(k[skip] - b'0');
    let t2 = i16::from(isdd) * 10 * i16::from(k[tlen - 3] - b'0');
    let t3 = i16::from(k[tlen - 1] - b'0');
    
    sign * (t1 + t2 + t3)
}
```

**Optimizations:**

* `select_unpredictable`: Removes branch misprediction
* Direct digit extraction: No loops
* Handles both positive and negative, 1-2 digit temperatures

```
┌──────────────────────────────────────────────────────────────┐
│                TEMPERATURE PARSING CASES                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   Input          →  Parsed Value                             │
│   ─────────────────────────────                              │
│   "12.3"         →  123                                      │
│   "-5.0"         →  -50                                      │
│   "123.4"        →  1234                                     │
│   "-99.9"        →  -999                                     │
│                                                              │
│   All handled without branches, in ~10 CPU cycles            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Chapter Final: The Complete Architecture

#### Final Code Structure

```
┌──────────────────────────────────────────────────────────────┐
│                    brrrc ARCHITECTURE                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   main()                                                     │
│   │                                                          │
│   ├── mmap(file) ──────────────────────────► &[u8]          │
│   │   (Zero-copy file access)                                │
│   │                                                          │
│   ├── Thread::scope ──────────────────────────┐            │
│   │   │                                        │            │
│   │   ├── Thread 1: process_chunk() ──► HashMap           │
│   │   │   ├── find_newline() (SIMD)              │            │
│   │   │   ├── split_semi()                       │            │
│   │   │   ├── parse_temperature()                │            │
│   │   │   └── update_stats()                     │            │
│   │   │                                        │            │
│   │   ├── Thread 2: ...                          │            │
│   │   │                                        │            │
│   │   └── Thread N: ...                          │            │
│   │                                              │            │
│   └── Aggregate: merge all HashMaps ────────────┘            │
│       │                                                      │
│       └── print() ──────────────────────────► Output        │
│           (Sorted by station name)                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

#### Stat Data Structure

```
#[derive(Copy, Clone)]
struct Stat {
    min: i16,    // Minimum temperature (×10)
    max: i16,    // Maximum temperature (×10)
    sum: i64,    // Sum of all temperatures (for average)
    count: u32,  // Number of measurements
}
```

**Output format:**

```
{Station=Min/Avg/Max, ...}
{Abha=-5.0/18.0/39.9, Abidjan=-5.0/26.0/40.0, ...}
```

---

## Epilogue: The Rust Advantage

**Key lessons:**

1. **Profile before optimizing** - Each bottleneck was discovered through profiling
2. **Understand your data** - The one-decimal-place invariant enabled fixed-point arithmetic
3. **Work with the kernel** - mmap + madvise transformed I/O
4. **SIMD is powerful** - 32x throughput on simple operations
5. **Avoid allocations** - StrVec eliminates heap for small strings
6. **Parallelize intelligently** - Chunk at byte boundaries, align at newlines

**What Rust enabled:**

* **Zero-cost abstractions**: Unsafe code for hot paths, safe wrappers elsewhere
* **Portable SIMD**: `std::simd` provides vectorized operations without platform-specific intrinsics
* **Fearless concurrency**: `std::thread::scope` ensures thread safety
* **Control**: Direct memory layout with `#[repr(C)]` and unions

---

## The Code

**GitHub:** https://github.com/ygndotgg/1brrrc

**Build:**

```
cargo build --release
```

**Run:**

```
./target/release/brrrc
```

**Profile:**

```
perf record -F 99 ./target/release/brrrc
perf script > out.perf
flamegraph.pl out.perf > flamegraph.svg
```

---

**brrrc: One Billion Rows in 3.29 Seconds** | my thoughts