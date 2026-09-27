# BACSE202 — Module 3: Physical Database Design and Query Processing

> **Syllabus (9 hours):** File structures: operations on files – files of unordered and ordered records – hashing techniques: static and dynamic – indexing: single-level indexing, multi-level indexing, dynamic multilevel indexing – B+ tree indexing – LSM trees and its variants – Relational Algebra: operators – translating SQL queries into relational algebra – Tuple Relational Calculus – Query Optimization: query trees and heuristic optimization of query trees.

**Sources used for these notes**

| Source file (from the course OneDrive) | Covers |
|---|---|
| `BACSE202-DBS-Module-3 LSM trees.ppt` (284 slides) | Everything in Module 3 |
| `Types_of_Indexes_in_DBMS.pptx` (61 slides) | Index types, B+ tree insertion & deletion walkthroughs |
| `LSM trees 2026.pptx` (26 slides) | LSM trees (identical to slides 161–186 of the main deck) |

`BACSE202-DBS-Module-4.ppt` belongs to Module 4 and is **not** covered here.

Every example from the slides is reproduced and re-worked. Where a slide has an arithmetic or notation mistake, the correct answer is given and the mistake is listed in [§11 Errata](#11-errata--mistakes-found-in-the-slides). Content marked **(Supplementary)** is not on the slides but is part of the syllabus/textbook (Elmasri & Navathe) and is needed to complete a topic.

---

## Table of Contents

1. [Storage and File Structures](#1-storage-and-file-structures)
2. [Hashing](#2-hashing)
3. [Indexing](#3-indexing)
4. [B+ Tree Indexing](#4-b-tree-indexing)
5. [LSM Trees and Variants](#5-lsm-trees-log-structured-merge-trees)
6. [Relational Algebra](#6-relational-algebra)
7. [Translating SQL Queries into Relational Algebra](#7-translating-sql-queries-into-relational-algebra)
8. [Tuple Relational Calculus (Supplementary)](#8-tuple-relational-calculus-supplementary)
9. [Query Processing and Query Optimization](#9-query-processing-and-query-optimization)
10. [Quick Revision Sheet](#10-quick-revision-sheet)
11. [Errata – Mistakes Found in the Slides](#11-errata--mistakes-found-in-the-slides)

---

# 1. Storage and File Structures

## 1.1 Memory hierarchy

```
            ▲ Increase in COST
   ┌────────────────────────────┐
   │     Primary Storage        │  Cache & Main memory (RAM)  – fast, volatile
   ├────────────────────────────┤
   │    Secondary Storage       │  Magnetic disk, SSD         – slower, non-volatile
   ├────────────────────────────┤
   │    Tertiary Storage        │  Tapes & optical disks
   └────────────────────────────┘
            ▼ Increase in CAPACITY & ACCESS TIME
```

- Data is stored in bytes (8 bits).
- **Cache memory** – attached to the CPU.
- **Primary storage** – fast but volatile (RAM).
- **Secondary storage** – slower but non-volatile (SSD, HDD, or sequential devices like tape).
- Data moves secondary → primary → cache → CPU at execution time.
- **Databases are typically stored on secondary storage** (some are main-memory databases).

**Disk hardware:** a disk pack is a stack of platters on a spindle. Each surface has concentric **tracks**, and the same track across all platters forms an (imaginary) **cylinder**. Read/write heads sit on an **arm** moved by an **actuator**.

## 1.2 Buffers and double buffering

- **Buffers** are kept in main memory and hold data brought from secondary storage.
- **Double buffering:** while the I/O processor fills buffer B from disk, the CPU processes buffer A, and then they swap. I/O and processing overlap, so the CPU doesn't wait.

| Time → | t1 | t2 | t3 | t4 | t5 | t6 |
|---|---|---|---|---|---|---|
| Disk block I/O | i: Fill A | i+1: Fill B | i+2: Fill A | i+3: Fill B | i+4: Fill A | – |
| Disk block processing | – | i: Process A | i+1: Process B | i+2: Process A | i+3: Process B | i+4: Process A |

## 1.3 Table → Block → Record

- The database is stored as a collection of **files** (tables).
- Each file is a collection of **pages (blocks)**.
- Each block is a collection of **records**.
- A record is a sequence of **fields**.

Slide example: an Employee table (Name, Age, Gender, Salary) with **100 records stored 10 per block → 10 blocks**. Block 1 holds records 1–10 (Aaron, Ed … Acosta, Marc), Block 2 holds records 11–20 (Allen, Sam … Anderson, Sue), and so on up to Block 10 (Wright, Pam … Zimmer, Byron).

**Data block:** a block is a disk block. Data is transferred from disk to main memory **block by block**, never record by record.

## 1.4 Records, blocking factor, unused space

- **Fixed-length records** – every record occupies the same number of bytes.
  - Slide example: `Name(30) | Ssn(9) | Salary(4) | Job_code(4) | Department(20) | Hire_date(…)`. Fields start at byte 1, 31, 40, 44, 48, 68, and the record is 71 bytes long.
- **Variable-length records** – some fields vary in size (e.g. `Name`, `Department`). They use **separator characters**: one to separate a field name from its value (`=`), one between fields, and one to terminate the record.

With block size $B$ bytes and fixed record size $R$ bytes ($B \ge R$):

$$\text{Blocking factor } bfr = \left\lfloor \frac{B}{R} \right\rfloor \qquad \text{Unused space per block} = B - (bfr \times R)$$

$$\text{Blocks needed for } r \text{ records: } b = \left\lceil \frac{r}{bfr} \right\rceil$$

**Worked example (Supplementary):** $B = 512$, $R = 100$, $r = 30{,}000$:
$bfr = \lfloor 512/100 \rfloor = 5$, unused space $= 512 - 500 = 12$ bytes/block, $b = \lceil 30000/5 \rceil = 6000$ blocks.

## 1.5 I/O cost

- **I/O cost** is the number of blocks that must be read or written to access a record.
- DBMSs are optimised to **minimise I/O cost**, because a disk access is orders of magnitude slower than a memory access.

$$\text{Disk Access Time} = \text{Seek Time} + \text{Rotational Latency} \;(+\;\text{Block transfer time})$$

## 1.6 Operations on files

File I/O is hidden inside SQL. The DBMS **parses** the SQL statement, **converts** it to an internal representation, and **builds an execution plan** out of low-level file operations.

| Level | Operations |
|---|---|
| Basic (record-at-a-time) | `Open`, `Close`, `Reset`, `Read` (Get), `Find` (locate first match), `FindNext` |
| High level | `Insert`, `Modify` (update), `Delete` |
| (Supplementary) set-at-a-time | `FindAll`, `Find n`, `FindOrdered`, `Reorganize` |

## 1.7 Files of unordered records (Heap files)

Records are stored **in order of insertion** (new records go at the end of the file).

| Operation | Behaviour | Cost |
|---|---|---|
| Insert | Very efficient: read the last block, add the record, write it back | ~2 block accesses |
| Search | **Linear search** | Average $b/2$, worst $b$ block accesses, i.e. $O(n)$ |
| Delete | Find the block, remove the record (leaves a hole), **or** set a *deletion bit/marker* | Search + 1 write; needs periodic reorganisation |
| Update | Search + rewrite | Variable-length records may have to move |

Fixed-length records make linear search and record location simpler.

## 1.8 Files of ordered (sequential) records

Records are **physically sorted on an ordering key** field.

| Operation | Behaviour | Cost |
|---|---|---|
| Search on the ordering key | **Binary search** over blocks | $\lceil \log_2 b \rceil$ block accesses, i.e. $O(\log n)$ |
| Insert | **Expensive**: must find the correct position and shift records (often done using an *overflow file* that is merged later) | High |
| Delete / Update of key | **Expensive**, same reason as insert | High |
| Reading in key order | Very efficient (no sorting needed) | $b$ |

**Algorithm – Binary search on an ordering key of a disk file** (slide algorithm 17.1):

```
l ← 1; u ← b;                      (* b = number of file blocks *)
while (u ≥ l) do
begin
    i ← (l + u) div 2;
    read block i of the file into the buffer;
    if K < (ordering key value of the FIRST record in block i)
        then u ← i − 1
    else if K > (ordering key value of the LAST record in block i)
        then l ← i + 1
    else if the record with ordering key value = K is in the buffer
        then goto found
    else goto notfound;
end;
goto notfound;
```

## 1.9 File system vs. DBMS

A **file system** is a collection of raw data files on a disk. A **DBMS** is a set of programs dedicated to storing, maintaining, searching and managing data in databases.

| Advantages of a file system | Disadvantages of a file system |
|---|---|
| Easy to understand | Less security (data is easy to extract) |
| Easy to implement | Data inconsistency |
| Less hardware and software required | Redundancy |
| Fewer skills needed | Sharing information is cumbersome |
| Best for small databases | Slow for huge databases; searching is time-consuming |

## 1.10 File organisation

**File organisation** is the physical arrangement of records on disk. Types listed on the slides:

```
                         File Organization
   ┌────────────┬──────────┬─────────┬──────┬──────────────┬─────────────┐
 Sequential    Heap       Hash      ISAM   B+ Tree FO     Cluster FO
```

Another list on the slides: (1) serial, (2) sequential, (3) indexed sequential, (4) hashed file organisation.

A good file organisation gives:
- fast retrieval of records
- reduced disk access time
- efficient use of disk space

---

# 2. Hashing

## 2.1 Basic concepts

- **Hashing** maps large key values onto a smaller range of array indices using a **hash function**.
- **Hash table** – an array-based structure where each location (**bucket**) is reached through the hash function.
- **Bucket** – a location (index) in the hash table that stores one or more records. It is also called a *hash index*. On disk, a bucket is usually one disk block (or a cluster of blocks).

```
Key ──► Hash Function ──► Hash value ──► bucket 0 | 1 | 2 | … | n
```

Slide illustration: primary keys 100, 122, 106, 144, 223 with $H(x) = x \bmod 5$. $H(106) = 1$, so record (106, Ethan, Indore) goes to bucket 1. Likewise 100 → 0, 122 → 2, 144 → 4 and 223 → 3.

### Hash function

A hash function maps a key from a set $K$ into an index of a table of size $n$:

$$h : K \rightarrow \{0, 1, \dots, n-1\}$$

- A key can be a number, a string, a record, etc.
- $|K|$ is usually much larger than $n$, so **different keys can hash to the same location**.
- This is a **collision**, and the colliding keys are called **synonyms**.

**A good hash function should:**
1. minimise collisions
2. be easy and quick to compute
3. distribute keys evenly over the table
4. use all the information in the key

### Static vs. dynamic hashing

| Static hashing | Dynamic hashing |
|---|---|
| The hash function always maps keys to a **fixed set of buckets**; the table size never changes | The table can **grow (and shrink)**; the hash function changes as the table grows |
| e.g. division method + chaining/probing | e.g. extendible hashing, linear hashing |

## 2.2 Division method (modulo division)

$$h(key) = key \bmod m$$

Choose $m$ (the table size) larger than the number of keys. $m$ is usually **prime**, which spreads keys better.

**Example (slide):** table size 10, keys 20, 21, 24, 26, 32, 34

| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|---|---|---|---|---|---|---|---|---|---|---|
| Keys | 20 | 21 | 32 | | **24, 34 ← collision** | | 26 | | | |

$24 \bmod 10 = 34 \bmod 10 = 4$. This is a **hash collision** (hash clash): two distinct inputs give the same output.

## 2.3 Collision resolution techniques

```
Collision Resolution
├── Closed Addressing  (= Open Hashing)
│     └── Separate (overflow) chaining
├── Open Addressing    (= Closed Hashing)
│     ├── Linear probing      (u + i)   mod m
│     ├── Quadratic probing   (u + i²)  mod m
│     └── Double hashing      (u + v·i) mod m
└── Dynamic Hashing
      ├── Rehashing
      ├── Extendible hashing
      └── Linear hashing
```

> ⚠️ The names are confusing. **Open hashing = closed addressing**: the key stays at its home address and goes into a chain there. **Closed hashing = open addressing**: the key may be placed at another open address in the table itself.

---

## 2.4 Closed addressing: separate chaining

- Each table slot holds a pointer to a **linked list** (chain) of all keys that hash there.
- The array elements are pointers to the first node of each list.
- In the textbook version a new item is inserted at the **front** of its list.
- The table never "fills up"; chains just get longer.

### Example 1 (slide): $h(k) = k \bmod 7$, keys 50, 700, 76, 85, 92, 73, 101

| Key | $h(k)$ | Result | Probes |
|---|---|---|---|
| 50 | 50 mod 7 = 1 | slot 1 | 1 |
| 700 | 700 mod 7 = 0 | slot 0 | 1 |
| 76 | 76 mod 7 = 6 | slot 6 | 1 |
| 85 | 85 mod 7 = 1 | collision → chain at slot 1 | 2 |
| 92 | 92 mod 7 = 1 | collision → chain at slot 1 | 3 |
| 73 | 73 mod 7 = 3 | slot 3 | 1 |
| 101 | 101 mod 7 = 3 | collision → chain at slot 3 | 2 |

```
0 → 700
1 → 50 → 85 → 92
2 →
3 → 73 → 101
4 →
5 →
6 → 76
```

### Example 2 (slide): $h(k) = k \bmod 10$, keys 0, 1, 4, 9, 16, 25, 36, 49, 64, 81 (insert at front)

```
0 → 0
1 → 81 → 1
2 →
3 →
4 → 64 → 4
5 → 25
6 → 36 → 16
7 →
8 →
9 → 49 → 9
```

### Example 3 (slide): $h(k) = (2k+3) \bmod 10$, $m = 10$, keys 3, 2, 9, 6, 11, 13, 7, 12

| Key $k$ | $(2k+3) \bmod 10$ | Index |
|---|---|---|
| 3 | (6+3) mod 10 = 9 | 9 |
| 2 | (4+3) mod 10 = 7 | 7 |
| 9 | (18+3) mod 10 = 1 | 1 |
| 6 | (12+3) mod 10 = 5 | 5 |
| 11 | (22+3) mod 10 = 5 | 5 (collision with 6) |
| 13 | (26+3) mod 10 = 9 | 9 (collision with 3) |
| 7 | (14+3) mod 10 = 7 | 7 (collision with 2) |
| 12 | (24+3) mod 10 = 7 | 7 (collision with 2, 7) |

```
0 → –
1 → 9
2 → –
3 → –
4 → –
5 → 6 → 11
6 → –
7 → 2 → 7 → 12
8 → –
9 → 3 → 13
```

---

## 2.5 Open addressing (closed hashing)

When a collision happens, alternative cells are probed **inside the table** until an empty one is found. With $u = h(k)$ and $i = 0, 1, 2, \dots, m-1$:

| Technique | Probe sequence |
|---|---|
| Linear probing | $(u + i) \bmod m$ |
| Quadratic probing | $(u + i^2) \bmod m$ |
| Double hashing | $(u + v \cdot i) \bmod m$, where $v = h_2(k)$ |

### 2.5.1 Linear probing

- Insert $k$ at the first free location in $(h(k) + i) \bmod m$ for $i = 0, 1, \dots, m-1$.
- On a collision, check the **next slot sequentially**, wrapping to the start of the table at the end.
- Causes **primary clustering**: long runs of filled slots build up.
- Insert and search get slow once the table is about half full.

#### Example 1 (slide): $h(k) = k \bmod 7$, keys 50, 700, 76, 85, 92, 73, 101

| Key | $u = h(k)$ | Probe sequence | Final slot | Probes |
|---|---|---|---|---|
| 50 | 1 | 1 | 1 | 1 |
| 700 | 0 | 0 | 0 | 1 |
| 76 | 6 | 6 | 6 | 1 |
| 85 | 1 | 1 ✗, 2 ✓ | 2 | 2 |
| 92 | 1 | 1 ✗, 2 ✗, 3 ✓ | 3 | 3 |
| 73 | 3 | 3 ✗, 4 ✓ | 4 | 2 |
| 101 | 3 | 3 ✗, 4 ✗, 5 ✓ | 5 | 3 |

Final table: `0:700 | 1:50 | 2:85 | 3:92 | 4:73 | 5:101 | 6:76`

#### Example 2 (slide): $h(k) = (2k+3) \bmod 10$, $m = 10$, keys 3, 2, 9, 6, 11, 13, 7, 12

| Key | $h(k)$ | Probe sequence | Final index | Probes |
|---|---|---|---|---|
| 3 | 9 | 9 | 9 | 1 |
| 2 | 7 | 7 | 7 | 1 |
| 9 | 1 | 1 | 1 | 1 |
| 6 | 5 | 5 | 5 | 1 |
| 11 | 5 | (5+0)=5 ✗ → (5+1)=6 ✓ | 6 | 2 |
| 13 | 9 | (9+0)=9 ✗ → (9+1) mod 10=0 ✓ | 0 | 2 |
| 7 | 7 | 7 ✗ → 8 ✓ | 8 | 2 |
| 12 | 7 | 7 ✗, 8 ✗, 9 ✗, 0 ✗, 1 ✗, **2 ✓** | 2 | 6 |

Final table:

| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|---|---|---|---|---|---|---|---|---|---|---|
| Key | 13 | 9 | 12 | – | – | 6 | 11 | 2 | 7 | 3 |

**Hash index: 13, 9, 12, __, __, 6, 11, 2, 7, 3**

| Method | Pros | Cons |
|---|---|---|
| Linear probing | Simple; no extra memory for pointers | **Primary clustering** (long blocks of filled slots, e.g. slots 5, 6, 7, 8 form one cluster); performance drops as the table fills |

### 2.5.2 Quadratic probing

- Reduces the clustering of linear probing. The distance between probes grows quadratically ($1^2, 2^2, 3^2, \dots$).
- In the $i$-th iteration, look at slot $(h(k) + i^2) \bmod m$, always measured from the **original** hash location.

$$h'(k, i) = \left(h(k) + i^2\right) \bmod m, \quad i = 0, 1, 2, \dots$$

(The slides also call this the "mid-square method"; see Errata.)

#### Example 1 (slide): $m = 7$, $h(x) = x \bmod 7$, $f(i) = i^2$, insert 22, 30, 50

1. Create an empty table of size 7.
2. $h(22) = 1$ → slot 1 is empty → insert 22. $h(30) = 2$ → slot 2 is empty → insert 30.
3. $h(50) = 1$ → occupied. Try $1 + 1^2 = 2$ → occupied. Try $1 + 2^2 = 5$ → empty → **insert 50 at slot 5**.

Final: `0:– | 1:22 | 2:30 | 3:– | 4:– | 5:50 | 6:–`

#### Example 2 (slide, corrected): $h(k) = k \bmod 7$, keys 50, 700, 76, 85, 92, 73, 101

| Key | $u$ | Probe sequence $(u + i^2) \bmod 7$ | Slot | Probes |
|---|---|---|---|---|
| 50 | 1 | 1 | 1 | 1 |
| 700 | 0 | 0 | 0 | 1 |
| 76 | 6 | 6 | 6 | 1 |
| 85 | 1 | 1+0=1 ✗, 1+1=2 ✓ | 2 | 2 |
| 92 | 1 | 1 ✗, 2 ✗, 1+4=5 ✓ | 5 | 3 |
| 73 | **3** (73 = 10×7 + 3) | 3 ✓ | 3 | 1 |
| 101 | 3 | 3 ✗, 3+1=4 ✓ | 4 | 2 |

Final: `0:700 | 1:50 | 2:85 | 3:73 | 4:101 | 5:92 | 6:76`

> The slide wrote $73 \bmod 7 = 4$ and left 101 as "??". The correct values are shown above.

#### Example 3 (slide): $h(k) = (2k+3) \bmod 10$, $m = 10$, keys 3, 2, 9, 6, 11, 13, 7, 12

| Key | $h(k)$ | Probe sequence | Index | Probes |
|---|---|---|---|---|
| 3 | 9 | 9 | 9 | 1 |
| 2 | 7 | 7 | 7 | 1 |
| 9 | 1 | 1 | 1 | 1 |
| 6 | 5 | 5 | 5 | 1 |
| 11 | 5 | (5+0²)=5 ✗, (5+1²)=6 ✓ | 6 | 2 |
| 13 | 9 | (9+0²)=9 ✗, (9+1²) mod 10=0 ✓ | 0 | 2 |
| 7 | 7 | (7+0²)=7 ✗, (7+1²)=8 ✓ | 8 | 2 |
| 12 | 7 | 7 ✗, (7+1)=8 ✗, (7+4) mod 10=1 ✗, (7+9) mod 10=6 ✗, (7+16) mod 10=**3 ✓** | 3 | 5 |

| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|---|---|---|---|---|---|---|---|---|---|---|
| Key | 13 | 9 | – | 12 | – | 6 | 11 | 2 | 7 | 3 |

**Hash index: 13, 9, ___, 12, ___, 6, 11, 2, 7, 3**

#### Example 4 (slide – additional problem): $h(k) = k \bmod 10$, $h'(k,i) = (h(k) + i^2) \bmod 10$, keys 42, 16, 91, 33, 18, 27, 36, 62

| Key | $h(k)$ | Probes | Index |
|---|---|---|---|
| 42 | 2 | 2 ✓ | 2 |
| 16 | 6 | 6 ✓ | 6 |
| 91 | 1 | 1 ✓ | 1 |
| 33 | 3 | 3 ✓ | 3 |
| 18 | 8 | 8 ✓ | 8 |
| 27 | 7 | 7 ✓ | 7 |
| 36 | 6 | 6 ✗ (16); $i=1$: 7 ✗ (27); $i=2$: (6+4)=10 mod 10 = **0 ✓** | 0 |
| 62 | 2 | 2 ✗; $i$=1→3 ✗; 2→6 ✗; 3→11 mod 10=1 ✗; 4→18 mod 10=8 ✗; 5→27 mod 10=7 ✗; 6→38 mod 10=8 ✗; 7→51 mod 10=1 ✗; 8→66 mod 10=6 ✗; 9→83 mod 10=3 ✗ | **cannot be inserted** |

Final table: `0:36 | 1:91 | 2:42 | 3:33 | 4:empty | 5:empty | 6:16 | 7:27 | 8:18 | 9:empty`

> **Key lesson:** quadratic probing can **fail to find an empty slot even when the table has free cells** (slots 4, 5 and 9 are free here). $i^2 \bmod 10$ only takes the values {0, 1, 4, 5, 6, 9}, so from home slot 2 only slots {2, 3, 6, 7, 8, 1} are reachable. With a **prime** table size and the table less than half full, quadratic probing is guaranteed to find a slot.

| Method | Pros | Cons |
|---|---|---|
| Quadratic probing | Reduces **primary clustering**; spreads keys better than linear probing | Suffers from **secondary clustering**: keys with the same home slot follow the same probe sequence (e.g. every key hashing to 7 tries 7 → 8 → 1 → 6 → 3 …). May not find an empty slot unless the table size is prime |

### 2.5.3 Double hashing

- Uses **two hash functions**: $h_1$ gives the initial position and $h_2$ gives the **step size**.

$$h(k, i) = \left(h_1(k) + i \cdot h_2(k)\right) \bmod m$$

- Keys that share a home slot usually have different step sizes, so there is **no primary or secondary clustering**. It is one of the best probing methods and gives a nearly uniform distribution.
- $h_2(k)$ must **never be 0** and should be relatively prime to $m$ (e.g. $m$ prime). The slide examples show what goes wrong otherwise.

#### Example 1 (slide): $m = 7$, $h_1(k) = k \bmod 7$, $h_2(k) = 1 + (k \bmod 5)$, keys 27, 43, 92, 72

| Step | Key | Computation | Slot |
|---|---|---|---|
| 1 | 27 | 27 mod 7 = 6 (empty) | 6 |
| 2 | 43 | 43 mod 7 = 1 (empty) | 1 |
| 3 | 92 | 92 mod 7 = 1 → **collision**. $h_2(92) = 1 + (92 \bmod 5) = 1 + 2 = 3$. $h = (1 + 1 \times 3) \bmod 7 = 4$ (empty) | 4 |
| 4 | 72 | 72 mod 7 = 2 (empty) | 2 |

Final: `0:– | 1:43 | 2:72 | 3:– | 4:92 | 5:– | 6:27`

#### Example 2 (slide): keys 3, 2, 1, 6, 11, 13, 7, 12; $h_1(k) = (2k+3) \bmod 10$, $h_2(k) = (3k+1) \bmod 10$, $m = 10$, probe $(u + v \cdot i) \bmod m$

| Key | $u = h_1$ | $v = h_2$ | Probes | Slot |
|---|---|---|---|---|
| 3 | 9 | – | 9 | 9 |
| 2 | 7 | – | 7 | 7 |
| 1 | 5 | – | 5 | 5 |
| 6 | 5 ✗ | (18+1) mod 10 = 9 | 5+9·0=5 ✗; (5+9·1) mod 10=4 ✓ | 4 |
| 11 | 5 ✗ | (33+1) mod 10 = 4 | 5 ✗; 5+4=9 ✗; (5+8) mod 10=3 ✓ | 3 |
| 13 | 9 ✗ | (39+1) mod 10 = **0** | 9, 9, 9, … forever | **cannot be placed** ($h_2 = 0$) |
| 7 | 7 ✗ | (21+1) mod 10 = 2 | 7 ✗; 9 ✗; (7+4) mod 10=1 ✓ | 1 |
| 12 | 7 ✗ | (36+1) mod 10 = 7 | 7 ✗; (7+7) mod 10=4 ✗; (7+14) mod 10=1 ✗; (7+21) mod 10=8 ✓ | 8 |

Final: `0:– | 1:7 | 2:– | 3:11 | 4:6 | 5:1 | 6:– | 7:2 | 8:12 | 9:3` (13 not inserted)

#### Example 3 (slide): keys {3, 2, 9, 6, 11, 13, 7, 12}, $h_1(k) = (2k+3) \bmod 10$, $h_2(k) = (3k+1) \bmod 10$, $m = 10$

| Key | $h_1$ | $h_2$ | Probe sequence $(h_1 + i \cdot h_2) \bmod 10$ | Slot | Probes |
|---|---|---|---|---|---|
| 3 | 9 | – | 9 | 9 | 1 |
| 2 | 7 | – | 7 | 7 | 1 |
| 9 | 1 | – | 1 | 1 | 1 |
| 6 | 5 | – | 5 | 5 | 1 |
| 11 | 5 ✗ | 4 | 5 ✗ → 9 ✗ → (5+8) mod 10 = 3 ✓ | 3 | 3 |
| 13 | 9 ✗ | 0 | 9 → 9 → 9 … | **cannot map 13** | ∞ |
| 7 | 7 ✗ | 2 | 9 ✗, 1 ✗, 3 ✗, 5 ✗, 7 ✗, 9 ✗, 1 ✗, 3 ✗, 5 ✗ – only odd slots are ever visited, and all are full | **cannot map 7** | ∞ |
| 12 | 7 ✗ | 7 | (7+0) = 7 ✗ → (7+7) mod 10 = 4 ✓ | 4 | 2 |

| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|---|---|---|---|---|---|---|---|---|---|---|
| Key | – | 9 | – | 11 | 12 | 6 | – | 2 | – | 3 |

> Why 7 fails: $\gcd(h_2(7), m) = \gcd(2, 10) = 2$, so the probe sequence visits only $m/2 = 5$ slots. This is why $h_2(k)$ should be relatively prime to $m$.

#### Example 4 (slide – additional problem): $m = 11$, $h_1(k) = k \bmod 11$, $h_2(k) = 8 - (k \bmod 8)$, keys 20, 34, 45, 70, 56

| Key | $h_1$ | $h_2$ | Probe sequence $(h_1 + i \cdot h_2) \bmod 11$ | Index |
|---|---|---|---|---|
| 20 | 9 | – | 9 ✓ | **9** |
| 34 | 1 | – | 1 ✓ | **1** |
| 45 | 1 ✗ | 8 − 5 = 3 | (1+3) = 4 ✓ | **4** |
| 70 | 4 ✗ | 8 − 6 = 2 | (4+2) = 6 ✓ | **6** |
| 56 | 1 ✗ | 8 − 0 = 8 | (1+8)=9 ✗ → (1+16) mod 11=6 ✗ → (1+24) mod 11=3 ✓ | **3** |

| Method | Pros | Cons |
|---|---|---|
| Double hashing | Avoids both primary and secondary clustering; distributes keys uniformly; uses the table efficiently | Needs **two good hash functions**; slightly more complex to implement |

### 2.5.4 Summary of open-addressing methods

| | Linear | Quadratic | Double |
|---|---|---|---|
| Probe | $(u+i) \bmod m$ | $(u+i^2) \bmod m$ | $(u+i\,v) \bmod m$ |
| Primary clustering | Yes | No | No |
| Secondary clustering | Yes | Yes | No |
| Always finds a free slot? | Yes (if any exists) | Not guaranteed | Only if $\gcd(v, m) = 1$ |

---

## 2.6 Deficiencies of static hashing

In static hashing, $h$ maps keys to a **fixed set of $B$ bucket addresses**, but databases grow and shrink over time.

- If there are **too few buckets** and the file grows → too many overflows → performance degrades.
- If space is allocated for anticipated growth → a lot of **space is wasted** initially (buckets are underfull).
- If the database shrinks → space is wasted again.
- One solution is **periodic reorganisation** with a new hash function, but this is expensive and disrupts normal operation.
- **Better solution:** let the number of buckets change dynamically → **dynamic hashing**.

## 2.7 Dynamic hashing techniques

1. **Periodic rehashing.** When the number of entries reaches (say) 1.5 × the table size, create a new table of (say) 2 × the size and **rehash all entries** into it.
2. **Linear hashing.** Do the rehashing **incrementally**: buckets are split one at a time in a fixed round-robin order.
3. **Extendible hashing.** Designed for disk-based hashing. Several hash values can share one bucket, and the **directory doubles without doubling the number of buckets**.

---

## 2.8 Extendible hashing

A dynamic hashing technique that uses a **directory of pointers to buckets**. It handles overflow by **bucket splitting** and **directory expansion**.

```
   Global depth = 2
   Directory             Buckets (local depth)
   ┌────┐
   │ 00 │ ───────────► [ Data ]  LD = 2
   │ 01 │ ───────────► [ Data ]  LD = 2
   │ 10 │ ───────────► [ Data ]  LD = 2
   │ 11 │ ───────────► [ Data ]  LD = 2
   └────┘
```

### Key terminology

| Term | Meaning |
|---|---|
| **Directory** | Array of pointers to buckets. Number of entries $= 2^{GD}$ |
| **Bucket** | Stores the actual keys/records. **Several directory entries may point to the same bucket** |
| **Global depth (GD)** | Number of hash bits used to index the directory (kept in the file header) |
| **Local depth (LD)** | Number of bits actually used to distinguish the keys of a particular bucket |
| Rule | $LD \le GD$ always. A bucket is pointed to by $2^{GD - LD}$ directory entries |

The hash value is treated as a binary number, and its **last GD bits (LSBs)** give the directory index. (Some examples use the **first GD bits (MSBs)** instead; both work if used consistently.)

### Basic working (insert)

1. Take the data item, e.g. **49**.
2. Convert it to binary: **49 → 110001**.
3. Check the global depth, e.g. **GD = 3**.
4. Take GD LSBs: **001**.
5. Go to directory entry **001** and follow its pointer to the bucket.
6. Insert into the bucket.
7. If the bucket **overflows**, handle it as below and rehash the split elements.
8. Insertion is complete.

### Overflow handling ("Step 7")

| Case | Condition | Action |
|---|---|---|
| **Case 1** | $LD = GD$ | **Double the directory** ($GD \leftarrow GD + 1$), **split** the bucket, $LD \leftarrow LD + 1$, rehash the bucket's keys |
| **Case 2** | $LD < GD$ | **Split the bucket only** (no directory doubling), $LD \leftarrow LD + 1$, re-point half of its directory entries to the new bucket, rehash its keys |

> **Rule to remember:** *If a bucket whose local depth equals the global depth is split, the directory must be doubled.*

**Search:** apply $h$, take the last GD bits, follow the directory pointer, then search that one bucket. That is **1 directory lookup + 1 bucket access**.

---

### Example 1 (slide): insert 16, 4, 6, 22, 24, 10, 31, 7, 9, 20, 26 — bucket size 3, hash = GD LSBs

Binary forms:

| Key | 16 | 4 | 6 | 22 | 24 | 10 | 31 | 7 | 9 | 20 | 26 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Binary | 10000 | 00100 | 00110 | 10110 | 11000 | 01010 | 11111 | 00111 | 01001 | 10100 | 11010 |

**Initial state:** GD = 1. Directory 0 → empty bucket (LD 1), directory 1 → empty bucket (LD 1).

**Insert 16** (1000**0**) → LSB 0 → directory 0.

```
GD=1   0 → [16]        LD=1
       1 → [ ]         LD=1
```

**Insert 4** (10**0**) **and 6** (11**0**) → LSB 0 → directory 0.

```
GD=1   0 → [16, 4, 6]  LD=1   (full)
       1 → [ ]         LD=1
```

**Insert 22** (1011**0**) → directory 0 → bucket full → **overflow**. $LD = GD = 1$ → **Case 1**: split the bucket and double the directory. GD = 2, and 16, 4, 6, 22 are rehashed on 2 LSBs: 16 → **00**, 4 → **00**, 6 → **10**, 22 → **10**.

```
GD=2   00 → [16, 4]    LD=2
       10 → [6, 22]    LD=2
       01 ┐
       11 ┴→ [ ]       LD=1   (shared bucket)
```

**Insert 24** (110**00**) → 00 and **10** (10**10**) → 10. No overflow.

```
GD=2   00 → [16, 4, 24]   LD=2
       10 → [6, 22, 10]   LD=2
       01, 11 → [ ]       LD=1
```

**Insert 31** (111**11**), **7** (1**11**), **9** (10**01**) → endings 11, 11, 01 → all go to the shared bucket. No overflow.

```
GD=2   00 → [16, 4, 24]   LD=2
       10 → [6, 22, 10]   LD=2
       01, 11 → [31, 7, 9] LD=1
```

**Insert 20** (101**00**) → 00 → full → overflow. $LD = GD = 2$ → **Case 1**: double the directory (GD = 3) and split. Rehash 16, 4, 24, 20 on 3 LSBs: 16 → **000**, 24 → **000**, 4 → **100**, 20 → **100**.

```
GD=3   000 → [16, 24]            LD=3  ┐ split buckets
       100 → [4, 20]             LD=3  ┘
       010, 110 → [6, 22, 10]    LD=2
       001, 011, 101, 111 → [31, 7, 9]  LD=1
```

**Insert 26** (11**010**) → 010 → bucket [6, 22, 10] is full → overflow. $LD = 2 < GD = 3$ → **Case 2**: split only, no doubling. Rehash on 3 bits: 10 = 01**010** → 010, 26 → 010, 6 = 00**110** → 110, 22 = 10**110** → 110.

**Final state:**

```
GD=3   000 → [16, 24]            LD=3
       100 → [4, 20]             LD=3
       010 → [10, 26]            LD=3   ┐ affected buckets
       110 → [6, 22]             LD=3   ┘
       001, 011, 101, 111 → [31, 7, 9]  LD=1
```

---

### Example 2 (slide – textbook style): bucket capacity 4, GD = 2

**Initial state** (keys shown as `k*` = data entry for key k):

```
GD=2   00 → A [4*, 12*, 32*, 16*]   LD=2
       01 → B [1*, 5*, 21*]         LD=2
       10 → C [10*]                 LD=2
       11 → D [15*, 7*, 19*]        LD=2
```

- **Search 5\*:** 5 = 1**01** → last 2 bits 01 → bucket B → found.
- **Insert 13\*:** 13 = 11**01** → 01 → B has room → B = [1*, 5*, 21*, 13*].
- **Insert 20\*:** 20 = 101**00** → 00 → A is **full** → split and redistribute. A: 32 = 100**000**, 16 = 10**000** → keep A (000). A2 ("split image" of A): 4 = **100**, 12 = 1**100**, 20 = 10**100** → A2 (100).
  - *Is this enough?* No. With GD = 2 the directory cannot tell A from A2, and $LD(A) = GD$, so we **double the directory** and set GD = 3. "The first two bits say which pair of buckets; the third bit distinguishes between them."

```
GD=3   000 → A  [32*, 16*]          LD=3
       100 → A2 [4*, 12*, 20*]      LD=3
       001, 101 → B [1*, 5*, 21*, 13*]  LD=2
       010, 110 → C [10*]           LD=2
       011, 111 → D [15*, 7*, 19*]  LD=2
```

- **Insert 9\*:** 9 = 1**001** → 001 → B is **full**. $LD(B) = 2 < GD = 3$ → split B **without doubling the directory**. Rehash on 3 bits: 1 = **001**, 9 = 1**001** → B (001); 5 = **101**, 21 = 10**101**, 13 = 1**101** → B2 (101).

```
GD=3   000 → A  [32*, 16*]          LD=3
       100 → A2 [4*, 12*, 20*]      LD=3
       001 → B  [1*, 9*]            LD=3
       101 → B2 [5*, 21*, 13*]      LD=3
       010, 110 → C [10*]           LD=2
       011, 111 → D [15*, 7*, 19*]  LD=2
```

*When NOT to double the directory?* When the overflowing bucket has $LD < GD$.

---

### Problem 1 (slide, with deletion): bucket capacity 3, initial GD = 2; insert 8, 4, 12, 16, 20; delete 12, 20

Binary: 8 = 01000, 4 = 00100, 12 = 01100, 16 = 10000, 20 = 10100. All end in **00**.

**Insert**
1. 8, 4, 12 → bucket 00 = {8, 4, 12} (full).
2. 16 → 00 → overflow. $LD = GD = 2$ → double the directory → GD = 3. Redistribute on 3 bits: 8 = **000**, 16 = **000**, 4 = **100**, 12 = **100**.
   - 000 = {8, 16}, 100 = {4, 12}
3. 20 = 10**100** → 100 → {4, 12, 20}. No overflow (capacity 3).

**Delete**
1. Delete 12 → 100 = {4, 20}.
2. Delete 20 → 100 = {4}.
3. Check merge with the **buddy** bucket (flip the top local bit: 100 ↔ 000). 000 has 2 keys, total $1 + 2 = 3 \le 3$ → **merge**, and LD drops from 3 to 2.
4. No bucket now has LD = 3 → **directory shrinks**, GD drops from 3 to 2.

**Final:** GD = 2, directory 00 → {4, 8, 16}.

### Problem 2 (slide, with deletion): bucket capacity 2, initial GD = 2; insert 1, 5, 9, 13, 17; delete 9, 17, 13

Binary: 1 = 00001, 5 = 00101, 9 = 01001, 13 = 01101, 17 = 10001. All end in **01**.

**Insert**
1. 1, 5 → bucket 01 = {1, 5} (full).
2. 9 → overflow. $LD = GD = 2$ → double the directory, GD = 3, LD 2 → 3. Redistribute on 3 bits: 1 = **001**, 9 = 1**001** → 001; 5 = **101** → 101.
   - 001 = {1, 9}, 101 = {5}
3. 13 = 1**101** → 101 → {5, 13}.
4. 17 = 10**001** → 001 → {1, 9, 17} → overflow. $LD = GD = 3$ → double the directory, GD = 4. Redistribute on 4 bits: 1 = **0001**, 17 = **0001** → 0001; 9 = **1001** → 1001.
   - 0001 = {1, 17} (LD 4), 1001 = {9} (LD 4), and {5, 13} has LD 3 (pointed to by 0101 and 1101). Other entries are empty.

**Delete 9**
- Bucket 1001 becomes empty. Its buddy (flip the top local bit) is 0001 = {1, 17}. Total $0 + 2 = 2 \le 2$ → **merge** → {1, 17}, LD 4 → 3. Entries 0001 and 1001 now point to the same bucket.
- Does any bucket have LD = 4? No → **shrink the directory**: GD 4 → 3, directory size 8.

**Delete 17**
- {1, 17} → {1}. Its buddy is 101 = {5, 13}. Total $1 + 2 = 3 > 2$ → **cannot merge**.
- State: GD = 3, 001 → {1}, 101 → {5, 13}.

**Delete 13**
- {5, 13} → {5}. Buddy {1}: total $1 + 1 = 2 \le 2$ → **merge** → {1, 5}, LD 3 → 2.
- All buckets now have LD ≤ 2 → shrink the directory: GD 3 → 2, directory size 4.

**Final answer:** GD = 2, directory 01 → {1, 5}; the other entries are empty.

---

### Example 3 (slide – MSB version): insert 17, 5, 6, 22, 24, 11, 30, 7, 10, 21, 27 — bucket size 3, hash = first GD bits (MSBs)

| Key | 17 | 5 | 6 | 22 | 24 | 11 | 30 | 7 | 10 | 21 | 27 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 5-bit binary | 10001 | 00101 | 00110 | 10110 | 11000 | 01011 | 11110 | 00111 | 01010 | 10101 | 11011 |

Start with GD = 1, two buckets (LD 1).

1. **17** (1…) → dir 1; **5** (0…) → dir 0; **6** (0…) → dir 0; **22** (1…) → dir 1.
2. **Insert 24** (**1**1000) → MSB 1 → dir 1.
   ```
   GD=1   0 → [5, 6]
          1 → [17, 22, 24]   (full)
   ```
3. **Insert 11** (**0**1011) → MSB 0 → dir 0 → [5, 6, 11].
4. **Insert 30** (**1**1110) → dir 1 → full → overflow. $LD = GD = 1$ → split and double, GD = 2. Rehash on 2 MSBs: 17 (**10**001) → 10, 22 (**10**110) → 10, 24 (**11**000) → 11, 30 (**11**110) → 11.
   ```
   GD=2   00, 01 → [5, 6, 11]    LD=1
          10     → [17, 22]      LD=2
          11     → [24, 30]      LD=2
   ```

The slides stop here. **Completing the example:**

5. **Insert 7** (**00**111) → 00 → [5, 6, 11] full. $LD = 1 < GD = 2$ → **split only**. Rehash on 2 MSBs: 5 (00), 6 (00), 7 (00) → 00; 11 (**01**011) → 01.
   - 00 → [5, 6, 7] (LD 2), 01 → [11] (LD 2)
6. **Insert 10** (**01**010) → 01 → [11, 10].
7. **Insert 21** (**10**101) → 10 → [17, 22, 21].
8. **Insert 27** (**11**011) → 11 → [24, 30, 27].

**Final (MSB example):**

```
GD=2   00 → [5, 6, 7]      LD=2
       01 → [11, 10]       LD=2
       10 → [17, 22, 21]   LD=2
       11 → [24, 30, 27]   LD=2
```

---

### Advantages and limitations of extendible hashing

| Advantages | Limitations |
|---|---|
| Data retrieval is cheap (1 directory lookup + 1 bucket) | The directory can grow very large if records are **skewed** (many keys share the same bits) |
| No data loss: storage grows dynamically | Every bucket has a **fixed size** |
| Only the overflowing bucket's keys are rehashed when the hash function changes | Memory is wasted on pointers when GD − LD becomes large |
| No overflow chains in normal operation | Complicated to code |

### Summary of hashing

- Hash-based indexes are **best for equality searches** and **cannot support range searches**.
- Static hashing can build **long overflow chains**.
- Extendible hashing avoids overflow pages by **splitting a full bucket** when a new entry is added. Duplicates may still need overflow pages.
- The directory tracks the buckets and doubles periodically. It can get large with skewed data, which costs an extra I/O if it doesn't fit in memory.
- **(Supplementary) Linear hashing** needs no directory. Buckets are split in round-robin order (bucket `next`, then `next+1`, …) whenever any overflow occurs, using the pair $h_i(k) = k \bmod 2^i M$ and $h_{i+1}(k) = k \bmod 2^{i+1} M$.

---

# 3. Indexing

## 3.1 What is an index?

- An **index** is a data structure that lets the DBMS find records quickly **without scanning the whole file**, like the index of a textbook.
- In its simplest form an index is a small table with **two columns**:
  1. a copy of the **search key** (e.g. the primary or candidate key)
  2. a **pointer** to the disk block (or record) that holds that key
- An index takes a search key as input and efficiently returns the matching records.
- Index entries have the form `<field value, pointer>`, **ordered by field value**. An index is an **access path** to the data, and index structures provide **secondary access paths**.
- **Goal:** minimise disk block accesses during query processing.
- Index files are typically **much smaller** than the data file.

**Search key:** an attribute or set of attributes used to look up records in a file.

### Book analogy (from the slides)

| Without an index (ordered file) | With an index |
|---|---|
| Open the middle page and move left/right, i.e. binary search | Look up the index page and jump straight to the chapter |
| ≈ $\log_2 N$ steps, e.g. **10 steps for 1000 blocks** | Much faster than binary search |

| Book index page | | Database index file | |
|---|---|---|---|
| Chapter 1 | page 2 | Search key 1 | address of block 1 |
| Chapter 2 | page 40 | Search key 10 | address of block 2 |
| Chapter n | page 100 | Search key 21 | address of block 3 |

Library example: a catalogue **indexed by Author** answers "Where are the books by Rui?", and a catalogue **indexed by Title** answers "Where is 'Database'?" (pointing to both *Database by Rao* and *Database by Rui*). One file (the books) can have **several indexes on different search keys**.

### Two basic kinds of indices

- **Ordered indices** – search keys are stored in **sorted order**.
- **Hash indices** – search keys are distributed across **buckets** by a **hash function** (§2).

### Index evaluation metrics

- **Access types** supported efficiently:
  - records with a specified attribute value (point/equality query)
  - records with an attribute value in a specified range (range query)
- **Access time**
- **Insertion time**
- **Deletion time**
- **Space overhead**

## 3.2 Types of indexing

```
                    Indexing methods
          ┌─────────────────┼──────────────────┐
   Ordered indices     Clustering index    Secondary index
          │
    Primary index
     ┌────┴────┐
   Dense     Sparse
                                  (+ Multilevel indexes, B-tree / B+-tree)
```

**Running example table** (used by the *Types of Indexes* slides):

```sql
CREATE TABLE Student (
    RollNo     INT PRIMARY KEY,
    Name       VARCHAR(50),
    Department VARCHAR(20),
    Marks      INT
);
```

| RollNo | Name | Department | Marks |
|---|---|---|---|
| 101 | Arun | CSE | 85 |
| 102 | Banu | ECE | 78 |
| 103 | Chitra | CSE | 92 |
| 104 | David | IT | 88 |
| 105 | Ezhil | CSE | 75 |

```sql
CREATE INDEX idx_rollno ON Student(RollNo);
```

## 3.3 Ordered index

- Index entries are kept in **sorted order** (ascending or descending) of the search key, like a book index or library catalogue. Each key is associated with the records that contain it.
- Useful for **equality searches** and especially **range searches**.

| Index key | Points to |
|---|---|
| 101 | Record 101 |
| 102 | Record 102 |
| 103 | Record 103 |
| 104 | Record 104 |
| 105 | Record 105 |

**Multiple indices on one file.** A table can have several indexes, each on a different search key, and each helps a different kind of query:

```sql
CREATE INDEX idx_department ON Student(Department);
CREATE INDEX idx_marks      ON Student(Marks);
CREATE INDEX idx_name       ON Student(Name);
```

Ordered indices are either **clustered** or **non-clustered**.

## 3.4 Clustered vs. non-clustered index

### Clustered index

A **clustered index** determines the **physical order of the rows** in the table according to the indexed column.

```sql
CREATE CLUSTERED INDEX idx_rollno ON Student(RollNo);   -- SQL Server syntax
SELECT * FROM Student WHERE RollNo BETWEEN 102 AND 104;  -- range search: rows are adjacent on disk
```

| idx_rollno | Actual data (stored in this order) |
|---|---|
| 101 | Arun, CSE, 85 |
| 102 | Banu, ECE, 78 |
| 103 | Chitra, CSE, 92 |
| 104 | David, IT, 88 |
| 105 | Ezhil, CSE, 75 |

- Sorts the data physically.
- Very good for **range-based searches**.
- **Only one clustered index per table**, because rows can be physically sorted only one way.

```sql
CREATE CLUSTERED INDEX idx_student_age ON Student(Age);  -- data stored in order of Age
```

### Non-clustered index

A separate index structure containing the **search key** and a **pointer/reference to the row**. The **physical order of the table does not change**.

The table stays physically ordered by `RollNo`, and we index `Name`:

```sql
CREATE NONCLUSTERED INDEX idx_name ON Student(Name);
SELECT * FROM Student WHERE Name = 'Chitra';   -- idx_name: Chitra → Row 103
```

| idx_name | Pointer / reference |
|---|---|
| Arun | → Row 101 |
| Banu | → Row 102 |
| Chitra | → Row 103 |
| David | → Row 104 |
| Ezhil | → Row 105 |

## 3.5 Primary index

- Defined on the **ordering key field** of an **ordered file**. The file is physically sorted on a field whose value is **unique** for every record, i.e. the primary key.
- The index file is itself an ordered file of **fixed-length entries with two fields**: <primary key value, pointer to a disk block>.
- There is **one index entry per data block**. Its key is the first record of the block, called the **block anchor**.
- Many DBMSs automatically create a primary index when a `PRIMARY KEY` is declared. It enforces uniqueness and is used for point lookups.

```sql
CREATE TABLE Student ( StudentID INT PRIMARY KEY, Name VARCHAR(50), Age INT );
-- Primary index created on StudentID
```

**Slide example (block anchors, 4 records per block):**

| Primary index (Student No → block ptr) | Data block contents (Student ID, Branch No) |
|---|---|
| 1001 → Block 1 | (1001,1) (1002,2) (1003,2) (1004,1) |
| 1005 → Block 2 | (1005,1) (1006,3) (1007,4) (1008,3) |
| 1009 → Block 3 | (1009,3) (1010,3) (1011,4) (1012,1) |
| 1013 → Block 4 | (1013,2) (1014,2) … |

**Textbook example (Elmasri Fig. 17.1)**, where the ordering key is `Name`:

| Index entry $\langle K(i), P(i) \rangle$ | Data block |
|---|---|
| Aaron, Ed | Aaron, Ed; Abbot, Diane; …; Acosta, Marc |
| Adams, John | Adams, John; Adams, Robin; …; Akers, Jan |
| Alexander, Ed | Alexander, Ed; Alfred, Bob; …; Allen, Sam |
| Allen, Troy | Allen, Troy; Anders, Keith; …; Anderson, Rob |
| Anderson, Zach | Anderson, Zach; Angel, Joe; …; Archer, Sue |
| Arnold, Mack | Arnold, Mack; Arnold, Steven; …; Atkins, Timothy |

**Disadvantage:** inserting and deleting records means moving records around and changing index entries (block anchors). **Solutions:** use an unordered **overflow file**, or a **linked list of overflow records** for each block.

A primary index can be **dense** or **sparse**. The textbook primary index is sparse, with one entry per block.

## 3.6 Dense index

A **dense index** has an index entry for **every search-key value** in the file (for a unique key, effectively one per record).
- Searches are faster (direct pointer to the record), but it **needs more space**.

| Index | Data record |
|---|---|
| 101 → | Arun |
| 102 → | Banu |
| 103 → | Chitra |
| 104 → | David |
| 105 → | Ezhil |

Slide diagrams:
- Numeric: index 10, 20, 30, 40, 50, 60, 70, 80 → each points to its own record (records are stored 2 per block: [10, 20] [30, 40] [50, 60] [70, 80]).
- Countries: China, Canada, Russia, USA → each points to its own record, e.g. (China, Beijing, 3,705,386).

**Dense index on `ID` of `instructor`:** every ID has an entry.

| ID | Name | Dept | Salary |
|---|---|---|---|
| 10101 | Srinivasan | Comp. Sci. | 65000 |
| 12121 | Wu | Finance | 90000 |
| 15151 | Mozart | Music | 40000 |
| 22222 | Einstein | Physics | 95000 |
| 32343 | El Said | History | 60000 |
| 33456 | Gold | Physics | 87000 |
| 45565 | Katz | Comp. Sci. | 75000 |
| 58583 | Califieri | History | 62000 |
| 76543 | Singh | Finance | 80000 |
| 76766 | Crick | Biology | 72000 |
| 83821 | Brandt | Comp. Sci. | 92000 |
| 98345 | Kim | Elec. Eng. | 80000 |

**Dense index on `dept_name`** (file sorted by dept_name): the entries Biology → Crick, Comp. Sci. → Srinivasan, Elec. Eng. → Kim, Finance → Wu, History → El Said, Music → Mozart and Physics → Einstein each point to the **first** record with that value. The rest of the group follows sequentially.

### Numerical example – dense index

- Data file: **1,000,000 tuples**, 10 per 4 KB (4096-byte) block.
- Data blocks $= 1{,}000{,}000 / 10 = 100{,}000$ blocks; data file size $= 100{,}000 \times 4\text{ KB} = 400\text{ MB}$.
- Index entry = key 30 B + pointer 8 B = **38 B**.
- Entries per index block $= \lfloor 4096 / 38 \rfloor = 107 \approx$ **100** (the slides round to 100).
- Index blocks $= 1{,}000{,}000 / 100 = 10{,}000$ blocks → **40 MB** (might fit in main memory).

## 3.7 Sparse index

A **sparse index** has entries for **only some** search-key values, typically **one per data block** (the least key in the block).
- It is only applicable when records are **sequentially ordered on the search key**.
- A range of keys shares the same block address.
- It needs less space and less maintenance on insert/delete, but it is **slower** than a dense index at locating a record.

| Sparse index entry | Data block |
|---|---|
| 101 → | Block 1 (101, 102, 103) |
| 104 → | Block 2 (104, 105, 106) |
| 107 → | Block 3 (107, …) |

**To locate a record with key $K$:**
1. Find the index entry with the **largest search-key value ≤ K**.
2. Follow its pointer and **search the file sequentially** from there.

**Sparse index on `ID` of `instructor`:** entries 10101 → Srinivasan, 32343 → El Said and 76766 → Crick. To find 22222: the largest entry ≤ 22222 is 10101, so scan from Srinivasan → Wu → Mozart → **Einstein**.

Slide numeric diagram: sparse entries 10 → [10, 20], 30 → [30, 40], 70 → [70, 80]. To find 50 or 60, start at entry 30 and scan forward.

### Numerical example – sparse index (same file as above)

- One (key, pointer) entry for the **first record of every block** → 100,000 entries.
- $100{,}000 \times 38\text{ B} \approx 3.8\text{ MB}$ → $100{,}000 / 100 = 1{,}000$ blocks ≈ **4 MB**.
- If the index fits in main memory, finding a record takes **1 disk I/O**.

### Cost of lookup

Binary search on the index needs about $\lceil \log_2(\text{number of index blocks}) \rceil$ I/Os:

$$\text{Dense: } \lceil \log_2 10{,}000 \rceil = 14 \text{ I/Os} \qquad \text{Sparse: } \lceil \log_2 1{,}000 \rceil = 10 \text{ I/Os} \quad (+1 \text{ for the data block})$$

Every binary search starts at the **middle** block, then the 1/4 and 3/4 points, then 1/8, 3/8, 5/8, 7/8, and so on. **Keeping these frequently used index blocks in main memory reduces I/Os significantly.**

### Dense vs. sparse

| Feature | Dense | Sparse |
|---|---|---|
| Entries | Every search-key value/record | Only selected keys/blocks |
| Space | More | Less |
| Search | Find $K$ directly and follow the pointer | Find the **largest key ≤ K**, follow the pointer to the block, then search the block |
| Existence test | Can answer "is there a record with key $K$?" **from the index alone** | **Cannot**; must read the data block |
| Requirement | Works for any file | File must be **sorted** on the search key |

**Memory trick:** DENSE = EVERY; SPARSE = SOME.

**Good trade-off:**
- For a **clustered index**: a sparse index with one entry per block (the least key in the block).
- For an **unclustered index**: a sparse index on top of a dense index, i.e. a **multilevel index**.

## 3.8 Clustering index

- Used when the file is **physically ordered on a non-key field** (one without distinct values), so **many records share the same value** of that field.
- The index file is an ordered file with two fields: <**clustering field value**, **pointer to the first disk block** containing that value>.
- There is **one index entry per distinct value**, so it is a **sparse** index.
- Example: employees stored grouped by `Dept_no`. Each department is one cluster, and the index points to the cluster as a whole.

| Department | Students grouped together |
|---|---|
| CSE | Arun, Chitra, Ezhil |
| ECE | Banu |
| IT | David |

```sql
CREATE INDEX idx_department ON Student(Department);
```

**Slide diagram – clustering index on Dept ID** (blocks chained by block pointers):

| Index (Dept ID) | → Block | Block contents |
|---|---|---|
| 1 | Block 1 | 1, 1, 1, 2 |
| 2 | Block 1 (first "2" is its last record) | – |
| – | Block 2 | 2, 2, 2, 2 |
| 3 | Block 3 | 3, 3, 3, 4 |
| 4 | Block 3 (first "4" is its last record) | – |
| 5 | Block 4 | 4, 5, 5, 5 |

Each index pointer goes to the **block containing the first record** with that value.

> The slides also describe a clustered index as one where "the records themselves are stored in the index, not pointers" (an index-organised table). When the key isn't unique, you can combine two or more columns to make it unique.

## 3.9 Secondary index

- An **additional access path** on a **non-ordering field**, which may be a candidate key or a non-key column. Also called a **non-clustering index**.
- The index file has two fields: <indexing field value, **block pointer or record pointer**>.
- Because the file is **not sorted** on this field, the index **must be dense**: one entry per record for a key field, or per distinct value with a **bucket of pointers** for a non-key field.
- It usually needs **more storage and a longer search time** than a primary index, because it is dense.

| Department | Matching RollNos |
|---|---|
| CSE | 101, 103, 105 |
| ECE | 102 |
| IT | 104 |

```sql
CREATE INDEX idx_dept ON Student(Department);
```

**Bank example:** data is stored sequentially by `acc_no`, but we want all accounts of a specific branch. A secondary index on `branch` maps each branch value to a bucket of pointers to all its records.

**Slide diagram – two-level secondary index on an unsorted field:** data blocks [10, 20] [10, 40] [40, 20] [10, 30]. Index entry 10 → bucket of 3 record pointers, 20 → bucket of 2, 30 → bucket of 1, 40 → bucket of 2.

**Secondary index on `salary` of `instructor`** (file ordered by ID):

| Salary (index) | Bucket → record(s) |
|---|---|
| 40000 | Mozart |
| 60000 | El Said |
| 62000 | Califieri |
| 65000 | Srinivasan |
| 72000 | Crick |
| 75000 | Katz |
| 80000 | **Singh, Kim** (bucket with 2 pointers) |
| 87000 | Gold |
| 90000 | Wu |
| 92000 | Brandt |
| 95000 | Einstein |

> **Secondary indices have to be dense.**

## 3.10 Summary of single-level ordered indexes (Supplementary – Elmasri table)

| | Ordering field of file | Non-ordering field |
|---|---|---|
| **Key field** | Primary index (sparse / non-dense) | Secondary index (key) – dense |
| **Non-key field** | Clustering index (sparse) | Secondary index (non-key) – dense or bucket-based |

| Index type | Number of first-level entries | Dense or sparse | Block anchoring on data file |
|---|---|---|---|
| Primary | Number of blocks in the data file | Sparse | Yes |
| Clustering | Number of distinct field values | Sparse | Yes / No |
| Secondary (key) | Number of records | Dense | No |
| Secondary (non-key) | Number of records or distinct values | Dense or sparse | No |

**Quick revision – 6 words:** Ordered = **SORTED**, Primary = **PRIMARY KEY**, Dense = **EVERY**, Sparse = **SOME**, Clustering = **GROUP**, Secondary = **ADDITIONAL**.

> These categories overlap. An index can be both ordered and dense, and a primary index is sparse when the file is ordered by the primary key.

## 3.11 Multilevel indexes

- If the (first-level) index is itself too large to fit in memory, build a **primary index on the index**. The index file is ordered with unique keys, so this second level can be sparse.
- **First (base) level** = the original index file.
- **Second level** = a primary index to the first level.
- **Third level** = a primary index to the second level, and so on until the top level fits in **one block**.
- Each level cuts the remaining search space by a factor of the **fan-out** $fo$ (index entries per block), not by 2 as in binary search.

$$\text{Number of levels } t = \left\lceil \log_{fo}(r_1) \right\rceil, \qquad \text{I/Os for a search} = t + 1$$

where $r_1$ is the number of first-level entries.

**Slide diagram – two-level primary index resembling ISAM** (Elmasri Fig. 17.6):

```
Second (top) level:   [ 2 | 35 | 55 | 85 ]
                         │    │    │    └──────────────────┐
First (base) level:  [2 8 15 24] [35 39 44 51] [55 63 71 80] [85]
                      │ │  │  │    │  │  │  │    │  │  │  │    │
Data blocks:        (2,5)(8,12)(15,21)(24,29)(35,36)(39,41)(44,46)(51,52)(55,58)(63,66)(71,78)(80,82)(85,89)
```

Another slide shows an Outer index (in RAM) → Inner index blocks → Data blocks. For example, primary-level entries 100, 200, 300 point to secondary-level blocks {100, 110, 120}, {200, 210, 220} and {300, 310, 320}, which point to the data blocks.

**Worked example (Supplementary), using the dense index from §3.6:** the first level has 10,000 blocks. With $fo = 100$, the second level has $\lceil 10{,}000/100 \rceil = 100$ blocks and the third level has 1 block. So $t = 3$, and a search costs **3 index I/Os + 1 data I/O = 4**, compared with 14 + 1 for binary search.

**Dynamic multilevel indexes:** a static multilevel index (ISAM) degrades with insertions and deletions, because overflow blocks build up. **B-trees and B+-trees** are multilevel indexes that **stay balanced automatically** and leave space in each node for inserts. That is the next section.

## 3.12 Advantages and disadvantages of indexing

| Advantages | Disadvantages |
|---|---|
| Fewer total I/O operations to retrieve data | Extra **storage** for index structures |
| Faster search and retrieval for users | **Slower INSERT, UPDATE and DELETE**, because every index must be maintained |
| An index-organised table needs no separate ROWID, which saves tablespace | The slides also list some Oracle index-organised-table restrictions: it needs a primary key with unique values, other indexes can't be built on the indexed data, and the table can't be partitioned |

The DBMS **optimiser** decides whether using an index is cheaper than a full table scan:

```sql
SELECT * FROM Student WHERE RollNo = 104;               -- point lookup (primary index)
SELECT * FROM Student WHERE Department = 'CSE';         -- secondary index
SELECT * FROM Student WHERE RollNo BETWEEN 102 AND 104; -- range search (clustered / B+ tree)
```

## 3.13 Index definition in SQL

```sql
CREATE INDEX <index-name> ON <relation-name> (<attribute-list>);
-- e.g.
CREATE INDEX b_index ON branch(branch_name);

CREATE UNIQUE INDEX ...   -- also enforces that the search key is a candidate key
                          -- (not really needed if the UNIQUE constraint is supported)
DROP INDEX <index-name>;
```

Most database systems also let you specify the **type of index** (B+-tree, hash, etc.) and **clustering**.

---

# 4. B+ Tree Indexing

## 4.1 Why B+ trees?

| Indexed-sequential (ISAM) files | B+-tree index files |
|---|---|
| Performance degrades as the file grows, because many overflow blocks are created | **Automatically reorganises itself** with small, local changes on insert/delete |
| Periodic reorganisation of the entire file is required | No reorganisation of the whole file is needed |

The minor disadvantages of B+-trees are extra insert/delete overhead and space overhead, since nodes can be half empty. **The advantages outweigh the disadvantages**, which is why B+-trees are used extensively (the default index in almost every RDBMS).

## 4.2 Properties

- A **balanced multiway search tree** used as a **dynamic multilevel index**. The slides say "balanced binary search tree", but each node has many children.
- **All leaves are at the same depth**, so every root-to-leaf path has the same length.
- **Internal (non-leaf) nodes hold only keys and child pointers.** They form a sparse multilevel index that guides the search.
- **Leaf nodes hold every key**, each with a pointer to its record or to a bucket of records.
- **Leaves are linked** in key order by a next-leaf pointer, so the tree supports both **random** (root → leaf) and **sequential/range** (leaf → leaf) access.

```
B+ tree (from the slides):
                         [ 9 ]
                  ┌────────┴─────────┐
                [ 5 ]             [13 | 17]
              ┌───┴───┐        ┌─────┼──────┐
          [1 3] ⇄ [5 7] ⇄ [9 11] ⇄ [13 15] ⇄ [17]        ← linked leaves
```

```
Labelled structure (slide):
Level 0 (root)        [5]                    ← internal node / child pointers
Level 1           [3]      [7 | 8]           ← search-key values
Level 2 (leaf)  [1 3]→[5]→[6 7]→[8]→[9 12]   ← key values, data pointers, sibling pointers
```

### B+ tree node structure

$$\boxed{P_1 \mid K_1 \mid P_2 \mid K_2 \mid \cdots \mid P_{n-1} \mid K_{n-1} \mid P_n}$$

- $K_i$ are search-key values, with $K_1 < K_2 < \dots < K_{n-1}$ (assume no duplicates).
- In a **non-leaf node**, $P_i$ points to the child subtree containing keys $K_{i-1} \le x < K_i$.
- In a **leaf node**, $P_i$ ($i = 1 \dots n-1$) points to the record with key $K_i$, and $P_n$ **points to the next leaf** in key order.
- If $L_i$ and $L_j$ are leaves with $i < j$, every key in $L_i$ is ≤ every key in $L_j$.

### Example B+ tree on `instructor.name` (slide)

```
                          [ Mozart ]
               ┌─────────────┴──────────────┐
        [ Einstein | Gold ]            [ Srinivasan ]
      ┌────────┼─────────┐             ┌──────┴───────┐
[Brandt Califieri Crick]→[Einstein El Said]→[Gold Katz Kim]→[Mozart Singh]→[Srinivasan Wu]
```

Each leaf key points to its instructor record; e.g. Brandt → (83821, Comp. Sci., 92000).

B-tree index slide example ($n = 3$, handwritten): root [100]; children [30] and [120 150 180]; leaves [3 5 11] → [30 35] → [100 101 110] → [120 130] → [150 157 179] → [180 200].

## 4.3 Order and occupancy rules

**Order $m$** = the maximum number of **child pointers** in a node.

The slides give this table:

| Property | Root node | Internal node (non-root) | Leaf node |
|---|---|---|---|
| Maximum children | $m$ | $m$ | – |
| Minimum children | 2 (if internal) or 1 (if leaf) | $\lceil m/2 \rceil$ | – |
| Maximum keys | $m-1$ | $m-1$ | $m$ (see note) |
| Minimum keys | 1 (if internal) or 0 (if leaf) | $\lceil m/2 \rceil - 1$ | $\lceil m/2 \rceil$ (see note) |

> ⚠️ **Convention used in all the worked examples** (and in the construction problem slide):
>
> $$\text{max keys in any node} = m-1, \quad \text{min children (internal)} = \lceil m/2 \rceil, \quad \text{min keys (internal)} = \lceil m/2 \rceil - 1, \quad \text{min keys (leaf)} = \left\lceil \frac{m-1}{2} \right\rceil$$
>
> The order-3 example overflows a leaf at 3 keys, and the order-4 examples overflow a leaf at 4 keys. So in practice **leaf max = m − 1**, not m. For $m = 3$: leaf holds 1–2 keys. For $m = 4$: leaf holds 2–3 keys. **Use this convention in exams unless told otherwise, and state it.**

| | $m = 3$ | $m = 4$ |
|---|---|---|
| Max children | 3 | 4 |
| Max keys (every node) | 2 | 3 |
| Min children (non-root internal) | $\lceil 3/2 \rceil = 2$ | 2 |
| Min keys (non-root internal) | 1 | 1 |
| Min keys (leaf) | $\lceil 2/2 \rceil = 1$ | $\lceil 3/2 \rceil = 2$ |
| Root | at least 2 children (unless it is a leaf) | same |

## 4.4 Search

1. Start at the root.
2. In each internal node, find the smallest $K_i$ with $x < K_i$ and follow $P_i$. If $x \ge$ every key, follow the last pointer.
3. At the leaf, look for $x$. **Range query $[a, b]$:** find the leaf for $a$, then follow the leaf links until a key exceeds $b$.

Cost ≈ **height of the tree** (a few I/Os even for millions of keys), because height $\approx \lceil \log_{\lceil m/2 \rceil} N \rceil$.

## 4.5 Insertion

**Algorithm:**
1. Search for the leaf $N$ where the new key $D$ belongs.
2. Insert $D$ into $N$ in sorted order.
   - **Case I:** $N$ has space → done.
   - **Case II:** $N$ is full → **overflow**. Either transfer a key to a sibling that has room, **or split**:
     - **Leaf split:** divide the $m$ keys into two leaves (left gets $\lfloor m/2 \rfloor$, right gets the rest). **Copy** the first key of the right leaf **up** into the parent; it stays in the leaf too.
     - **Internal split:** divide the $m$ keys. The middle key (position $\lfloor m/2 \rfloor + 1$) **moves up** (push up, not kept in either half).
     - If the parent overflows, repeat. If the root splits, a new root is created and **the height grows by 1**.

> **Leaf split → copy up. Internal split → push up.** This is the most-asked detail.

### Example 1 (slides): order $m = 3$, insert 5, 15, 25, 35, 45

Max keys 2, min internal keys 1, leaf capacity 2.

```
Insert 5:   [5]
Insert 15:  [5 15]
Insert 25:  [5 15 25] overflow → split → [5] | [15 25], copy 15 up
                 [15]
               /      \
            [5]  →  [15 25]
Insert 35:  leaf [15 25 35] overflow → [15] | [25 35], copy 25 up
                 [15 25]
               /    |    \
            [5] → [15] → [25 35]
Insert 45:  leaf [25 35 45] overflow → [25] | [35 45], copy 35 up
            root [15 25 35] overflow → internal split, push 25 up
                       [25]
                     /      \
                 [15]        [35]
                /    \      /    \
             [5] → [15] → [25] → [35 45]
```

### Example 2 (slides): order $m = 4$, insert 10, 4, 90, 8, 1

Max keys 3.

```
Insert 10:  [10 _ _]
Insert 4:   [4 10 _]
Insert 90:  [4 10 90]                     (full)
Insert 8:   [4 8 10 90]  overflow
            split → [4 8] | [10 90], copy 10 up
                  [10]
                /      \
           [4 8]  →  [10 90]
Insert 1:   1 < 10 → left leaf → [1 4 8]  (fits, no split)
                  [10]
                /      \
          [1 4 8] →  [10 90]
```

### Example 3 (slide problem): order $m = 4$, insert 2, 4, 6, 8, 10, 12, 14, 16, 18, 20

Max keys 3, min internal keys 1, min leaf keys 2. **Promote the first key of the right node.**

```
Insert 2,4,6:  [2 4 6]
Insert 8:      [2 4 6 8] overflow → [2 4] | [6 8], promote 6
                     [6]
                   /     \
               [2 4]    [6 8]
Insert 10:     right leaf [6 8 10]
Insert 12:     [6 8 10 12] overflow → [6 8] | [10 12], promote 10
                     [6 10]
                   /   |   \
               [2 4] [6 8] [10 12]
Insert 14:     [10 12 14]
Insert 16:     [10 12 14 16] overflow → [10 12] | [14 16], promote 14
                     [6 10 14]
                  /    |    |    \
              [2 4] [6 8] [10 12] [14 16]
Insert 18:     [14 16 18]
Insert 20:     [14 16 18 20] overflow → [14 16] | [18 20], promote 18
               root becomes [6 10 14 18] → overflow (max 3 keys)
               ROOT SPLIT: left internal [6 10], push 14 up, right internal [18]

                              [14]
                        /              \
                   [6 10]              [18]
                 /   |    \           /     \
             [2 4] [6 8] [10 12]  [14 16] [18 20]
```

✔ Height increased by 1. ✔ All nodes are valid.

### Example 4 (homework on slides): keys F, S, Q, K, C, L, H, T, V, W, M, R

(Solved here using the convention above, in alphabetical order.)

**Order 3** (max 2 keys; leaf split 1 | 2 with copy-up; internal split pushes the middle key up):

| Step | Key | What happens |
|---|---|---|
| 1 | F | [F] |
| 2 | S | [F S] |
| 3 | Q | [F Q S] overflow → [F] \| [Q S], root [Q] |
| 4 | K | K < Q → [F K] |
| 5 | C | [C F K] overflow → [C] \| [F K], copy F → root [F Q] |
| 6 | L | F ≤ L < Q → [F K L] overflow → [F] \| [K L], copy K → root [F K Q] overflow → push K up: new root [K], children [F] and [Q] |
| 7 | H | → leaf [F] → [F H] |
| 8 | T | → leaf [Q S] → [Q S T] overflow → [Q] \| [S T], copy S → internal [Q S] |
| 9 | V | → [S T V] overflow → [S] \| [T V], copy T → internal [Q S T] overflow → push S up: root [K S], internals [Q] and [T] |
| 10 | W | → [T V W] overflow → [T] \| [V W], copy V → internal [T V] |
| 11 | M | K ≤ M < S, M < Q → leaf [K L] → [K L M] overflow → [K] \| [L M], copy L → internal [L Q] |
| 12 | R | K ≤ R < S, R ≥ Q → leaf [Q] → [Q R] |

```
Order 3 final:
                               [K  S]
              ┌──────────────────┼────────────────────┐
             [F]               [L  Q]                [T  V]
           ┌──┴──┐          ┌────┼─────┐          ┌───┼────┐
          [C]  [F H]       [K] [L M] [Q R]       [S] [T] [V W]

Leaf chain: C → F H → K → L M → Q R → S → T → V W
```

**Order 4** (max 3 keys; leaf split 2 | 2 with copy-up):

| Step | Key | What happens |
|---|---|---|
| 1–3 | F, S, Q | [F Q S] |
| 4 | K | [F K Q S] overflow → [F K] \| [Q S], root [Q] |
| 5 | C | [C F K] |
| 6 | L | [C F K L] overflow → [C F] \| [K L], copy K → root [K Q] |
| 7 | H | [C F H] |
| 8 | T | [Q S T] |
| 9 | V | [Q S T V] overflow → [Q S] \| [T V], copy T → root [K Q T] |
| 10 | W | [T V W] |
| 11 | M | [K L M] |
| 12 | R | [Q R S] |

```
Order 4 final:
                      [K  Q  T]
          ┌─────────┬────┴────┬─────────┐
      [C F H] → [K L M] → [Q R S] → [T V W]
```

## 4.6 Deletion

**Algorithm:**
1. Find the leaf $N$ containing key $K$ and delete $K$ from it.
2. If $N$ still has **at least the minimum** number of keys → done. Update a parent separator if needed.
3. Otherwise **underflow**:
   - **Borrow (redistribute)** a key from an immediate sibling that has more than the minimum, and update the separator key in the parent, **or**
   - **Merge** $N$ with a sibling and remove the separator from the parent. This may make the parent underflow, and the fix continues upward.
4. If the root ends up with no keys (only one child), remove it: **the height decreases by 1**.

### Deletion cases (slides, order $m = 3$: max 2 keys, min 1 key per non-root node)

Starting tree:

```
                    [25]
             ┌───────┴────────┐
           [15]             [35 45]
         ┌──┴───┐       ┌─────┼──────┐
        [5]  [15 20]  [25 30] [35 40] [45 55]
```

**Case I(a) – key only in a leaf, leaf has more than the minimum → just delete.** Delete **40**:

```
                    [25]
           [15]             [35 45]
        [5]  [15 20]  [25 30] [35] [45 55]
```

**Case I(b) – key only in a leaf, leaf is at the minimum → delete, borrow from the immediate sibling, update the parent.** Delete **5**: leaf [5] becomes empty. Borrow 15 from the right sibling [15 20]; the leaves become [15] and [20], and the parent key changes 15 → 20.

```
                    [25]
           [20]             [35 45]
       [15]  [20]    [25 30] [35] [45 55]
```

**Case II(a) – key is in both a leaf and an internal node.** Delete it from the leaf, delete it from the internal node, and replace it with its **in-order successor**. Delete **45**: leaf [45 55] → [55], and internal [35 45] → [35 55] (55 is the in-order successor).

```
                    [25]
           [20]             [35 55]
       [15]  [20]    [25 30] [35] [55]
```

**Case II(b) – key is in a leaf and an internal node, and the deletion leaves the leaf below the minimum.** Delete from the leaf, **borrow from the immediate sibling through the parent**, and use the borrowed key to fill the internal node. Delete **35**: leaf [35] becomes empty. Borrow 30 from the left sibling [25 30]; the leaves become [25] and [30], and internal [35 55] → [30 55].

```
                    [25]
           [20]             [30 55]
       [15]  [20]      [25] [30] [55]
```

**Case II(c) – the deletion causes underflow that reaches the parent.** Delete the key, **merge with the sibling**, and update the **grandparent** using the in-order successor. Delete **25** (it is in a leaf and in the root): leaf [25] becomes empty and merges away. The root key 25 is replaced by its in-order successor 30, which leaves internal [30 55] → [55].

```
                    [30]
           [20]             [55]
       [15]  [20]        [30]  [55]
```

**Case III – underflow propagates to the root, so the height shrinks.** Delete **55**: leaf [55] is removed, so internal [55] has only one child and underflows. It can't borrow from [20] (which is at the minimum), so it **merges**: the root key 30 comes down and joins [20] to form [20 30]. The root is now empty, so it is removed and **the height decreases by 1**.

```
             [20  30]
          ┌─────┼──────┐
        [15]  [20]   [30]
```

### Deletion example from the construction problem (order $m = 4$; delete 6, 8, 10, 12)

Start from the final tree of Example 3 (min leaf keys = 2, min internal keys = 1):

```
                              [14]
                   [6 10]              [18]
             [2 4] [6 8] [10 12]   [14 16] [18 20]
```

**Delete 6:** leaf [6 8] → [8] → **underflow** (min 2). The siblings [2 4] and [10 12] each have exactly 2 keys, so neither can lend one → **merge with the left sibling**: [2 4 8]. Remove separator 6 from the parent → [10].

```
                              [14]
                   [10]                [18]
             [2 4 8]  [10 12]      [14 16] [18 20]
```

**Delete 8:** [2 4 8] → [2 4]. Valid (2 keys).

**Delete 10:** leaf [10 12] → [12] → underflow. The left sibling [2 4] has only 2 keys, so **merge**: [2 4 12]. The parent internal node [10] loses its only key, has 0 keys, and **underflows**.

**Internal node merge:** merge the empty internal node with its sibling [18], **bringing down the root key 14** → new internal node [14 18] with children [2 4 12], [14 16], [18 20]. The root now has no keys, so it is removed and **the height is reduced**.

```
                 [14  18]
          ┌─────────┼─────────┐
     [2 4 12]    [14 16]    [18 20]
```

**Delete 12** (the slides stop before this step): [2 4 12] → [2 4], which is still valid. **Final tree:**

```
                 [14  18]
          ┌─────────┼─────────┐
       [2 4]     [14 16]    [18 20]
```

(Separators 14 and 18 still correctly divide the leaves; an internal key does not have to exist in a leaf.)

### Example: Delete 52 (slide)

Tree (internal nodes hold up to 2 keys, leaves up to 3):

```
                                [22]
                  [16]                              [41]
          [11]            [18]              [28]              [58]
      [1 8]  [11 15]  [16 17] [18 19]   [22 23] [28 31]   [41 52] [58 59 61]
```

The slides show only the "before" tree. **Solution** (leaf min = 2, i.e. the $m = 4$ leaf rule):
- Delete 52 → leaf [41 52] becomes [41] → underflow.
- The right sibling [58 59 61] has 3 keys, more than the minimum, so **borrow** 58 → leaves [41 58] and [59 61].
- Update the parent separator 58 → **59**.

```
                                [22]
                  [16]                              [41]
          [11]            [18]              [28]              [59]
      [1 8]  [11 15]  [16 17] [18 19]   [22 23] [28 31]   [41 58] [59 61]
```

(If the leaf minimum were 1, the leaf [41] would be valid and nothing else would change.)

---

# 5. LSM Trees (Log-Structured Merge Trees)

## 5.1 Limitations of B+ trees for write-heavy workloads

- **High random I/O**: every insert/update modifies a leaf page somewhere on disk.
- **High write amplification**: a whole page is rewritten to change one record, plus the splits.
- **High cost for frequent updates/inserts.**
- **Low write throughput** under heavy workloads.
- **High maintenance overhead** (page splits and merges).

➡ **LSM trees give high write throughput by using sequential writes.**

| | B-tree / B+ tree | LSM tree |
|---|---|---|
| Strength | Great for reads and range queries | **High write throughput** |
| Writes | Random writes → slower, higher write amplification | **Sequential writes → faster** |
| Used in | PostgreSQL, MySQL (InnoDB) | Cassandra, RocksDB, LevelDB, HBase |

## 5.2 What is an LSM tree?

- **LSM = Log-Structured Merge tree**, invented by **Patrick O'Neil et al. (1996)**.
- Instead of writing every record to its final place on disk immediately, an LSM tree:
  1. **first writes into memory**
  2. **later writes sequentially to disk** (as sorted files)
  3. **merges the files periodically** (compaction)
- **Sequential disk writes are much faster than random writes.**

**Why LSM trees?** Modern applications such as Facebook, Twitter, WhatsApp, banking transactions, IoT sensors and log analytics generate huge write loads. Suppose there are **10,000 inserts/second**. Updating a B+ tree for every insert causes random disk writes, frequent page splits and reduced performance. LSM trees solve this problem.

## 5.3 Architecture

```
            Write                                    Read
              │                                        │
              ├──────────► WAL (on disk, crash safety) │
              ▼                                        ▼
  RAM   ┌──────────┐  full   ┌────────────────────┐   (1) MemTable
        │ MemTable │ ──────► │ Immutable MemTable │   (2) Immutable MemTable
        └──────────┘         └─────────┬──────────┘   (3) Bloom filter → SSTables
 ──────────────────────────────────────┼────────────────────────────── (memory / disk boundary)
  DISK                    Flush (minor compaction)
                                       ▼
        Level 0 :  [SST][SST]                  ~10 MB   (key ranges may overlap)
                     │  major compaction (merge sort)
        Level 1 :  [SST][SST][SST]             ~100 MB  (non-overlapping, sorted)
                     │
        Level N :  [SST][SST][SST][SST]        ~1000 MB
```

Notes from the architecture slide (RocksDB style):
- The WAL can be turned off.
- A new WAL is created on each flush.
- A single WAL captures the write log for all column families.
- At L0, SST files may have overlapping key ranges. At L1–Ln, SST files contain non-overlapping, ordered key sequences.

## 5.4 Components

### (1) Write-Ahead Log (WAL)
- Before data goes into the MemTable, the operation is **appended to the WAL on disk**. This is a sequential append, so it is cheap.
- **Purpose: crash recovery.** It prevents data loss if the system crashes. During recovery the database replays the WAL to rebuild the MemTable.

```
INSERT Student 105  →  Write WAL  →  Update MemTable
```

### (2) MemTable
- Stored in **RAM**, and data is written here first.
- Usually implemented as a **balanced tree** (or a skip list), so it stays **sorted by key**.
- Writes are very fast because they are in memory.

| Student ID | Name |
|---|---|
| 101 | Alice |
| 102 | Bob |
| 103 | Charlie |

### (3) Immutable MemTable
When the MemTable becomes full, it is made **read-only (immutable)** and no more updates are allowed. A new empty MemTable takes new writes while the immutable one is being flushed.

### (4) SSTable (Sorted String Table)
- When the (immutable) MemTable is flushed, it is written to disk as an **immutable file** called an **SSTable**.
- Properties:
  - **sorted by key**
  - **cannot be modified after creation**
  - **new updates create new SSTables**
- Each SSTable usually carries a small **index** and a **Bloom filter**.

## 5.5 Why writes are fast

1. **Buffer writes in the MemTable.** Incoming data goes into a sorted in-memory structure, so the write succeeds instantly.
2. **Flush immutable SSTables to disk.** When the buffer is full, it is written **sequentially** as a permanent sorted file (sequential disk I/O).
3. **Background leveled compaction.** Overlapping files are merged into hierarchical, disjoint levels to keep things organised and speed up searches.
4. **Bloom filters cut read latency.** Probabilistic filters skip unnecessary disk checks.

## 5.6 Searching (read path)

Search order, **newest to oldest**, stopping as soon as the key is found:

```
MemTable → Immutable MemTable → Newest SSTable → … → Oldest SSTable
```

**Example: search for 109**

| Structure | Keys | Found? |
|---|---|---|
| MemTable | 150, 160, 170 | ✗ No |
| SSTable-3 (newest) | 130, 140 | ✗ No |
| SSTable-2 | 110, 120 | ✗ No |
| SSTable-1 (oldest) | 101, 105, 109 | ✓ **Found** |

### Bloom filter
- **Problem:** checking every SSTable wastes disk reads.
- **Solution:** keep a **Bloom filter** per SSTable. It is a probabilistic structure that answers either **"definitely NOT present"** or **"possibly present"**. There are no false negatives, but there can be false positives.
- **Example (search 500):** SSTable-1 filter → No; SSTable-2 → No; SSTable-3 → Maybe → **only SSTable-3 is searched**. This saves disk reads.
- **Example (search 350)** with SSTable 1 (keys 1–500), 2 (200–800), 3 (600–1200) and 4 (1000–1500): the MemTable doesn't have it, the Bloom filters skip SSTables 3 and 4, and only SSTables 1 and 2 are checked.

> Writes are fast because they go to memory. Reads pay the cost of checking multiple files, which Bloom filters and compaction reduce.

## 5.7 Compaction (merge)

After many flushes there are many small SSTables, which increases search cost:

```
SSTable-1: 100 101 102     SSTable-2: 103 104     SSTable-3: 105 106
                     ↓ background compaction (merge sort)
Merged:    100 101 102 103 104 105 106
```

**Benefits:**
- removes duplicates (keeps only the newest version)
- removes deleted records (tombstones)
- reduces the number of SSTables
- improves query speed

Compaction costs CPU and disk I/O.

## 5.8 Delete and update

- **Delete:** deleting in place is expensive, so a **tombstone** marker is written instead. For example, deleting ID = 103 is actually stored as `103 → Deleted`. The record is **removed permanently during compaction**.
- **Update:** LSM **never overwrites**. Updating a salary from 50000 to 55000 writes a **new version (55000)**. Reads see the newest version, and the **old version (50000) is removed during compaction**.

## 5.9 LSM tree variants (compaction strategies)

### Variant 1: Size-Tiered Compaction (STCS)
- **Used in:** Apache Cassandra (earlier versions), ScyllaDB (configurable).
- **Idea:** merge SSTables of **similar size**.

```
10 MB + 10 MB + 10 MB → 30 MB ;   30 MB + 30 MB → 60 MB
(also: 4 MB ×4 → 16 MB ;  16 MB ×2 → 32 MB)
```

- ✅ fast writes, less write amplification
- ❌ many SSTables, so read performance drops and more disk space is used

### Variant 2: Leveled Compaction (LCS)
- **Used in:** LevelDB, RocksDB, CockroachDB.
- **Idea:** organise SSTables into **levels**, each about **10× larger** than the previous one.

```
Level 0 : ~4 files (newest, smallest, e.g. 4 MB each)
Level 1 : 10× larger   (e.g. 40 MB, limit 80 MB)
Level 2 : 10× larger   (e.g. 400 MB, limit 800 MB)
Level 3 : …            (e.g. 4000 MB)
```

- Within a level (L1 and up), SSTables **don't overlap**, so a key is in at most one SSTable per level.
- ✅ excellent read performance, fewer SSTables to search
- ❌ more compaction work, higher write amplification

### Variant 3: Universal Compaction
- **Used in:** RocksDB.
- **Idea:** merge based on **age** instead of levels (old files merged → newest files).
- ✅ good for time-series data, lower write amplification

### Variant 4: FIFO Compaction
- Mostly used for **log databases and cache systems**.
- **Old files are simply deleted.** For example, with Day1, Day2, Day3, Day4, delete Day1.
- Useful when only recent data matters.

### Comparison of LSM variants

| Feature | Size-Tiered | Leveled | Universal | FIFO |
|---|---|---|---|---|
| Write speed | ★★★★★ | ★★★☆☆ | ★★★★☆ | ★★★★★ |
| Read speed | ★★★☆☆ | ★★★★★ | ★★★★☆ | ★★☆☆☆ |
| Compaction cost | Low | High | Medium | Very low |
| Storage overhead | High | Low | Medium | Very low |
| Best use case | Write-heavy systems | Read-heavy systems | Time-series databases | Logs and caches |

---

# 6. Relational Algebra

## 6.1 Role of relational algebra in a DBMS

```
SQL Query ──Parser──► Relational Algebra Expression ──Query Optimizer──► Query Execution Plan ──Code Generator──► Executable Code
```

- **Relational algebra (RA)** is the basic set of operations of the relational model. It lets a user specify basic retrieval requests.
- Every operation takes relation(s) as input and produces a **new relation**, so operations can be **composed** (closure property).
- A sequence of RA operations forms a **relational algebra expression**.
- RA is **procedural**: it says *how*, via the order of operations. SQL and relational calculus are **declarative**.

## 6.2 Classification of operators

| Category | Operators |
|---|---|
| **Unary** | SELECT $\sigma$, PROJECT $\pi$, RENAME $\rho$ |
| **From set theory** (binary) | UNION $\cup$, INTERSECTION $\cap$, DIFFERENCE $-$, CARTESIAN PRODUCT $\times$ |
| **Binary relational** | JOIN $\bowtie$ (theta, equi, natural), DIVISION $\div$ |
| **Additional/extended** | Aggregate functions & grouping ($\mathcal{F}$ or $\gamma$), outer joins (⟕ ⟖ ⟗) |

| Basic (fundamental) operators | Derived operators (expressible using basic ones) |
|---|---|
| $\sigma, \pi, \rho, \cup, -, \times$ | $\bowtie$ (join), $\div$ (division), $\cap$ (intersection) |

For example, $R \cap S = R - (R - S)$ and $R \bowtie_\theta S = \sigma_\theta(R \times S)$.

## 6.3 Example schema: COMPANY (Elmasri & Navathe)

```
EMPLOYEE      (FNAME, MINIT, LNAME, SSN, BDATE, ADDRESS, SEX, SALARY, SUPERSSN, DNO)
DEPARTMENT    (DNAME, DNUMBER, MGRSSN, MGRSTARTDATE)
DEPT_LOCATIONS(DNUMBER, DLOCATION)
PROJECT       (PNAME, PNUMBER, PLOCATION, DNUM)
WORKS_ON      (ESSN, PNO, HOURS)
DEPENDENT     (ESSN, DEPENDENT_NAME, SEX, BDATE, RELATIONSHIP)
```

Primary keys: SSN, DNUMBER, (DNUMBER, DLOCATION), PNUMBER, (ESSN, PNO), (ESSN, DEPENDENT_NAME).

## 6.4 SELECT ($\sigma$)

Selects the **subset of tuples (rows)** that satisfy a selection condition (predicate):

$$\sigma_{p}(r)$$

$\sigma$ is the select operator, $p$ is the predicate (a propositional-logic formula using $=, \ne, <, \le, >, \ge$ joined by $\wedge$ AND, $\vee$ OR, $\neg$ NOT), and $r$ is the relation.

- **Selection does not reduce columns, only rows.** The result has the same schema as $r$.
- $\sigma$ is **commutative**: $\sigma_{c_1}(\sigma_{c_2}(R)) = \sigma_{c_2}(\sigma_{c_1}(R)) = \sigma_{c_1 \wedge c_2}(R)$.

**Examples:**

$$\sigma_{SALARY > 30000}(EMPLOYEE)$$

selects the employees whose salary is more than 30,000.

```sql
SELECT * FROM Student WHERE dept = 'CSE';
```
$$\sigma_{dept='CSE'}(Student)$$

(WHERE → selection; FROM Student → relation; `SELECT *` → no projection.)

$$\sigma_{Id>3000 \,\vee\, Hobby='hiking'}(Person) \qquad \sigma_{Id>3000 \,\wedge\, Id<3999}(Person)$$
$$\sigma_{\neg(Hobby='hiking')}(Person) \;\equiv\; \sigma_{Hobby \ne 'hiking'}(Person)$$

$$\sigma_{(DNO=4 \,\wedge\, SALARY>25000) \,\vee\, (DNO=5 \,\wedge\, SALARY>30000)}(EMPLOYEE)$$

Result:

| FNAME | MINIT | LNAME | SSN | BDATE | ADDRESS | SEX | SALARY | SUPERSSN | DNO |
|---|---|---|---|---|---|---|---|---|---|
| Franklin | T | Wong | 333445555 | 1955-12-08 | 638 Voss, Houston, TX | M | 40000 | 888665555 | 5 |
| Jennifer | S | Wallace | 987654321 | 1941-06-20 | 291 Berry, Bellaire, TX | F | 43000 | 888665555 | 4 |
| Ramesh | K | Narayan | 666884444 | 1962-09-15 | 975 Fire Oak, Humble, TX | M | 38000 | 333445555 | 5 |

## 6.5 PROJECT ($\pi$)

Keeps **only the listed attributes (columns)** and **eliminates duplicate tuples**, because a relation is a set.

$$\pi_{LNAME, FNAME, SALARY}(EMPLOYEE)$$

| LNAME | FNAME | SALARY |
|---|---|---|
| Smith | John | 30000 |
| Wong | Franklin | 40000 |
| Zelaya | Alicia | 25000 |
| Wallace | Jennifer | 43000 |
| Narayan | Ramesh | 38000 |
| English | Joyce | 25000 |
| Jabbar | Ahmad | 25000 |
| Borg | James | 55000 |

$$\pi_{SEX, SALARY}(EMPLOYEE)$$

| SEX | SALARY |
|---|---|
| M | 30000 |
| M | 40000 |
| F | 25000 |
| F | 43000 |
| M | 38000 |
| M | 25000 |
| M | 55000 |

Only **7 rows** instead of 8. Alicia Zelaya and Joyce English are both (F, 25000), and that tuple appears **only once** in the result, because PROJECT removes duplicates. The (M, 25000) row is Ahmad Jabbar.

### SELECT + PROJECT

```sql
SELECT name, age FROM Student WHERE dept = 'CSE';
```
$$\pi_{name,\,age}\left(\sigma_{dept='CSE'}(Student)\right)$$

First $\sigma$ filters rows, then $\pi$ filters columns. **Selection is applied before projection** so that attributes needed for filtering aren't lost.

## 6.6 Sequences of operations and RENAME ($\rho$)

As a single nested expression:

$$\pi_{FNAME, LNAME, SALARY}\left(\sigma_{DNO=5}(EMPLOYEE)\right)$$

Or as a sequence with named intermediate relations:

$$\text{DEP5\_EMPS} \leftarrow \sigma_{DNO=5}(EMPLOYEE)$$
$$\text{RESULT} \leftarrow \pi_{FNAME, LNAME, SALARY}(\text{DEP5\_EMPS})$$

**RENAME** renames a relation and/or its attributes:
- $\rho_{a/b}(R)$ renames attribute $b$ of $R$ to $a$ (slide notation).
- General (Elmasri) forms: $\rho_{S(B_1,\dots,B_n)}(R)$ renames the relation to $S$ and its attributes to $B_1 \dots B_n$; $\rho_S(R)$ renames only the relation; $\rho_{(B_1,\dots,B_n)}(R)$ renames only the attributes.
- Example: $\text{RESULT}(\text{FirstName}, \text{LastName}, \text{Salary}) \leftarrow \pi_{FNAME, LNAME, SALARY}(\text{DEP5\_EMPS})$

## 6.7 Set operations: $\cup$, $\cap$, $-$

**Union compatibility** (required for all three):
1. Both relations have the **same number of attributes**.
2. Corresponding attributes have **compatible domains**.

Duplicates are removed automatically. By convention the result takes the **attribute names of the first relation**.

### UNION $R \cup S$ — tuples in $R$ or $S$ (or both)

**Example 1:**

| Course_1 | | | Course_2 | | | Course_1 ∪ Course_2 | |
|---|---|---|---|---|---|---|---|
| C_id | C_name | | C_id | C_name | | C_id | C_name |
| 11 | Foundation C | | 12 | Python | | 11 | Foundation C |
| 21 | C++ | | 21 | C++ | | 21 | C++ |
| 31 | JAVA | | | | | 31 | JAVA |
| | | | | | | 12 | Python |

**Example 2:**

```sql
SELECT sid FROM Enroll UNION SELECT sid FROM Student;
```
$$\pi_{sid}(Enroll) \cup \pi_{sid}(Student)$$

**Example 3:** S1 = {(22, dustin, 7, 45.0), (31, lubber, 8, 55.5), (58, rusty, 10, 35.0)}, and S2 = {(28, yuppy, 9, 35.0), (31, lubber, 8, 55.5), (44, guppy, 5, 35.0), (58, rusty, 10, 35.0)}.

$S1 \cup S2$ = {(22, dustin, 7, 45.0), (31, lubber, 8, 55.5), (58, rusty, 10, 35.0), (44, guppy, 5, 35.0), (28, yuppy, 9, 35.0)}. That is 5 tuples; lubber and rusty appear only once.

### INTERSECTION $R \cap S$ — tuples in both $R$ and $S$

**Example 1:** A = {(1, A, 2), (2, B, 4), (3, C, 6)}, B = {(1, A, 2), (4, D, 8), (5, E, 10)} over (k, x, y) → $A \cap B$ = {(1, A, 2)}.

**Example 2:**

```sql
SELECT sid FROM Student INTERSECT SELECT sid FROM Enroll;
```
$$\pi_{sid}(Student) \cap \pi_{sid}(Enroll)$$

**Example 3:** EMP_TEST = {(100, James, Troy, 232434), (104, Kathy, Holland, 324343)}, EMP_DESIGN = {(103, Rose, Freser Town, 6744545), (102, Marry, Novi, 343613), (105, Laurry, Rochester Hills, 97676), (104, Kathy, Holland, 324343)} → EMP_TEST ∩ EMP_DESIGN = {(104, Kathy, Holland, 324343)}.

### DIFFERENCE $R - S$ — tuples in $R$ but not in $S$

**Example 1:** Course_1 − Course_2 = {(11, Foundation C), (31, JAVA)}.

**Example 2:**

```sql
SELECT sid FROM Student EXCEPT SELECT sid FROM Enroll;   -- students not enrolled in any course
```
$$\pi_{sid}(Student) - \pi_{sid}(Enroll)$$

**Example 3:** X1 = {(Anoop, 22), (Saurav, 22), (Rakesh, 20), (Pritesh, 19)}, X2 = {(Anoop, 22), (Anurag, 23), (Ganesh, 21), (Saurav, 22), (Rakesh, 20)}.
- $X1 - X2$ = {(Pritesh, 19)}
- $X2 - X1$ = {(Anurag, 23), (Ganesh, 21)}

This shows that **difference is not commutative**.

### STUDENT / INSTRUCTOR (Elmasri Fig. 7.11)

STUDENT(FN, LN) = {Susan Yao, Ramesh Shah, Johnny Kohler, Barbara Jones, Amy Ford, Jimmy Wang, Ernest Gilbert}
INSTRUCTOR(FNAME, LNAME) = {John Smith, Ricardo Browne, Susan Yao, Francis Johnson, Ramesh Shah}

| Expression | Result |
|---|---|
| STUDENT ∪ INSTRUCTOR | Susan Yao, Ramesh Shah, Johnny Kohler, Barbara Jones, Amy Ford, Jimmy Wang, Ernest Gilbert, John Smith, Ricardo Browne, Francis Johnson (10 tuples) |
| STUDENT ∩ INSTRUCTOR | Susan Yao, Ramesh Shah |
| STUDENT − INSTRUCTOR | Johnny Kohler, Barbara Jones, Amy Ford, Jimmy Wang, Ernest Gilbert |
| INSTRUCTOR − STUDENT | John Smith, Ricardo Browne, Francis Johnson |

$\cup$ and $\cap$ are **commutative and associative**; $-$ is **neither**.

## 6.8 CARTESIAN PRODUCT ($\times$)

$R \times S$ combines **every tuple of $R$ with every tuple of $S$**:

$$\text{degree}(R \times S) = \text{degree}(R) + \text{degree}(S), \qquad |R \times S| = |R| \cdot |S|$$

Not union-compatible. It is mostly meaningful when **followed by a selection**.

**Example 1:** A(n) = {1, 2, 3}, B(c) = {x, y, z} → `SELECT * FROM A CROSS JOIN B` gives $3 \times 3 = 9$ tuples: (1,x) (1,y) (1,z) (2,x) (2,y) (2,z) (3,x) (3,y) (3,z).

**Example 2:** A1(Name, RollNo) = {(Anoop, 1), (Anurag, 2)}, A2(Name, RollNo) = {(Anoop, 1), (Anurag, 2), (Ganesh, 3)} → $A1 \times A2$ has $2 \times 3 = 6$ tuples:

| Name | RollNo | Name | RollNo |
|---|---|---|---|
| Anoop | 1 | Anoop | 1 |
| Anoop | 1 | Anurag | 2 |
| Anoop | 1 | Ganesh | 3 |
| Anurag | 2 | Anoop | 1 |
| Anurag | 2 | Anurag | 2 |
| Anurag | 2 | Ganesh | 3 |

**Example 3 (COMPANY):** the dependents of female employees.

$$\text{FEMALE\_EMPS} \leftarrow \sigma_{SEX='F'}(EMPLOYEE)$$
$$\text{EMPNAMES} \leftarrow \pi_{FNAME, LNAME, SSN}(\text{FEMALE\_EMPS})$$
$$\text{EMP\_DEPENDENTS} \leftarrow \text{EMPNAMES} \times DEPENDENT$$
$$\text{ACTUAL\_DEPENDENTS} \leftarrow \sigma_{SSN=ESSN}(\text{EMP\_DEPENDENTS})$$
$$\text{RESULT} \leftarrow \pi_{FNAME, LNAME, DEPENDENT\_NAME}(\text{ACTUAL\_DEPENDENTS})$$

## 6.9 JOIN ($\bowtie$)

A join is a **Cartesian product followed by a selection**:

$$R \bowtie_{\theta} S \;=\; \sigma_{\theta}(R \times S)$$

```
Joins
├── Inner joins: Theta join, Equi join, Natural join
└── Outer joins: Left outer ⟕, Right outer ⟖, Full outer ⟗
```

| SQL join | Returns |
|---|---|
| `(INNER) JOIN` | Records with matching values in both tables |
| `LEFT (OUTER) JOIN` | All records from the left table plus the matched records from the right |
| `RIGHT (OUTER) JOIN` | All records from the right table plus the matched records from the left |
| `FULL (OUTER) JOIN` | All records when there is a match in either table (all rows of both) |

### Theta join ($\bowtie_\theta$)

This is the general (conditional) join: $R \bowtie_\theta S$, where $\theta$ is any condition using $=, \ne, <, \le, >, \ge$.

**Example:**

| Customer | | | Order | |
|---|---|---|---|---|
| Cid | Cname | Age | Oid | Oname |
| 101 | Ajay | 20 | 101 | Pizza |
| 102 | Vijay | 19 | 101 | Noodles |
| 103 | Sita | 21 | 103 | Burger |

$$Customer \bowtie_{Customer.Cid > Order.Oid} Order$$

| Cid | Cname | Age | Oid | Oname |
|---|---|---|---|---|
| 102 | Vijay | 19 | 101 | Pizza |
| 102 | Vijay | 19 | 101 | Noodles |
| 103 | Sita | 21 | 101 | Pizza |
| 103 | Sita | 21 | 101 | Noodles |

(101 is not greater than any Oid, and 103 is not greater than 103.)

**SQL → RA:**

```sql
SELECT Student.name, Enroll.grade
FROM Student, Enroll
WHERE Student.sid = Enroll.sid;
```
$$\pi_{name,\,grade}\left(Student \bowtie_{Student.sid = Enroll.sid} Enroll\right)$$

That is, $Student \times Enroll$ followed by $\sigma$ gives $\bowtie$, and then projection.

**Join condition vs. filter condition:**

```sql
SELECT name FROM Student, Enroll
WHERE Student.sid = Enroll.sid AND grade = 'A';
```
$$\pi_{name}\left(\sigma_{grade='A'}\left(Student \bowtie_{Student.sid=Enroll.sid} Enroll\right)\right)$$

`Student.sid = Enroll.sid` is the **join** condition; `grade = 'A'` is a **filter** (selection).

### Equi join

A theta join whose condition contains **only equalities**. The result keeps **both** copies of the join attribute.

**Example 1:**

| Faculty | | | Offering | |
|---|---|---|---|---|
| FacSSN | FacName | | OfferNo | FacSSN |
| 111-11-1111 | joe | | 1111 | 111-11-1111 |
| 222-22-2222 | sue | | 2222 | 222-22-2222 |
| 333-33-3333 | sara | | 3333 | 111-11-1111 |

$$\text{RESULT} = Faculty \bowtie_{Faculty.FacSSN = Offering.FacSSN} Offering$$

```sql
SELECT * FROM Faculty, Offering WHERE Faculty.FacSSN = Offering.FacSSN;
```

| FacSSN | FacName | OfferNo | FacSSN |
|---|---|---|---|
| 111-11-1111 | joe | 1111 | 111-11-1111 |
| 222-22-2222 | sue | 2222 | 222-22-2222 |
| 111-11-1111 | joe | 3333 | 111-11-1111 |

(sara has no offering, so she does not appear.)

**Example 2:** S1(sid, sname, rating, age) = {(22, dustin, 7, 45.0), (31, lubber, 8, 55.5), (58, rusty, 10, 35.0)} and R1(sid, bid, day) = {(22, 101, 10/10/96), (58, 103, 11/12/96)}, where R1.sid is a foreign key. Then $S1 \bowtie_{S1.sid=R1.sid} R1$ = {(22, dustin, 7, 45.0, 101, 10/10/96), (58, rusty, 10, 35.0, 103, 11/12/96)}.

**Example 3 (COMPANY):** the manager of each department.

$$\text{DEPT\_MGR} \leftarrow DEPARTMENT \bowtie_{MGRSSN = SSN} EMPLOYEE$$

| DNAME | DNUMBER | MGRSSN | … | FNAME | MINIT | LNAME | SSN | … |
|---|---|---|---|---|---|---|---|---|
| Research | 5 | 333445555 | … | Franklin | T | Wong | 333445555 | … |
| Administration | 4 | 987654321 | … | Jennifer | S | Wallace | 987654321 | … |
| Headquarters | 1 | 888665555 | … | James | E | Borg | 888665555 | … |

**Example 4 (SQL `INNER JOIN … USING`):** Class(ID, NAME) = {(1, AAA), (2, BBB), (4, DDD)}, Classinfo(ID, ADDRESS) = {(1, CHENNAI), (2, MUMBAI), (3, DELHI)}.

```sql
SELECT * FROM CLASS INNER JOIN CLASSINFO USING (ID);
```

| ID | NAME | ADDRESS |
|---|---|---|
| 1 | AAA | CHENNAI |
| 2 | BBB | MUMBAI |

### Natural join ($\bowtie$, written `*` in Elmasri)

- An equi join on **all attributes with the same name**, with the duplicate columns **removed**.
- It can only be performed if there is a common attribute, and the **name and type of that attribute must be the same**.

**Example 1:** `SELECT * FROM CLASS NATURAL JOIN CLASSINFO;` gives the same 2 rows as above: (1, AAA, CHENNAI) and (2, BBB, MUMBAI).

**Example 2:**

| Employee | | | Dept | |
|---|---|---|---|---|
| Name | EmpId | DeptName | DeptName | Manager |
| Harry | 3415 | Finance | Finance | George |
| Sally | 2241 | Sales | Sales | Harriet |
| George | 3401 | Finance | Production | Charles |
| Harriet | 2202 | Sales | | |

$Employee \bowtie Dept$:

| Name | EmpId | DeptName | Manager |
|---|---|---|---|
| Harry | 3415 | Finance | George |
| Sally | 2241 | Sales | Harriet |
| George | 3401 | Finance | George |
| Harriet | 2202 | Sales | Harriet |

(Production has no employee and does not appear.)

**Example 3:** $r(A,B,C,D)$ and $s(B,D,E)$, joined on the **common attributes B and D**.

| r: A | B | C | D |
|---|---|---|---|
| α | 1 | α | a |
| β | 2 | γ | a |
| γ | 4 | β | b |
| α | 1 | γ | a |
| δ | 2 | β | b |

| s: B | D | E |
|---|---|---|
| 1 | a | α |
| 3 | a | β |
| 1 | a | γ |
| 2 | b | δ |
| 3 | b | ε |

$r \bowtie s$:

| A | B | C | D | E |
|---|---|---|---|---|
| α | 1 | α | a | α |
| α | 1 | α | a | γ |
| α | 1 | γ | a | α |
| α | 1 | γ | a | γ |
| δ | 2 | β | b | δ |

Equivalently, $r \bowtie s = \pi_{r.A, r.B, r.C, r.D, s.E}\left(\sigma_{r.B = s.B \,\wedge\, r.D = s.D}(r \times s)\right)$.

**Example 4 (COMPANY):**
- $\text{PROJ\_DEPT} \leftarrow PROJECT \bowtie \rho_{(DNAME, DNUM, MGRSSN, MGRSTARTDATE)}(DEPARTMENT)$ joins on DNUM.
- $\text{DEPT\_LOCS} \leftarrow DEPARTMENT \bowtie DEPT\_LOCATIONS$ joins on DNUMBER.

PROJ_DEPT:

| PNAME | PNUMBER | PLOCATION | DNUM | DNAME | MGRSSN | MGRSTARTDATE |
|---|---|---|---|---|---|---|
| ProductX | 1 | Bellaire | 5 | Research | 333445555 | 1988-05-22 |
| ProductY | 2 | Sugarland | 5 | Research | 333445555 | 1988-05-22 |
| ProductZ | 3 | Houston | 5 | Research | 333445555 | 1988-05-22 |
| Computerization | 10 | Stafford | 4 | Administration | 987654321 | 1995-01-01 |
| Reorganization | 20 | Houston | 1 | Headquarters | 888665555 | 1981-06-19 |
| Newbenefits | 30 | Stafford | 4 | Administration | 987654321 | 1995-01-01 |

DEPT_LOCS:

| DNAME | DNUMBER | MGRSSN | MGRSTARTDATE | LOCATION |
|---|---|---|---|---|
| Headquarters | 1 | 888665555 | 1981-06-19 | Houston |
| Administration | 4 | 987654321 | 1995-01-01 | Stafford |
| Research | 5 | 333445555 | 1988-05-22 | Bellaire |
| Research | 5 | 333445555 | 1988-05-22 | Sugarland |
| Research | 5 | 333445555 | 1988-05-22 | Houston |

### Outer joins

An outer join keeps the matching tuples **plus the tuples that don't match**, padding the missing attributes with **NULL**.

| Operator | Keeps all tuples of | Symbol |
|---|---|---|
| Left outer join | Left relation | $R$ ⟕ $S$ |
| Right outer join | Right relation | $R$ ⟖ $S$ |
| Full outer join | Both relations | $R$ ⟗ $S$ |

**Class / Classinfo examples:**

```sql
SELECT * FROM CLASS NATURAL LEFT OUTER JOIN CLASSINFO;
SELECT * FROM CLASS NATURAL RIGHT OUTER JOIN CLASSINFO;
SELECT * FROM CLASS NATURAL FULL OUTER JOIN CLASSINFO;
```

| LEFT: ID | NAME | ADDRESS | | RIGHT: ID | NAME | ADDRESS | | FULL: ID | NAME | ADDRESS |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | AAA | CHENNAI | | 1 | AAA | CHENNAI | | 1 | AAA | CHENNAI |
| 2 | BBB | MUMBAI | | 2 | BBB | MUMBAI | | 2 | BBB | MUMBAI |
| 4 | DDD | NULL | | 3 | NULL | DELHI | | 3 | NULL | DELHI |
| | | | | | | | | 4 | DDD | NULL |

**DEPT / EMPLOYEE examples:** DEPT = {(10, Account), (20, Design), (30, Testing)}, EMPLOYEE = {(100, James, 10), (101, Kathy, 20), (103, Rose, 10), (104, Marry, 20)}.

DEPT ⟕ EMPLOYEE (left outer):

| DEPT_ID | DEPT_NAME | EMP_ID | ENAME | DEPT_ID |
|---|---|---|---|---|
| 10 | Account | 100 | James | 10 |
| 10 | Account | 103 | Rose | 10 |
| 20 | Design | 101 | Kathy | 20 |
| 20 | Design | 104 | Marry | 20 |
| 30 | Testing | NULL | NULL | NULL |

EMPLOYEE ⟖ DEPT (right outer):

| EMP_ID | ENAME | DEPT_ID | DEPT_ID | DEPT_NAME |
|---|---|---|---|---|
| 100 | James | 10 | 10 | Account |
| 101 | Kathy | 20 | 20 | Design |
| 103 | Rose | 10 | 10 | Account |
| 104 | Marry | 20 | 20 | Design |
| NULL | NULL | NULL | 30 | Testing |

EMPLOYEE ⟗ DEPT (full outer, with an extra employee (105, Alex, 40)):

| EMP_ID | ENAME | DEPT_ID | DEPT_ID | DEPT_NAME |
|---|---|---|---|---|
| 100 | James | 10 | 10 | Account |
| 101 | Kathy | 20 | 20 | Design |
| 103 | Rose | 10 | 10 | Account |
| 104 | Marry | 20 | 20 | Design |
| 105 | Alex | 40 | NULL | NULL |
| NULL | NULL | NULL | 30 | Testing |

**Products example (left join):** TABLE1 = {(Kiwis, $6), (Onions, $3), (Tomatoes, $7)}, TABLE2 = {(Kiwis, 10), (Onions, 6), (Broccoli, 5)}. The LEFT JOIN on Products gives {(Kiwis, $6, 10), (Onions, $3, 6), (Tomatoes, $7, NULL)}.

**Right join example:** table112(id, bval1, bval2) = {(701, 405, 16), (704, 409, 14), (706, 403, 13), (709, 401, 12)} and table111(id, aval1) = {(1, 405), (2, 401), (3, 200), (4, 400)}. table112 RIGHT JOIN table111 ON bval1 = aval1 gives:

| id | bval1 | bval2 | id | aval1 |
|---|---|---|---|---|
| 701 | 405 | 16 | 1 | 405 |
| 709 | 401 | 12 | 2 | 401 |
| NULL | NULL | NULL | 3 | 200 |
| NULL | NULL | NULL | 4 | 400 |

**Customers / Orders (full outer join on CustomerId):** Customers = {(1, Shree), (2, Kalpana), (3, Basavaraj)}, Orders = {(100, 1, 2014-01-29), (200, 4, 2014-01-30), (300, 3, 2014-01-31)} →

| CustomerId | Name | OrderId | CustomerId | OrderDate |
|---|---|---|---|---|
| 1 | Shree | 100 | 1 | 2014-01-29 |
| 2 | Kalpana | NULL | NULL | NULL |
| 3 | Basavaraj | 300 | 3 | 2014-01-31 |
| NULL | NULL | 200 | 4 | 2014-01-30 |

**PARTS / PRODUCTS (all three at once):** PARTS(PART, PROD#) = {WIRE 10, MAGNETS 10, BLADES 205, PLASTIC 30, OIL 160}, PRODUCTS(PROD#, PRICE) = {505 3.70, 10 45.75, 205 18.90, 30 7.55}.

| | Left outer join | Right outer join | Full outer join |
|---|---|---|---|
| WIRE 10 45.75 | ✓ | ✓ | ✓ |
| MAGNETS 10 45.75 | ✓ | ✓ | ✓ |
| BLADES 205 18.90 | ✓ | ✓ | ✓ |
| PLASTIC 30 7.55 | ✓ | ✓ | ✓ |
| OIL 160 NULL (unmatched part) | ✓ | – | ✓ |
| NULL 505 3.70 (unmatched product) | – | ✓ | ✓ |

## 6.10 DIVISION ($\div$)

- Used for queries containing **"all"**, **"for all"**, **"in all"** or **"every"**. For example: *Which person has an account in **all** the banks of a particular city? Which students have taken **all** the courses required to graduate?*
- For $R(A, B) \div S(B)$: the result contains the **values of $A$ that are paired in $R$ with every value of $B$ in $S$**.

$$R \div S = \pi_A(R) - \pi_A\big((\pi_A(R) \times S) - R\big)$$

(i.e. all candidate $A$ values, minus those that are missing at least one $B$ from $S$.)

**Example (slide):** A(x, y) = {(x1,y1), (x1,y2), (x1,y3), (x1,y4), (x2,y1), (x2,y2), (x3,y2), (x4,y2), (x4,y4)}

| Divisor | B | A ÷ B |
|---|---|---|
| B1 | {y2} | {x1, x2, x3, x4} (everyone has y2) |
| B2 | {y2, y4} | {x1, x4} |
| B3 | {y1, y2, y4} | {x1} |

**Example – "students who enrolled in ALL courses":**

$$\pi_{sid,\,cid}(Enroll) \div \pi_{cid}(Course)$$

```sql
SELECT sid FROM Enroll
GROUP BY sid
HAVING COUNT(DISTINCT cid) = (SELECT COUNT(*) FROM Course);
```

## 6.11 Aggregate functions and grouping (extended RA)

- Common functions on collections of values: **SUM, AVERAGE, MAXIMUM, MINIMUM**, and **COUNT** (counts tuples or values).
- Classical RA does **not** support aggregation. This is **extended RA**.

| Notation | Meaning |
|---|---|
| $\mathcal{F}_{MAX\ Salary}(Employee)$ | Maximum salary |
| $\mathcal{F}_{MIN\ Salary}(Employee)$ | Minimum salary |
| $\mathcal{F}_{SUM\ Salary}(Employee)$ | Sum of salaries |
| ${}_{DNO}\mathcal{F}_{COUNT\ SSN,\ AVERAGE\ SALARY}(EMPLOYEE)$ | Per department: number of employees and average salary (grouping attributes go on the left) |
| $\gamma_{count(*)}(Student)$ | `SELECT COUNT(*) FROM Student;` |
| $\gamma_{dept,\ count(*)}(Student)$ | `SELECT dept, COUNT(*) FROM Student GROUP BY dept;` |

In $\gamma$ notation, $\gamma$ is the grouping operator: the first attribute(s) are the grouping columns and the rest are aggregate functions.

Slide SQL example:

```sql
SELECT country.country_name,
       COUNT(city.lat) AS lat_count, SUM(city.lat) AS lat_sum, AVG(city.lat) AS lat_avg,
       MIN(city.lat)   AS lat_min,   MAX(city.lat) AS lat_max
FROM city INNER JOIN country ON city.country_id = country.id
GROUP BY country.id, country.country_name;
```

| country_name | lat_count | lat_sum | lat_avg | lat_min | lat_max |
|---|---|---|---|---|---|
| Deutschland | 1 | 52.520008 | 52.520008 | 52.520008 | 52.520008 |
| Hrvatska | 1 | 45.815399 | 45.815399 | 45.815399 | 45.815399 |
| Polska | 1 | 52.237049 | 52.237049 | 52.237049 | 52.237049 |
| Srbija | 1 | 44.787197 | 44.787197 | 44.787197 | 44.787197 |
| United States of America | 2 | 74.782845 | 37.391422 | 34.052235 | 40.730610 |

## 6.12 Summary of operators

| Operation | Symbol | Purpose | SQL equivalent |
|---|---|---|---|
| SELECT | $\sigma$ (sigma) | Filter rows | `WHERE` |
| PROJECT | $\pi$ (pi) | Choose columns (removes duplicates) | `SELECT DISTINCT col…` |
| RENAME | $\rho$ (rho) | Rename relation/attributes | `AS` |
| UNION | $\cup$ (cup) | Tuples in either | `UNION` |
| INTERSECTION | $\cap$ (cap) | Tuples in both | `INTERSECT` |
| DIFFERENCE | $-$ (minus) | Tuples in first, not second | `EXCEPT` / `MINUS` |
| CARTESIAN PRODUCT | $\times$ (times) | All combinations | `CROSS JOIN` / `FROM R, S` |
| JOIN | $\bowtie$ (bow-tie) | Product + selection | `JOIN … ON` |
| DIVISION | $\div$ | "For all" queries | `NOT EXISTS … NOT EXISTS` / `GROUP BY … HAVING COUNT` |
| AGGREGATE | $\mathcal{F}$ / $\gamma$ | Aggregates and grouping | `GROUP BY`, `COUNT`, `SUM` … |

---

# 7. Translating SQL Queries into Relational Algebra

## 7.1 Query blocks

- A **query block** is the basic unit that is translated into RA operators and optimised.
- A query block contains a **single SELECT-FROM-WHERE** expression, plus GROUP BY and HAVING if they are part of it.
- **Nested queries** are identified as **separate query blocks**.
- SQL aggregate operators (MAX, MIN, SUM, COUNT, AVG) must be included in the **extended algebra**.

**Basic mapping:**

$$\texttt{SELECT } A_1,\dots,A_n \texttt{ FROM } R_1,\dots,R_m \texttt{ WHERE } P \;\;\Longrightarrow\;\; \pi_{A_1,\dots,A_n}\left(\sigma_P(R_1 \times R_2 \times \dots \times R_m)\right)$$

| SQL clause | RA |
|---|---|
| `FROM R1, R2` | $R_1 \times R_2$ (or $\bowtie$ once the join condition is attached) |
| `WHERE` | $\sigma$ |
| `SELECT` list | $\pi$ |
| `GROUP BY` / aggregates | $\mathcal{F}$ / $\gamma$ |
| `UNION` / `INTERSECT` / `EXCEPT` | $\cup$ / $\cap$ / $-$ |
| `AS` | $\rho$ |

## 7.2 Example: nested query → two blocks

```sql
SELECT LNAME, FNAME
FROM   EMPLOYEE
WHERE  SALARY > ( SELECT MAX(SALARY)
                  FROM   EMPLOYEE
                  WHERE  DNO = 5 );
```

| Block | SQL | RA |
|---|---|---|
| Inner | `SELECT MAX(SALARY) FROM EMPLOYEE WHERE DNO = 5` | $\mathcal{F}_{MAX\ SALARY}\left(\sigma_{DNO=5}(EMPLOYEE)\right)$ → a constant $C$ |
| Outer | `SELECT LNAME, FNAME FROM EMPLOYEE WHERE SALARY > C` | $\pi_{LNAME, FNAME}\left(\sigma_{SALARY > C}(EMPLOYEE)\right)$ |

The inner block is evaluated once, and its result $C$ is used by the outer block. This is an *uncorrelated* nested query.

## 7.3 Example: movie stars

```sql
SELECT movieTitle
FROM   StarsIn, MovieStar
WHERE  starName = name AND birthdate = 1960;
```
$$\pi_{movieTitle}\left(\sigma_{starName = name \,\wedge\, birthdate = 1960}(StarsIn \times MovieStar)\right)$$

## 7.4 Example: Stafford projects (Q2)

*For every project located in 'Stafford', retrieve the project number, the controlling department number, and the department manager's last name, address and birthdate.*

```sql
SELECT P.PNUMBER, P.DNUM, E.LNAME, E.ADDRESS, E.BDATE
FROM   PROJECT AS P, DEPARTMENT AS D, EMPLOYEE AS E
WHERE  P.DNUM = D.DNUMBER AND D.MGRSSN = E.SSN AND P.PLOCATION = 'Stafford';
```

$$\pi_{PNUMBER, DNUM, LNAME, ADDRESS, BDATE}\Big(\big(\left(\sigma_{PLOCATION='Stafford'}(PROJECT)\right) \bowtie_{DNUM=DNUMBER} DEPARTMENT\big) \bowtie_{MGRSSN=SSN} EMPLOYEE\Big)$$

## 7.5 Sub-queries

**IN → join (semi-join):**

```sql
SELECT name FROM Student WHERE sid IN (SELECT sid FROM Enroll);
```

- Step 1 (subquery): $\pi_{sid}(Enroll)$
- Step 2 (equivalent join): $\pi_{name}(Student \bowtie Enroll)$, or as a semi-join, $\pi_{name}(Student \ltimes Enroll)$

`IN` is usually converted into a **semi-join**.

**NOT IN → difference:**

```sql
SELECT name FROM Student WHERE sid NOT IN (SELECT sid FROM Enroll);
```

Convert `NOT IN` into **set difference**. Both operands must be union-compatible, so take the difference on `sid` and join back:

$$\pi_{name}\Big(Student \bowtie \big(\pi_{sid}(Student) - \pi_{sid}(Enroll)\big)\Big)$$

(The slide writes $\pi_{name}(Student - (Student \bowtie Enroll))$. The idea is right, but $Student$ and $Student \bowtie Enroll$ have different schemas. See Errata.)

## 7.6 More SQL → RA practice (Supplementary, COMPANY schema)

| Query | Relational algebra |
|---|---|
| Names and addresses of employees in the 'Research' department | $\pi_{FNAME, LNAME, ADDRESS}\left(\sigma_{DNAME='Research'}(DEPARTMENT) \bowtie_{DNUMBER=DNO} EMPLOYEE\right)$ |
| SSNs of employees who work in dept 5 **or** supervise someone in dept 5 | $\pi_{SSN}(\sigma_{DNO=5}(EMPLOYEE)) \cup \pi_{SUPERSSN}(\sigma_{DNO=5}(EMPLOYEE))$ |
| Employees with no dependents | $\pi_{LNAME, FNAME}\Big(EMPLOYEE \bowtie \big(\pi_{SSN}(EMPLOYEE) - \rho_{(SSN)}(\pi_{ESSN}(DEPENDENT))\big)\Big)$ |
| Employees who work on **all** projects controlled by dept 5 | $\rho_{(SSN,PNO)}(\pi_{ESSN,PNO}(WORKS\_ON)) \div \rho_{(PNO)}(\pi_{PNUMBER}(\sigma_{DNUM=5}(PROJECT)))$, then join with EMPLOYEE for names |

---

# 8. Tuple Relational Calculus (Supplementary)

> On the syllabus, but not covered by the Module-3 slides. These notes follow Elmasri & Navathe, Ch. 8.

## 8.1 Idea

- **Relational calculus** is a **declarative (non-procedural)** formal query language: you describe **what** you want, not how to get it. SQL is based on it.
- **Tuple relational calculus (TRC)** variables range over **tuples** of a relation. (Domain relational calculus variables range over attribute values.)
- A TRC query has the form

$$\{\, t \mid \text{COND}(t) \,\}$$

where $t$ is a **tuple variable** and $\text{COND}(t)$ is a formula. The result is the set of all tuples $t$ that make COND true.

## 8.2 Formulas

Atoms:
1. $R(t)$ – $t$ is a tuple of relation $R$ (the **range relation**)
2. $t_i.A \;\text{op}\; t_j.B$ with op $\in \{=, <, \le, >, \ge, \ne\}$
3. $t_i.A \;\text{op}\; c$, where $c$ is a constant

Formulas are built with $\wedge$ (AND), $\vee$ (OR), $\neg$ (NOT) and the quantifiers:
- **Existential** $(\exists t)(F)$ – true if **some** tuple $t$ makes $F$ true.
- **Universal** $(\forall t)(F)$ – true if **every** tuple $t$ makes $F$ true.

A tuple variable is **bound** if it is quantified and **free** otherwise. Only the free variables may appear to the left of the bar $\mid$.

Useful equivalences:

$$(\forall x)(P(x)) \equiv \neg(\exists x)(\neg P(x)), \qquad (\exists x)(P(x)) \equiv \neg(\forall x)(\neg P(x)), \qquad P \Rightarrow Q \equiv \neg P \vee Q$$

## 8.3 Examples

| Query | TRC | RA equivalent |
|---|---|---|
| Employees earning more than 50000 | $\{\, t \mid EMPLOYEE(t) \wedge t.SALARY > 50000 \,\}$ | $\sigma_{SALARY>50000}(EMPLOYEE)$ |
| Only their first and last names | $\{\, t.FNAME, t.LNAME \mid EMPLOYEE(t) \wedge t.SALARY > 50000 \,\}$ | $\pi_{FNAME,LNAME}(\sigma_{SALARY>50000}(EMPLOYEE))$ |
| Name and address of employees in 'Research' | $\{\, t.FNAME, t.LNAME, t.ADDRESS \mid EMPLOYEE(t) \wedge (\exists d)(DEPARTMENT(d) \wedge d.DNAME='Research' \wedge d.DNUMBER = t.DNO) \,\}$ | $\pi(\sigma_{DNAME='Research'}(DEPARTMENT) \bowtie_{DNUMBER=DNO} EMPLOYEE)$ |
| Employees with no dependents | $\{\, e.FNAME, e.LNAME \mid EMPLOYEE(e) \wedge \neg(\exists d)(DEPENDENT(d) \wedge e.SSN = d.ESSN) \,\}$ | uses $-$ |
| Employees who work on **every** project of dept 5 | $\{\, e.LNAME \mid EMPLOYEE(e) \wedge (\forall x)\big(\neg PROJECT(x) \vee x.DNUM \ne 5 \vee (\exists w)(WORKS\_ON(w) \wedge w.ESSN = e.SSN \wedge w.PNO = x.PNUMBER)\big) \,\}$ | uses $\div$ |

## 8.4 Safe expressions and expressive power

- A TRC expression is **safe** if every value in its result comes from the **domain of the expression** (values that appear in the relations or constants it mentions). For example, $\{ t \mid \neg EMPLOYEE(t) \}$ is **unsafe**, because it would return infinitely many tuples.
- **Codd's theorem:** safe TRC, safe DRC and basic relational algebra have the **same expressive power**. A language that can express every RA query is called **relationally complete**.

---

# 9. Query Processing and Query Optimization

## 9.1 Basic steps in query processing

```
 query ──► [Parser & Translator] ──► relational-algebra expression ──► [Optimizer] ──► execution plan ──► [Evaluation engine] ──► query output
                                                                          ▲                                      ▲
                                                               statistics about data                            data
```

1. **Parsing and translation** – check syntax, verify relation/attribute names, and translate into an internal form (a relational algebra expression, i.e. a query tree).
2. **Optimisation** – choose a suitable **execution strategy** (plan) among the many equivalent ones.
3. **Evaluation** – the query-execution engine runs the plan and returns the answer.

**Query optimisation** is the process of choosing a suitable execution strategy for processing a query. It rarely finds the absolute best plan; the goal is a *reasonably efficient* one.

**Process for heuristic optimisation:**
1. The parser generates an **initial internal representation** (a canonical query tree).
2. **Heuristic rules** are applied to optimise it.
3. A **query execution plan** is generated to execute groups of operations, based on the access paths (indexes, hashing, sort order) available on the files.

> **Main heuristic:** apply first the operations that **reduce the size of intermediate results**, i.e. apply **SELECT and PROJECT before JOIN** and other binary operations.

## 9.2 Query tree and query graph

**Query tree:**
- A tree data structure corresponding to a **relational algebra expression**.
- **Leaf nodes = input relations**; **internal nodes = RA operations**.
- Execution: run an internal node's operation whenever its operands are available, replace the node by the resulting relation, and repeat up to the root.
- It fixes an **order** of execution.

**Query graph:**
- A graph data structure corresponding to a **relational calculus expression**.
- **Relation nodes** are single circles, **constant nodes** are double circles, and edges are **selection or join conditions**. The attributes to retrieve are shown in square brackets.
- It does **not** indicate an order of operations. There is **only one graph per query**, but many trees.

### Q2 (Stafford query) as trees and a graph

**(a) Query tree for the RA expression of Q2** (the numbers give the execution order):

```
                 π P.PNUMBER, P.DNUM, E.LNAME, E.ADDRESS, E.BDATE
                                  │
                   (3)  ⋈ D.MGRSSN = E.SSN
                        ┌─────────┴─────────┐
          (2)  ⋈ P.DNUM = D.DNUMBER          E
               ┌────────┴────────┐
 (1) σ P.PLOCATION='Stafford'     D
               │
               P
```

**(b) Initial (canonical) query tree for SQL Q2**, a direct translation of the SQL that is very inefficient:

```
                 π P.PNUMBER, P.DNUM, E.LNAME, E.ADDRESS, E.BDATE
                                  │
        σ P.DNUM=D.DNUMBER AND D.MGRSSN=E.SSN AND P.PLOCATION='Stafford'
                                  │
                                  ×
                        ┌─────────┴─────────┐
                        ×                   E
                   ┌────┴────┐
                   P         D
```

**(c) Query graph for Q2:**

```
   [P.PNUMBER, P.DNUM]                          [E.LNAME, E.ADDRESS, E.BDATE]
          (P) ───── P.DNUM = D.DNUMBER ───── (D) ───── D.MGRSSN = E.SSN ───── (E)
           │
   P.PLOCATION = 'Stafford'
           │
     (('Stafford'))
```

## 9.3 Heuristic optimisation of query trees

The same query corresponds to many RA expressions, and so to many query trees. **Heuristic optimisation** transforms the initial canonical tree into an equivalent **final query tree that is efficient to execute**.

### Steps

1. **Move SELECT operations down** the query tree, as close to the leaves as possible.
2. **Apply the more restrictive SELECT first**, i.e. rearrange the leaves so the relation with the most selective condition is processed first.
3. **Replace CARTESIAN PRODUCT + SELECT with JOIN.**
4. **Move PROJECT operations down** the tree, keeping only the attributes needed later.

### Worked example: the "Aquarius" query

```sql
SELECT LNAME
FROM   EMPLOYEE, WORKS_ON, PROJECT
WHERE  PNAME = 'Aquarius' AND PNUMBER = PNO AND ESSN = SSN AND BDATE > '1957-12-31';
```

**(a) Initial canonical tree:**

```
π LNAME
 └─ σ PNAME='Aquarius' AND PNUMBER=PNO AND ESSN=SSN AND BDATE>'1957-12-31'
     └─ ×
        ├─ ×
        │  ├─ EMPLOYEE
        │  └─ WORKS_ON
        └─ PROJECT
```

(The product of all three relations is huge.)

**(b) Move SELECT operations down.** Break the conjunctive σ into single conditions (cascade of σ) and push each as far down as its attributes allow:

```
π LNAME
 └─ σ PNUMBER=PNO
     └─ ×
        ├─ σ ESSN=SSN
        │   └─ ×
        │      ├─ σ BDATE>'1957-12-31'
        │      │   └─ EMPLOYEE
        │      └─ WORKS_ON
        └─ σ PNAME='Aquarius'
            └─ PROJECT
```

**(c) Apply the more restrictive SELECT first.** `PNAME = 'Aquarius'` matches only **one** project, so swap the leaves and process PROJECT first:

```
π LNAME
 └─ σ ESSN=SSN
     └─ ×
        ├─ σ PNUMBER=PNO
        │   └─ ×
        │      ├─ σ PNAME='Aquarius'
        │      │   └─ PROJECT
        │      └─ WORKS_ON
        └─ σ BDATE>'1957-12-31'
            └─ EMPLOYEE
```

**(d) Replace × + σ with JOIN:**

```
π LNAME
 └─ ⋈ ESSN=SSN
    ├─ ⋈ PNUMBER=PNO
    │  ├─ σ PNAME='Aquarius'
    │  │   └─ PROJECT
    │  └─ WORKS_ON
    └─ σ BDATE>'1957-12-31'
        └─ EMPLOYEE
```

**(e) Move PROJECT operations down.** Keep only the attributes needed by later operations:

```
π LNAME
 └─ ⋈ ESSN=SSN
    ├─ π ESSN
    │   └─ ⋈ PNUMBER=PNO
    │      ├─ π PNUMBER
    │      │   └─ σ PNAME='Aquarius'
    │      │       └─ PROJECT
    │      └─ π ESSN, PNO
    │          └─ WORKS_ON
    └─ π SSN, LNAME
        └─ σ BDATE>'1957-12-31'
            └─ EMPLOYEE
```

## 9.4 Transformation (equivalence) rules

$E, E_1, E_2, E_3$ are RA expressions, $\theta$ are conditions, and $L$ are attribute lists.

1. **Cascade of σ:** a conjunctive selection can be broken into a sequence of individual selections.
$$\sigma_{\theta_1 \wedge \theta_2}(E) = \sigma_{\theta_1}\left(\sigma_{\theta_2}(E)\right)$$

2. **Commutativity of σ:**
$$\sigma_{\theta_1}\left(\sigma_{\theta_2}(E)\right) = \sigma_{\theta_2}\left(\sigma_{\theta_1}(E)\right)$$

3. **Cascade of π:** only the last (outermost) projection in a sequence is needed.
$$\pi_{L_1}\left(\pi_{L_2}\left(\dots\left(\pi_{L_n}(E)\right)\dots\right)\right) = \pi_{L_1}(E) \qquad (L_1 \subseteq L_2 \subseteq \dots \subseteq L_n)$$

4. **Selections combine with Cartesian products and theta joins:**
$$\sigma_{\theta}(E_1 \times E_2) = E_1 \bowtie_{\theta} E_2$$
$$\sigma_{\theta_1}(E_1 \bowtie_{\theta_2} E_2) = E_1 \bowtie_{\theta_1 \wedge \theta_2} E_2$$

5. **Theta joins (and natural joins, and ×) are commutative:**
$$E_1 \bowtie_{\theta} E_2 = E_2 \bowtie_{\theta} E_1$$

6. **Joins are associative:**
$$(E_1 \bowtie E_2) \bowtie E_3 = E_1 \bowtie (E_2 \bowtie E_3)$$
$$(E_1 \bowtie_{\theta_1} E_2) \bowtie_{\theta_2 \wedge \theta_3} E_3 = E_1 \bowtie_{\theta_1 \wedge \theta_3} (E_2 \bowtie_{\theta_2} E_3)$$
where $\theta_2$ involves attributes from $E_2$ and $E_3$ only.

7. **σ distributes over theta join:**
   - (a) if $\theta_0$ involves only attributes of $E_1$:
   $$\sigma_{\theta_0}(E_1 \bowtie_{\theta} E_2) = \left(\sigma_{\theta_0}(E_1)\right) \bowtie_{\theta} E_2$$
   - (b) if $\theta_1$ involves only $E_1$'s attributes and $\theta_2$ only $E_2$'s:
   $$\sigma_{\theta_1 \wedge \theta_2}(E_1 \bowtie_{\theta} E_2) = \left(\sigma_{\theta_1}(E_1)\right) \bowtie_{\theta} \left(\sigma_{\theta_2}(E_2)\right)$$

8. **π distributes over theta join:**
   - (a) if the join condition $\theta$ involves only attributes in $L_1 \cup L_2$ ($L_1$ from $E_1$, $L_2$ from $E_2$):
   $$\pi_{L_1 \cup L_2}(E_1 \bowtie_{\theta} E_2) = \left(\pi_{L_1}(E_1)\right) \bowtie_{\theta} \left(\pi_{L_2}(E_2)\right)$$
   - (b) in general, with $L_3$ = attributes of $E_1$ used in $\theta$ but not in $L_1 \cup L_2$, and $L_4$ = attributes of $E_2$ used in $\theta$ but not in $L_1 \cup L_2$:
   $$\pi_{L_1 \cup L_2}(E_1 \bowtie_{\theta} E_2) = \pi_{L_1 \cup L_2}\left(\left(\pi_{L_1 \cup L_3}(E_1)\right) \bowtie_{\theta} \left(\pi_{L_2 \cup L_4}(E_2)\right)\right)$$

9. **∪ and ∩ are commutative** (set difference is **not**):
$$E_1 \cup E_2 = E_2 \cup E_1, \qquad E_1 \cap E_2 = E_2 \cap E_1$$

10. **∪ and ∩ are associative:**
$$(E_1 \cup E_2) \cup E_3 = E_1 \cup (E_2 \cup E_3), \qquad (E_1 \cap E_2) \cap E_3 = E_1 \cap (E_2 \cap E_3)$$

11. **σ distributes over ∪, ∩ and −:**
$$\sigma_P(E_1 - E_2) = \sigma_P(E_1) - \sigma_P(E_2) = \sigma_P(E_1) - E_2$$
(and similarly $\sigma_P(E_1 \cup E_2) = \sigma_P(E_1) \cup \sigma_P(E_2)$, $\sigma_P(E_1 \cap E_2) = \sigma_P(E_1) \cap \sigma_P(E_2)$)

12. **π distributes over ∪:**
$$\pi_L(E_1 \cup E_2) = \left(\pi_L(E_1)\right) \cup \left(\pi_L(E_2)\right)$$

## 9.5 Outline of the heuristic optimisation algorithm (Elmasri)

1. Use **rule 1** (cascade of σ) to break up SELECTs with conjunctive conditions.
2. Use **rules 2, 7 and 11** (commutativity of σ with other operations) to move each SELECT **as far down the tree** as its attributes allow.
3. Use **rules 5, 6 and 10** (commutativity/associativity of binary operations) to rearrange the leaves so that the **most restrictive SELECTs are executed first** and no Cartesian products are created unnecessarily.
4. Use **rule 4** to combine a Cartesian product with a following SELECT into a **JOIN**.
5. Use **rules 3, 8 and 12** (cascade of π and π distributing over other operations) to break down and **move PROJECTs down** the tree, creating new ones as needed.
6. Identify subtrees that can be executed by a **single algorithm** (e.g. pipelining a σ into a ⋈).

> **Summary of heuristics:** push σ down → most selective σ first → × + σ ⇒ ⋈ → push π down.

---

# 10. Quick Revision Sheet

### Formulas

| Concept | Formula |
|---|---|
| Blocking factor | $bfr = \lfloor B / R \rfloor$ |
| Unused space per block | $B - bfr \cdot R$ |
| Blocks for $r$ records | $b = \lceil r / bfr \rceil$ |
| Disk access time | seek time + rotational latency (+ transfer) |
| Linear search (unordered) | avg $b/2$, worst $b$ block accesses |
| Binary search (ordered) | $\lceil \log_2 b \rceil$ |
| Multilevel index levels | $t = \lceil \log_{fo} r_1 \rceil$; search cost $t + 1$ |
| Linear probing | $(h(k) + i) \bmod m$ |
| Quadratic probing | $(h(k) + i^2) \bmod m$ |
| Double hashing | $(h_1(k) + i \cdot h_2(k)) \bmod m$, with $h_2(k) \ne 0$ |
| Extendible hashing | directory size $2^{GD}$; entries per bucket $2^{GD-LD}$; double only if $LD = GD$ |
| B+ tree (order $m$) | max keys $m-1$; internal min children $\lceil m/2 \rceil$; internal min keys $\lceil m/2 \rceil - 1$; leaf min keys $\lceil (m-1)/2 \rceil$ |
| Join | $R \bowtie_\theta S = \sigma_\theta(R \times S)$ |
| Intersection | $R \cap S = R - (R - S)$ |
| Division | $R \div S = \pi_A(R) - \pi_A((\pi_A(R) \times S) - R)$ |
| Product sizes | degree $n + m$, cardinality $\lvert R \rvert \cdot \lvert S \rvert$ |

### One-liners
- **Open hashing = separate chaining** (closed addressing); **closed hashing = open addressing** (probing).
- Linear probing → **primary** clustering. Quadratic → **secondary** clustering. Double hashing → neither.
- Hash indexes are good for **equality**, bad for **range** queries. B+ trees are good for both.
- **Dense = every** key; **sparse = some** (one per block, file must be sorted). **Secondary indexes must be dense.**
- **Primary** = ordering key field. **Clustering** = ordering non-key field. **Secondary** = non-ordering field.
- Only **one clustered index** per table.
- **B+ tree:** all data pointers in the leaves, leaves linked, balanced. Leaf split **copies** up, internal split **pushes** up.
- **LSM:** WAL → MemTable → Immutable MemTable → SSTable flush → compaction. Delete = **tombstone**, update = **new version**. Bloom filter = "definitely not" or "maybe".
- STCS = write-optimised; LCS = read-optimised; Universal = time-series; FIFO = logs/caches.
- σ filters **rows**, π filters **columns** and removes duplicates.
- ∪, ∩, − need **union compatibility**.
- "All/every" queries → **division**.
- Query **tree** = RA (ordered, many per query); query **graph** = relational calculus (unordered, one per query).
- Heuristics: **push σ down, most restrictive σ first, × + σ → ⋈, push π down.**

---

# 11. Errata – Mistakes Found in the Slides

Slide numbers refer to `BACSE202-DBS-Module-3 LSM trees.ppt` unless marked *idx* (`Types_of_Indexes_in_DBMS.pptx`).

| Slide | What the slide says | Correct version |
|---|---|---|
| 5 | Double-buffer figure shows "Fill A" for blocks i+3 and i+4 | Buffers alternate: i+3 → Fill **B**, i+4 → Fill A |
| 17 | $H(x) \bmod 5$ table shows slots 3 and 4 empty | 223 mod 5 = 3 and 144 mod 5 = 4 (the figure only shows 3 of the 5 keys) |
| 37 | Quadratic probing is "also known as the mid-square method" | Mid-square is a *hash-function* method, not a probing method |
| 38, 41 | Collision function written $f(i) = i*2$ / $(u + i*2)$ | Should be $i^2$, as the worked steps actually use |
| 41 | $73 \bmod 7 = 4$ | $73 \bmod 7 = 3$ |
| 41 | 101 → collision "??" | With the slide's own placements slot 3 was free. With the correct 73 → 3, 101 collides at 3 and goes to $3 + 1^2 = 4$ |
| 49 | Formula written $(u + v^i) \bmod m$ | $(u + v \cdot i) \bmod m$ |
| 54 | $(7 + 9 \times 2) \bmod 10 = 6$ | $25 \bmod 10 = 5$ (and slot 6 is empty in that table) |
| 54 | Key 7 row: "cannot map k 13" | Should read "cannot map **k 7**" |
| 89 | Bucket B2 shown as {5, 21, 13, 9} | $9 = 1001$ → **001** → stays in B = {1, 9}; B2 = {5, 21, 13} |
| 97 | After deleting 17: "001 → {1}, 101 → {5}" | 13 hasn't been deleted yet: 101 → **{5, 13}** |
| 110 | MSB example: directory 01 has no pointer; local depths shown as 1 / blank | 00 and 01 both → [5, 6, 11] (LD 1); [17, 22] and [24, 30] have LD **2** |
| 126 | Sparse lookup: "largest search-key value **<** K" | **≤ K** (as idx slide 25 states) |
| 128 | Secondary index "should be a candidate key" | It can be on **any non-ordering field**, key or non-key (slide 129) |
| 136, 138, idx 38–39 | B-/B+-tree called a "balanced **binary** search tree" | A balanced **multiway** (m-ary) search tree. Linked leaves are a B+-tree feature |
| 138 | "Lead nodes", "2 to 4 values", "3 to 5 children" for n = 3 | "Leaf"; the stated ranges don't match n = 3 |
| 142 vs 124/131 | El Said 80000, Califieri 60000 | Inconsistent with the other slides (El Said 60000, Califieri 62000). Irrelevant to the B+ tree structure |
| 147, idx 42, 49, 53 | Leaf max keys = $m$, leaf min keys = $\lceil m/2 \rceil$ | Every worked example uses leaf max $= m-1$ and leaf min $= \lceil (m-1)/2 \rceil$ (as slide 155 states) |
| 153–154 | "Delete 52" – result never shown | See §4.6: borrow 58 from the right sibling, separator 58 → 59 |
| 160 | "Delete 6, 8, 10, 12" – 12 is never deleted | After deleting 12: root [14 18] → [2 4], [14 16], [18 20] |
| 194 | "σ is the predicate … prepositional logic"; output text says 300000 | σ is the *operator*, $p$ is the predicate, *propositional* logic; the condition is SALARY > 30,000 |
| 203 | UNION symbol typed as υ (upsilon) | ∪ |
| 212 | Result table titled "UNION" | It is the result of **INTERSECT** |
| 217 | Caption (c) "STUDENT ∪ INSTRUCTOR" | (c) is **STUDENT ∩ INSTRUCTOR** |
| 231 | `Faculty.FacSSM` | `Faculty.FacSSN` |
| 249 | Classinfo header "ID, NAME" | "ID, ADDRESS" |
| 250 | Result OrderDate values differ from the input table | Dates should match the Orders table; the row structure is correct |
| 254 | $\pi_{name}(Student - (Student \bowtie Enroll))$ | Not union-compatible. Use $\pi_{name}(Student \bowtie (\pi_{sid}(Student) - \pi_{sid}(Enroll)))$ |
| 272 | `P.NUMBER` | `P.PNUMBER` |
| 276 | `PNMUBER` | `PNUMBER` |
| 281 | $E_1 \bowtie_{\sigma_{\theta_2}} E_2$ | $E_1 \bowtie_{\theta_2} E_2$ |
| idx 20 | Gold's ID 33465 | 33456 (as on the other slides) |
| idx 32 | Secondary-level index lists 320 before 310 | Index entries must be sorted: 300, 310, 320 |
