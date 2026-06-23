# Advanced DBMS: RocksDB Architecture

This document explores the architecture and internals of **RocksDB**, a high-performance embedded key-value store developed by Meta (Facebook) based on Google's LevelDB. The focus is to analyze Log-Structured Merge-tree (LSM-tree) storage engines, tracing data flow through the read and write paths, explaining compaction strategies, and reviewing key database metrics: Write, Read, and Space Amplification.

---

## 1. Problem Background

Traditional database engines (like InnoDB or PostgreSQL) utilize **B-Trees** to organize on-disk pages. While B-Trees are optimized for reads ($O(\log N)$ page lookups), they suffer from significant limitations under write-heavy workloads:
- **Random Writes**: Updating a B-Tree page requires writing to random blocks on disk. On SSDs and NVMe flash drives, random writes trigger expensive garbage collection blocks and cause write-amplification, reducing the physical lifespan of the drive.
- **In-Place Updates**: Overwriting pages in place requires locking, which limits concurrent write throughput.

To address these limitations, Meta created **RocksDB** in 2012. Built as an embedded C++ library, RocksDB implements a **Log-Structured Merge-tree (LSM-tree)** designed to:
- **Transform Random Writes into Sequential Writes**: All writes are appended sequentially to a log file in memory, maximizing write throughput.
- **Optimize for Flash Storage**: Sequential writes align with flash memory write paths, reducing drive wear and increasing SSD longevity.
- **Scale on Multi-Core Systems**: Utilizing lock-free data structures in memory and background thread pools for disk I/O.

---

## 2. Architecture Overview

RocksDB operates as an in-process key-value store. Its LSM-tree architecture divides storage into a fast in-memory write buffer and multiple tiered, sorted on-disk levels.

```mermaid
graph TD
    subgraph Client["Application Process"]
        App["Application Code"]
    end
    
    subgraph Mem["In-Memory Structures"]
        MemTable["Active MemTable (SkipList)"]
        ImmMemTable["Immutable MemTable(s)"]
    end
    App -->|Write (Put/Delete)| MemTable
    App -->|Read (Get)| MemTable
    
    subgraph Disk["Disk Storage"]
        WAL[("Write-Ahead Log (.log)")]
        
        subgraph L0["Level 0 (Unsorted SSTables)"]
            SST0_1["SSTable 1"]
            SST0_2["SSTable 2"]
        end
        
        subgraph L1["Level 1 (Sorted, Non-Overlapping)"]
            SST1_1["SSTable A"]
            SST1_2["SSTable B"]
        end
        
        subgraph L2["Level 2 (Larger Capacity)"]
            SST2_1["SSTable X"]
            SST2_2["SSTable Y"]
        end
    end
    
    App -->|Write Log| WAL
    MemTable -->|Freeze| ImmMemTable
    ImmMemTable -->|Flush Thread| L0
    
    L0 -->|Compaction Thread| L1
    L1 -->|Compaction Thread| L2
```

### Core Components:
1. **Write-Ahead Log (WAL)**: An on-disk append-only log file. Every write is appended to the WAL before being applied in memory to ensure durability.
2. **MemTable**: An in-memory write buffer (typically implemented as a concurrent SkipList). Active writes are inserted into the MemTable in sorted order.
3. **Immutable MemTable**: When a MemTable reaches its capacity limit, it is frozen and becomes read-only. A background flush thread writes its sorted contents to disk as a Level 0 SSTable.
4. **SSTables (Sorted String Tables)**: Static, read-only on-disk files containing sorted key-value pairs. SSTables are divided into levels ($L_0$ to $L_n$), with each level having a larger size capacity than the preceding one.

---

## 3. Internal Design

### Data Flow: The Write Path

RocksDB write operations (inserts, updates, and deletes) are designed for speed:

```text
Write Operation (Put/Delete)
       │
       ├──► Append to Write-Ahead Log (WAL) on disk (sequential I/O)
       │
       └──► Insert into Active MemTable in memory (SkipList insertion)
               │
               ▼ (Active MemTable Full)
            Freeze active MemTable into Immutable MemTable
               │
               ▼ (Background Flush Thread)
            Flush sorted contents to disk as a Level 0 SSTable file (.sst)
```

- **Deletes**: Deleting a key does not immediately remove it from disk. Instead, the write path inserts a marker called a **Tombstone**. The tombstone moves down the levels during compaction until it reaches the lowest level, where the key is physically purged.

---

### Data Flow: The Read Path

Because keys are stored across multiple SSTable files and levels, reads are more complex:

```text
Read Operation (Get Key)
       │
       ├──► Search Active MemTable (in memory)
       │       [Found] ──► Return Value
       │
       ├──► Search Immutable MemTables (in memory)
       │       [Found] ──► Return Value
       │
       ├──► Search Level 0 SSTables (disk/cache) ──► (Keys can overlap between files)
       │       [Found] ──► Return Value
       │
       └──► Search Level 1 to Ln SSTables (disk/cache) ──► (Keys are non-overlapping within a level)
               [Found] ──► Return Value
```

To avoid reading every SSTable file on disk, RocksDB uses optimization structures:
- **Block Cache**: Caches uncompressed SSTable blocks in memory.
- **Bloom Filters**: Space-efficient probabilistic data structures associated with each SSTable. They determine if a key is **definitely not** in the SSTable, allowing the reader to skip files without performing disk reads.

---

### SSTable Internal Layout

Each Sorted String Table (`.sst`) file is divided into blocks (typically 4 KB to 8 KB):

```text
SSTable File Layout
┌──────────────────────────────────────────────┐
│ Data Block 1 (Sorted Key-Value Pairs)        │
├──────────────────────────────────────────────┤
│ Data Block 2 (Sorted Key-Value Pairs)        │
├──────────────────────────────────────────────┤
│ Filter Block (Bloom Filters for Data Blocks) │
├──────────────────────────────────────────────┤
│ Index Block (Maps Key Ranges to Data Blocks) │
├──────────────────────────────────────────────┤
│ Footer (Fixed size, points to Index/Filters) │
└──────────────────────────────────────────────┘
```

- **Data Blocks**: Store the sorted key-value pairs.
- **Filter Block**: Contains Bloom filters for all data blocks in the file.
- **Index Block**: Contains one entry per data block, storing a key that is greater than or equal to the last key in the block and less than the first key in the next block, mapping key ranges to block offsets.
- **Footer**: Located at the end of the file. It is read first to locate the Index and Filter blocks.

---

### The Compaction Process

Because new SSTables are flushed to disk periodically, duplicate keys and tombstone markers accumulate across different files and levels. RocksDB uses **Compaction** to merge files, clean up old keys, and maintain sorted order.

#### Why Compaction is Required:
- Reclaims disk space by removing superseded key versions and tombstones.
- Limits the number of SSTables a read query must check, improving read performance.
- Maintains the sorted property of levels $L_1$ to $L_n$ (files within these levels have non-overlapping key ranges).

#### Compaction Strategies:

##### 1. Leveled Compaction (Default)
Each level has a maximum capacity (e.g., $L_1 = 10\text{ MB}$, $L_2 = 100\text{ MB}$, $L_3 = 1\text{ GB}$).
- When Level $L$ exceeds its capacity, RocksDB selects one or more SSTables from Level $L$.
- It performs a **multi-way merge-sort** to merge these files with all overlapping SSTables at Level $L+1$.
- **Trade-off**: This strategy minimizes space amplification but increases write amplification because data is read and rewritten multiple times.

##### 2. Universal Compaction
Optimized for write-heavy workloads where Leveled Compaction causes I/O bottlenecks.
- It merges all SSTables into a single large file based on age or size heuristics.
- **Trade-off**: This strategy reduces write amplification but increases read and space amplification.

---

## 4. Design Trade-Offs: The Three Amplifications

LSM-trees are managed by balancing three metrics: **Write Amplification**, **Read Amplification**, and **Space Amplification**.

```text
                       Write Amplification
                       (SSTable Merges)
                             ▲
                            / \
                           /   \
                          /     \
                         ▼       ▼
          Read Amplification ◄───► Space Amplification
        (Binary Search Files)      (Tombstones & Old Keys)
```

1. **Write Amplification (WA)**:
   $$\text{WA} = \frac{\text{Bytes written to storage disk}}{\text{Bytes written to database API}}$$
   - In B-Trees, modifying a 100-byte row requires writing a full 8 KB page, resulting in high write amplification.
   - In RocksDB, Leveled Compaction requires reading, sorting, and rewriting SSTables across levels, which also increases write amplification (often $10\times$ to $30\times$).

2. **Read Amplification (RA)**:
   - The number of disk reads required to satisfy a single read request.
   - In B-Trees, this is low ($O(\log N)$ page reads).
   - In RocksDB, a read might check multiple SSTables across different levels. While Bloom filters reduce this overhead, read amplification remains higher than in B-Trees.

3. **Space Amplification (SA)**:
   $$\text{SA} = \frac{\text{Physical file size on disk}}{\text{Logical size of active data}}$$
   - In RocksDB, deleted keys (tombstones) and older values remain on disk until compaction runs, leading to higher space amplification than in B-Trees.

---

## 5. Suggested Questions & Answers

#### Why are LSM trees preferred in write-heavy workloads?
LSM-trees are optimized for writes because they turn random write operations into sequential disk writes. Instead of updating pages on disk in place, updates are written to an append-only log (WAL) and buffered in memory (MemTable). The memory buffer is flushed to disk sequentially as static SSTables, which maximizes write throughput and minimizes random I/O on flash memory.

#### Why can compaction become expensive?
Compaction requires reading multiple SSTable files from disk, performing a merge-sort in memory, and writing out new sorted SSTables. This process consumes significant CPU and disk I/O bandwidth. Under heavy write workloads, background compaction threads can saturate disk write bandwidth, causing the system to slow down write operations (known as a **write stall**) until compaction catches up.

#### How do Bloom Filters improve read performance?
Bloom filters are space-efficient probabilistic data structures stored in the Filter Block of each SSTable. When a read query searches for a key, RocksDB queries the Bloom filter first.
- If the filter returns **false**, the key is guaranteed not to be in that SSTable, allowing the reader to skip reading the file's index and data blocks from disk.
- If the filter returns **true**, the key might be in the file, and RocksDB proceeds to search the file. This reduces read amplification and disk seeks for non-existent keys.
