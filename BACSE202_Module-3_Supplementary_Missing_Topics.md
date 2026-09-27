# BACSE202 — Module 3: Supplementary Notes (Topics Missing or Thin in the Slides)

> These are the Module 3 syllabus topics that the course PPTs either **do not cover at all** or cover in only a line or two:
>
> | Topic | Coverage in the course PPTs |
> |---|---|
> | Tuple Relational Calculus | **Not covered at all** |
> | Operations on files | 1 slide of bullets |
> | Files of unordered and ordered records | 1 slide of bullets + binary search algorithm |
> | Linear hashing (dynamic hashing) | 1 bullet, no example |
> | Dynamic multilevel indexing | Never named; only implied through the B-tree/B+-tree slides |
>
> Reference: R. Elmasri & S. B. Navathe, *Fundamentals of Database Systems*, 7th ed. (the course textbook): Ch. 16 (files, hashing), Ch. 17 (indexing), Ch. 8 (relational calculus).
> Main notes: [BACSE202_Module-3_Physical_Database_Design_and_Query_Processing.md](BACSE202_Module-3_Physical_Database_Design_and_Query_Processing.md)

---

## Table of Contents

1. [Operations on Files](#1-operations-on-files)
2. [Files of Unordered Records (Heap Files)](#2-files-of-unordered-records-heap-files)
3. [Files of Ordered Records (Sorted / Sequential Files)](#3-files-of-ordered-records-sorted--sequential-files)
4. [Linear Hashing](#4-linear-hashing)
5. [Dynamic Multilevel Indexing (B-Trees and B+-Trees)](#5-dynamic-multilevel-indexing-b-trees-and-b-trees)
6. [Tuple Relational Calculus](#6-tuple-relational-calculus)
7. [Quick Revision](#7-quick-revision)

---

# 1. Operations on Files

## 1.1 Basic terms

| Term | Meaning |
|---|---|
| **File** | A sequence of records stored in disk blocks |
| **File organisation** | How the records are *physically placed* on disk: heap, sorted, hashed, B-tree … |
| **Access method** | The group of programs/operations used to reach the records (e.g. "binary search", "hash lookup"). One organisation can support several access methods |
| **File header (descriptor)** | Stores information about the file: record format, field sizes, block addresses, address of the last block, etc. |
| **Current record pointer** | The record that the *next* operation works on (like a cursor) |
| **Selection condition** | The condition used to find records, e.g. `Ssn = '123456789'` or `Salary > 30000 AND Dno = 5` |
| **Static file** | A file on which updates are rare |
| **Dynamic file** | A file that changes constantly (frequent inserts/deletes) |

## 1.2 Record-at-a-time operations

These act on **one record** at a time, using the current record pointer.

| Operation | What it does |
|---|---|
| `Open` | Prepares the file for reading/writing: allocates buffers, reads the header, sets the pointer to the start |
| `Reset` | Sets the file pointer back to the beginning |
| `Find` (Locate) | Searches for the **first** record satisfying the condition; transfers its block into a buffer and makes it the current record |
| `Read` (Get) | Copies the current record from the buffer into a program variable; may also advance the pointer |
| `FindNext` | Searches for the **next** record satisfying the condition |
| `Delete` | Deletes the current record (and eventually updates the file on disk) |
| `Modify` | Changes some field values of the current record |
| `Insert` | Inserts a new record: finds the correct block, puts the record there, writes the block back |
| `Close` | Releases buffers and does any cleanup |

## 1.3 Set-at-a-time operations

These act on a **set of records** in one call.

| Operation | What it does |
|---|---|
| `Scan` | If the file was just opened/reset, returns the first record; otherwise returns the next one |
| `FindAll` | Locates **all** records satisfying the condition |
| `Find n` | Locates the first record satisfying the condition, then the next $`n-1`$ records |
| `FindOrdered` | Retrieves all records in a specified order |
| `Reorganize` | Rebuilds the file, e.g. to remove deleted records or re-sort it |

## 1.4 Example: how the DBMS uses these operations

```sql
SELECT * FROM EMPLOYEE WHERE Dno = 5;
```

A low-level plan using file operations:

```
Open(EMPLOYEE)
Find(EMPLOYEE, Dno = 5)             -- first matching record becomes current
while record found do
    Read(current record) → output it
    FindNext(EMPLOYEE, Dno = 5)
end
Close(EMPLOYEE)
```

```sql
UPDATE EMPLOYEE SET Salary = Salary * 1.1 WHERE Ssn = '123456789';
```

```
Open(EMPLOYEE); Find(EMPLOYEE, Ssn = '123456789'); Modify(Salary); Close(EMPLOYEE)
```

How **fast** `Find`, `Insert` and `Delete` are depends entirely on the **file organisation**. That's the point of Sections 2–5.

## 1.5 Records and blocks (needed for the cost calculations)

- **Blocking factor** (records per block, fixed-length records of size $`R`$, block size $`B`$):

```math
bfr = \left\lfloor \frac{B}{R} \right\rfloor, \qquad b = \left\lceil \frac{r}{bfr} \right\rceil \text{ blocks for } r \text{ records}
```

- **Unspanned** organisation: a record may **not** cross a block boundary. Unused space per block $`= B - bfr \cdot R`$. This is used for fixed-length records.
- **Spanned** organisation: a record may be split across blocks, with a pointer at the end of the block to the rest. No space is wasted, and records larger than a block become possible. This is used for large or variable-length records.

**Example:** $`B = 512`$ bytes, $`R = 200`$ bytes.
- Unspanned: $`bfr = \lfloor 512/200 \rfloor = 2`$. Each block wastes $`512 - 400 = 112`$ bytes (22%).
- Spanned: on average $`512/200 = 2.56`$ records per block, with no waste (ignoring the pointer).

**Allocating blocks to a file:**
- **Contiguous**: consecutive blocks; fast sequential read, hard to grow.
- **Linked**: each block points to the next; easy to grow, slow to jump.
- **Clusters**: linked runs of contiguous blocks.
- **Indexed**: index blocks point to the data blocks.

---

# 2. Files of Unordered Records (Heap Files)

## 2.1 Organisation

Records are placed in the file **in the order they are inserted**: each new record goes at the **end** of the file. This is the simplest organisation, also called a **heap** or **pile** file.

```
Block 1: [ Ravi  | 104 ]  [ Anu   | 311 ]  [ Kiran | 150 ]
Block 2: [ Zara  | 120 ]  [ Mohan | 101 ]  [ ----deleted---- ]
Block 3: [ Bala  | 275 ]  [ (free) ]       [ (free) ]         ← last block (address kept in header)
```

## 2.2 Operations and their cost

Let the file have $`b`$ blocks.

| Operation | How | Cost (block accesses) |
|---|---|---|
| **Insert** | Copy the last block (its address is in the header) into a buffer, add the record, write the block back | **≈ 2** (1 read + 1 write), so very efficient |
| **Search** (any field) | **Linear search**: read block after block | Average $`b/2`$ if exactly one record matches; **$`b`$** if none matches or all matches are needed. $`O(b)`$ |
| **Delete** | Find the record's block, remove it, write the block back. This leaves an **unused hole**. Alternative: set a **deletion marker (bit)** in the record | Search + 1 write |
| **Modify** (fixed length) | Find, change, rewrite | Search + 1 write |
| **Modify** (variable length) | May need to **delete the old record and insert a new one**, because the new version may not fit | Search + 2–3 |
| **Read in order of a field** | Must first make a **sorted copy** (external sort) | Very expensive |
| **Reorganize** | Periodically pack the blocks to reclaim holes and deleted records | $`b`$ reads + writes |

> Deleted space can instead be **reused** by later inserts (keep a free-space list), but then insertion is no longer "just append at the end".

## 2.3 Relative (direct) files

If a heap file has **fixed-length records** and **unspanned** blocks, record number $`i`$ (counting from 0) can be located **directly**:

```math
\text{block} = \left\lfloor \frac{i}{bfr} \right\rfloor, \qquad \text{position within block} = i \bmod bfr
```

This gives access *by position*, but it doesn't help searching *by value*.

## 2.4 Worked example: heap file

$`r = 30{,}000`$ EMPLOYEE records, $`R = 100`$ bytes, $`B = 1024`$ bytes, unspanned.

```math
bfr = \lfloor 1024/100 \rfloor = 10, \qquad b = \lceil 30000 / 10 \rceil = 3000 \text{ blocks}
```

| Question | Answer |
|---|---|
| Search on `Ssn` (unique), record exists | Average $`3000/2 = 1500`$ block accesses |
| Search on `Ssn`, record does **not** exist | 3000 (must scan everything) |
| Find **all** employees of `Dno = 5` | 3000 (any block may contain one) |
| Insert a new employee | 2 (read the last block, write it back) |
| Where is record number 12,345 (0-based)? | Block $`\lfloor 12345/10 \rfloor = 1234`$, position $`12345 \bmod 10 = 5`$ |

## 2.5 When to use a heap file

- ✅ Bulk loading (insert many records, then build indexes afterwards)
- ✅ Small tables
- ✅ When most queries read the **whole** file anyway
- ✅ Combined with a **secondary index** (the index gives fast lookup, the heap gives cheap insert)
- ❌ Frequent searches on a specific value
- ❌ Ordered output

---

# 3. Files of Ordered Records (Sorted / Sequential Files)

## 3.1 Organisation

Records are **physically sorted** on the values of one field, the **ordering field**. If the ordering field is also a key (unique), it is called the **ordering key**.

```
Ordering field = Name
Block 1: Aaron, Ed      | Abbot, Diane  | Acosta, Marc
Block 2: Adams, John    | Adams, Robin  | Akers, Jan
Block 3: Alexander, Ed  | Alfred, Bob   | Allen, Sam
 ...
Block n: Wright, Pam    | Wyatt, Charles| Zimmer, Byron
```

## 3.2 Advantages

1. **Reading records in ordering-field order** needs no sorting.
2. Finding the **next record** in order usually needs **no extra block access** (it's in the same block).
3. **Binary search** is possible on the ordering field: $`\lceil \log_2 b \rceil`$ block accesses.
4. **Range queries** on the ordering field are efficient. For example, for `Name BETWEEN 'Adams' AND 'Allen'`, binary search to the first match, then read consecutive blocks.

> ⚠️ There is **no advantage** for a search on a **non-ordering** field, which still needs a linear search.

## 3.3 Binary search on the ordering key (block level)

```
l ← 1; u ← b;
while (u ≥ l) do
begin
    i ← (l + u) div 2;
    read block i into the buffer;
    if K < (ordering key of FIRST record in block i) then u ← i − 1
    else if K > (ordering key of LAST record in block i) then l ← i + 1
    else if record with key K is in the buffer then goto found
    else goto notfound;
end;
goto notfound;
```

### Worked trace

An ordered file of 8 blocks, 3 records each (keys shown):

| Block | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|---|---|---|---|---|---|---|---|
| Keys | 5, 8, 12 | 15, 19, 22 | 25, 30, 33 | 37, 40, 44 | 48, 51, 55 | 60, 63, 67 | 70, 74, 79 | 82, 88, 95 |

**Search K = 63**

| Step | l | u | i = (l+u) div 2 | Block i range | Decision |
|---|---|---|---|---|---|
| 1 | 1 | 8 | 4 | 37 … 44 | 63 > 44 → l = 5 |
| 2 | 5 | 8 | 6 | 60 … 67 | within range, 63 is in the buffer → **found** |

**2 block accesses** (a linear search would need 6).

**Search K = 50** (not in the file)

| Step | l | u | i | Block i range | Decision |
|---|---|---|---|---|---|
| 1 | 1 | 8 | 4 | 37 … 44 | 50 > 44 → l = 5 |
| 2 | 5 | 8 | 6 | 60 … 67 | 50 < 60 → u = 5 |
| 3 | 5 | 5 | 5 | 48 … 55 | within range but 50 not in the block → **notfound** |

**3 block accesses**, and we know for certain that 50 is absent.

## 3.4 Insertion and deletion — the weak spot

**Insertion** is **expensive**: the record must go into its **correct sorted position**, so on average **half the file** must be shifted (read and rewritten), about $`b/2`$ reads + $`b/2`$ writes.

Techniques to make it cheaper:

| Technique | Idea | Trade-off |
|---|---|---|
| **Free space in each block** | Leave some empty slots per block when loading | Space waste; the problem returns once a block fills |
| **Overflow (transaction) file** | New records go into an **unordered overflow file**; periodically **merge** it with the main file | Search = binary search on the main file **plus linear search on the overflow file** |
| **Linked overflow per block** | Each block has a chain of overflow records | Chains slow down searches |

**Deletion** is also expensive if records are physically removed (shifting). Instead use **deletion markers** and **periodic reorganisation**.

**Modifying the ordering field** = delete the old record + insert the new one (it moves). Modifying a **non-ordering** field is cheap: find, change, rewrite.

## 3.5 Worked example: ordered file

The same file as §2.4: $`b = 3000`$ blocks, ordered on `Ssn`.

| Question | Heap file | Ordered file |
|---|---|---|
| Find one employee by `Ssn` | avg 1500 | $`\lceil \log_2 3000 \rceil = 12`$ |
| Find employees with `Ssn` between X and Y (say 50 blocks of matches) | 3000 | $`12 + 50 = 62`$ (approx.) |
| Find all employees with `Dno = 5` (non-ordering field) | 3000 | 3000 (**no benefit**) |
| Insert a new employee | 2 | avg $`b/2 = 1500`$ reads + 1500 writes (or 1 write to an overflow file) |
| List all employees in `Ssn` order | sort needed | 3000 (just read sequentially) |

## 3.6 Comparison of the basic file organisations

| Organisation | Access / search method | Average blocks to access one record |
|---|---|---|
| Heap (unordered) | Sequential scan (linear search) | $`b/2`$ |
| Ordered | Sequential scan | $`b/2`$ |
| Ordered | Binary search on the ordering key | $`\log_2 b`$ |
| Hashed (static) | Hash on the key | ≈ 1 (plus overflow) |

| | Heap | Ordered | Hashed |
|---|---|---|---|
| Insert | ✅ cheap | ❌ expensive | ✅ cheap (unless overflow) |
| Equality search on the key | ❌ $`b/2`$ | ✅ $`\log_2 b`$ | ✅ ≈ 1 |
| Range search on the key | ❌ $`b`$ | ✅ | ❌ (hashing destroys order) |
| Ordered output | ❌ sort | ✅ | ❌ sort |
| Search on another field | ❌ $`b`$ | ❌ $`b`$ | ❌ $`b`$ |

> Ordered files are rarely used alone in databases. They are usually combined with a **primary index** (indexed-sequential file), and the B+-tree (§5) solves their insertion problem.

---

# 4. Linear Hashing

## 4.1 Motivation

- **Static hashing** has a fixed number of buckets, so the file cannot grow gracefully.
- **Extendible hashing** grows, but needs a **directory**, and the directory **doubles** (it can become huge with skewed data).
- **Linear hashing** (W. Litwin, 1980) lets the file **grow and shrink one bucket at a time, without any directory**.

## 4.2 Key ideas

1. Start with $`M`$ buckets numbered $`0 \dots M-1`$ and the hash function $`h_0(K) = K \bmod M`$.
2. Buckets are split **in a fixed linear order**: bucket 0, then 1, then 2, …, regardless of which bucket overflowed. A variable $`n`$ (the **split pointer**, "next bucket to split") records where we are.
3. When a split is triggered, bucket $`n`$ is split using the **next-level hash function** $`h_{j+1}(K) = K \bmod 2^{j+1} M`$. Its records go either to bucket $`n`$ or to a **new bucket** $`n + 2^j M`$ appended at the end of the file. Then $`n \leftarrow n + 1`$.
4. The bucket that actually **overflowed** is handled temporarily with an **overflow chain**. Its turn to be split will come.
5. When $`n = 2^j M`$ (every bucket of the current level has been split), the round is over: set $`n \leftarrow 0`$, $`j \leftarrow j + 1`$. The file now has $`2^{j} M`$ buckets.

$`j`$ is called the **level**. At any time only **two** hash functions are in use, $`h_j`$ and $`h_{j+1}`$.

## 4.3 Address computation (search)

```
a ← h_j(K)
if a < n then            -- bucket a has already been split in this round
    a ← h_{j+1}(K)
search bucket a (and its overflow chain)
```

## 4.4 When is a split triggered?

| Policy | Rule |
|---|---|
| **Uncontrolled splitting** | Split (bucket $`n`$) **whenever any insertion causes an overflow** |
| **Controlled splitting** | Split whenever the **load factor** $`\ell = r / (bfr \cdot N)`$ exceeds a threshold (e.g. 0.9). Here $`r`$ = records and $`N`$ = current number of buckets |

**Shrinking:** when the load factor falls below a threshold, buckets are **merged** in reverse order. Decrement $`n`$ and merge bucket $`n + 2^j M`$ back into bucket $`n`$; when $`n`$ goes below 0, decrement $`j`$.

## 4.5 Worked example

**Setup:** $`M = 4`$ initial buckets, **bucket capacity = 2**, uncontrolled splitting (split on every overflow).
Level $`j = 0`$: $`h_0(K) = K \bmod 4`$, $`h_1(K) = K \bmod 8`$. Split pointer $`n = 0`$.

**Insert:** 8, 12, 5, 9, 6, 7, 11, 3, 15, 2, 14, 13, 19

### Step 1 — insert 8, 12, 5, 9, 6, 7, 11 (no overflow)

| Key | $`h_0 = K \bmod 4`$ | Bucket |
|---|---|---|
| 8 | 0 | 0 |
| 12 | 0 | 0 |
| 5 | 1 | 1 |
| 9 | 1 | 1 |
| 6 | 2 | 2 |
| 7 | 3 | 3 |
| 11 | 3 | 3 |

```
n = 0, j = 0
B0: [8, 12]
B1: [5, 9]
B2: [6]
B3: [7, 11]
```

### Step 2 — insert 3 → overflow → split bucket 0

- $`h_0(3) = 3 \ge n`$ → bucket 3, which is full → put 3 in an **overflow block** of B3.
- The overflow **triggers a split of bucket $`n = 0`$**, not bucket 3.
- Rehash B0's keys with $`h_1 = K \bmod 8`$: 8 → 0, 12 → **4**. Create new bucket B4 ($`= 0 + 2^0 \cdot 4`$).
- $`n \leftarrow 1`$.

```
n = 1, j = 0
B0: [8]
B1: [5, 9]
B2: [6]
B3: [7, 11] → overflow [3]
B4: [12]                         ← new
```

> Note: bucket 3 overflowed, but bucket 0 was split. This is the "linear" in linear hashing.

### Step 3 — insert 15 → overflow → split bucket 1

- $`h_0(15) = 3`$, and $`3 \ge n = 1`$, so use bucket 3 → full → overflow chain [3, 15].
- Split bucket $`n = 1`$ with $`h_1`$: 5 → **5**, 9 → 1. New bucket B5.
- $`n \leftarrow 2`$.

```
n = 2, j = 0
B0: [8]
B1: [9]
B2: [6]
B3: [7, 11] → overflow [3, 15]
B4: [12]
B5: [5]                          ← new
```

### Step 4 — insert 2 (no overflow)

$`h_0(2) = 2 \ge n = 2`$ → bucket 2 → B2 = [6, 2].

### Step 5 — insert 14 → overflow → split bucket 2

- $`h_0(14) = 2 \ge 2`$ → bucket 2 is full → overflow [14].
- Split bucket $`n = 2`$ with $`h_1`$ (all its keys, including the overflow): 6 → **6**, 2 → 2, 14 → **6**. New bucket B6. The overflow block disappears.
- $`n \leftarrow 3`$.

```
n = 3, j = 0
B0: [8]
B1: [9]
B2: [2]
B3: [7, 11] → overflow [3, 15]
B4: [12]
B5: [5]
B6: [6, 14]                      ← new
```

### Step 6 — insert 13 (address computation uses $`h_1`$)

- $`h_0(13) = 1`$, and $`1 < n = 3`$, so bucket 1 has already been split → use $`h_1(13) = 13 \bmod 8 = 5`$.
- B5 = [5, 13]. No overflow.

### Step 7 — insert 19 → overflow → split bucket 3 → round complete

- $`h_0(19) = 3 \ge n = 3`$ → bucket 3 → overflow chain [3, 15, 19].
- Split bucket $`n = 3`$ with $`h_1`$. Its keys are {7, 11, 3, 15, 19}: 7 → **7**, 11 → 3, 3 → 3, 15 → **7**, 19 → 3.
  - B3 gets {11, 3, 19}. Capacity is 2, so B3 = [11, 3] → overflow [19].
  - New bucket B7 = [7, 15].
- $`n \leftarrow 4`$. Now $`n = 2^0 \cdot M = 4`$ → **round complete**: $`n \leftarrow 0`$, $`j \leftarrow 1`$. From now on $`h_j = K \bmod 8`$ and $`h_{j+1} = K \bmod 16`$.

**Final state:**

```
n = 0, j = 1      (hash functions now: h1 = K mod 8, h2 = K mod 16)
B0: [8]
B1: [9]
B2: [2]
B3: [11, 3] → overflow [19]
B4: [12]
B5: [5, 13]
B6: [6, 14]
B7: [7, 15]
```

All 13 keys are present: 8 | 9 | 2 | 11, 3, 19 | 12 | 5, 13 | 6, 14 | 7, 15.

### Searching in the final file

| Search | $`a = K \bmod 8`$ | $`a < n = 0`$? | Bucket | Result |
|---|---|---|---|---|
| 19 | 3 | No | B3 | Found in the overflow block (2 block accesses) |
| 13 | 5 | No | B5 | Found (1 block access) |
| 10 | 2 | No | B2 | Not found |

### Searching in the middle of the process (state after Step 5: $`n = 3`$, $`j = 0`$)

| Search | $`h_0 = K \bmod 4`$ | $`< n`$? | Final address | Bucket |
|---|---|---|---|---|
| 12 | 0 | Yes (0 < 3) | $`h_1 = 12 \bmod 8 = 4`$ | B4 ✓ |
| 9 | 1 | Yes | $`9 \bmod 8 = 1`$ | B1 ✓ |
| 14 | 2 | Yes | $`14 \bmod 8 = 6`$ | B6 ✓ |
| 15 | 3 | No (3 ≥ 3) | 3 | B3 (overflow) ✓ |

## 4.6 Extendible vs. linear hashing

| Feature | Extendible hashing | Linear hashing |
|---|---|---|
| Directory | **Yes**, $`2^{GD}`$ pointers, doubles | **No directory** |
| Which bucket splits | The bucket that overflowed | The bucket at the split pointer $`n`$ (round-robin) |
| Overflow chains | Normally none | Yes, temporarily |
| Growth | Directory doubles; buckets grow one at a time | File grows exactly one bucket per split |
| Search cost | 1 directory + 1 bucket (≈ 1 if the directory is in memory) | ≈ 1 bucket (+ overflow blocks) |
| Skewed data | Directory can explode | Overflow chains get long |
| Hash functions in use | One, with GD bits | Two: $`h_j`$ and $`h_{j+1}`$ |

---

# 5. Dynamic Multilevel Indexing (B-Trees and B+-Trees)

## 5.1 The problem with static multilevel indexes

A (static) **multilevel index**, such as ISAM, is built as a sequence of **physically ordered files**:

```math
\text{fan-out } fo = bfr_i = \left\lfloor \frac{B}{R_i} \right\rfloor, \qquad
r_{j+1} = \left\lceil \frac{r_j}{fo} \right\rceil, \qquad
t = \left\lceil \log_{fo} r_1 \right\rceil \text{ levels}, \qquad \text{search} = t + 1 \text{ block accesses}
```

### Example (static multilevel index)

Using the file from §2.4: $`r = 30{,}000`$, $`B = 1024`$, $`b = 3000`$ data blocks. Index entry = key `Ssn` ($`V = 9`$ B) + block pointer ($`P = 6`$ B), so $`R_i = 15`$ B:

```math
fo = \lfloor 1024 / 15 \rfloor = 68
```

| Index | Level 1 entries | Level 1 blocks | Level 2 blocks | Level 3 blocks | Levels $`t`$ | Accesses $`t+1`$ | Single-level binary search |
|---|---|---|---|---|---|---|---|
| **Primary** (sparse, 1 per data block) | 3000 | $`\lceil 3000/68 \rceil = 45`$ | $`\lceil 45/68 \rceil = 1`$ | – | 2 | **3** | $`\lceil \log_2 45 \rceil + 1 = 7`$ |
| **Secondary** on a key (dense, 1 per record) | 30,000 | $`\lceil 30000/68 \rceil = 442`$ | $`\lceil 442/68 \rceil = 7`$ | 1 | 3 | **4** | $`\lceil \log_2 442 \rceil + 1 = 10`$ |

**But:** every level is an ordered file, so **inserting or deleting** a data record means shifting entries at every level. That is the same problem ordered files have (§3.4). ISAM handles it with **overflow areas**, which degrade performance until the whole index is **reorganised**.

**Solution: dynamic multilevel indexes.** Make every index level a **tree of nodes (one node = one disk block)** and leave **free space in every node**. Inserts and deletes are then handled by **local splitting and merging** of nodes, so the tree stays **balanced** automatically. The two structures are the **B-tree** and the **B+-tree**.

## 5.2 Search trees (background)

A **search tree of order $`p`$**: each node holds at most $`p - 1`$ keys and $`p`$ pointers

```math
\langle P_1, K_1, P_2, K_2, \dots, K_{q-1}, P_q \rangle, \quad q \le p, \quad K_1 < K_2 < \dots < K_{q-1}
```

and every key $`X`$ in the subtree under $`P_i`$ satisfies $`K_{i-1} < X < K_i`$.

The problem is that a plain search tree can become **unbalanced** (e.g. insert keys in sorted order and it degenerates into a chain). B-trees add rules that **keep the tree balanced** and nodes **at least half full**.

## 5.3 B-tree

### Definition (order $`p`$)

Each node has the form

```math
\langle P_1, \langle K_1, Pr_1 \rangle, P_2, \langle K_2, Pr_2 \rangle, \dots, \langle K_{q-1}, Pr_{q-1} \rangle, P_q \rangle, \quad q \le p
```

- $`P_i`$ = **tree pointer** (to a child node), and $`Pr_i`$ = **data pointer** (to the record with key $`K_i`$).
- **Every node, internal or leaf, stores data pointers**, and **each key appears exactly once** in the tree.
- $`K_1 < K_2 < \dots < K_{q-1}`$, and the keys in subtree $`P_i`$ lie between $`K_{i-1}`$ and $`K_i`$.
- Each node has **at most $`p`$** tree pointers.
- Each node **except the root and the leaves** has **at least $`\lceil p/2 \rceil`$** tree pointers.
- The root has at least 2 tree pointers unless it is the only node.
- A node with $`q`$ tree pointers has $`q - 1`$ keys.
- **All leaves are at the same level.** Leaf tree pointers are null.

### B-tree insertion

1. Search for the leaf where the key belongs and insert it in sorted order.
2. If the node now has $`p`$ keys (**overflow**), split it: the **median key moves up** to the parent (it is **not** kept below), and the left and right halves become two nodes.
3. If the parent overflows, split it too. If the root splits, a new root is created and the tree grows **one level taller at the top**.

### Example: B-tree of order $`p = 3`$ (max 2 keys per node), insert 8, 5, 1, 7, 3, 12, 9, 6

| Insert | Action | Tree after |
|---|---|---|
| 8 | | `[8]` |
| 5 | | `[5 8]` |
| 1 | `[1 5 8]` overflow → median 5 moves up | `[5]` → `[1]`, `[8]` |
| 7 | 7 > 5 | `[5]` → `[1]`, `[7 8]` |
| 3 | 3 < 5 | `[5]` → `[1 3]`, `[7 8]` |
| 12 | `[7 8 12]` overflow → median 8 moves up to the root | `[5 8]` → `[1 3]`, `[7]`, `[12]` |
| 9 | 9 > 8 | `[5 8]` → `[1 3]`, `[7]`, `[9 12]` |
| 6 | 5 < 6 < 8 | `[5 8]` → `[1 3]`, `[6 7]`, `[9 12]` |

```
Final B-tree (order 3):
                 [ 5 | 8 ]
          ┌──────────┼──────────┐
       [1 | 3]    [6 | 7]    [9 | 12]
Every key appears once; every key carries its own data pointer.
```

### B-tree deletion (summary)

- Deleting from a **leaf** that stays at least half full: just remove the key.
- Deleting from an **internal node**: replace the key with its **in-order predecessor or successor** (which is in a leaf), then delete that from the leaf.
- On **underflow** (fewer than $`\lceil p/2 \rceil - 1`$ keys): **borrow** from a sibling through the parent (rotation), or **merge** with a sibling **plus the separating key pulled down from the parent**. Merging can propagate up, and if the root becomes empty the height shrinks by 1.

## 5.4 B+-tree (the variation used in practice)

The differences from a B-tree:

| B-tree | B+-tree |
|---|---|
| Data pointers in **all** nodes | Data pointers **only in the leaves** |
| Each key appears **once** | Keys in internal nodes are **repeated** in the leaves (internal keys are only "signposts") |
| No leaf chain | **Leaves are linked** (next-leaf pointer) → fast range/sequential scans |
| Internal nodes hold $`\langle K, Pr \rangle`$ pairs, so **lower fan-out** | Internal nodes hold only keys + tree pointers, so **higher fan-out** and a **shallower tree** |
| Search may stop early at an internal node | Search always goes to a leaf, so cost is uniform |
| Leaf split: median moves up | Leaf split: first key of the right half is **copied** up; internal split: middle key **moves** up |

### Example: B+-tree of order 3 (max 2 keys per node), same keys 8, 5, 1, 7, 3, 12, 9, 6

(Leaf split of 3 keys: left keeps 1, right gets 2, and the first key of the right leaf is **copied** up. Internal split: the middle key **moves** up.)

| Insert | Action | Tree after |
|---|---|---|
| 8, 5 | | `[5 8]` |
| 1 | `[1 5 8]` → `[1]` \| `[5 8]`, copy 5 | `[5]` → `[1]`, `[5 8]` |
| 7 | `[5 7 8]` → `[5]` \| `[7 8]`, copy 7 | `[5 7]` → `[1]`, `[5]`, `[7 8]` |
| 3 | | `[5 7]` → `[1 3]`, `[5]`, `[7 8]` |
| 12 | `[7 8 12]` → `[7]` \| `[8 12]`, copy 8 → root `[5 7 8]` overflow → 7 moves up | `[7]` → (`[5]` → `[1 3]`, `[5]`), (`[8]` → `[7]`, `[8 12]`) |
| 9 | `[8 9 12]` → `[8]` \| `[9 12]`, copy 9 → internal `[8 9]` | `[7]` → (`[5]` → `[1 3]`, `[5]`), (`[8 9]` → `[7]`, `[8]`, `[9 12]`) |
| 6 | leaf `[5]` → `[5 6]` | see below |

```
Final B+-tree (order 3):
                         [ 7 ]
              ┌────────────┴─────────────┐
            [ 5 ]                     [ 8 | 9 ]
          ┌───┴───┐              ┌───────┼────────┐
       [1 3] → [5 6]    →     [7]  →  [8]   →  [9 12]
Leaves are linked; only the leaves have data pointers; 5, 7, 8, 9 appear twice.
```

Compare: the **B-tree** holds the same 8 keys in **2 levels**, while this **B+-tree** needs **3 levels** for such a tiny order. With realistic block sizes it's the other way round: the B+-tree's larger fan-out makes it **shallower** (next section).

## 5.5 Worked example: order and capacity calculation

**Given:** key field $`V = 9`$ bytes, block size $`B = 512`$ bytes, data (record) pointer $`Pr = 7`$ bytes, block (tree) pointer $`P = 6`$ bytes. Assume nodes are on average **69% full** (typical after random inserts).

### B-tree

One node holds $`p`$ tree pointers and $`p-1`$ ⟨key, data pointer⟩ pairs, and must fit in a block:

```math
p \cdot P + (p-1)(Pr + V) \le B \;\Rightarrow\; 6p + 16(p-1) \le 512 \;\Rightarrow\; 22p \le 528 \;\Rightarrow\; p = 24
```

At 69% full: $`0.69 \times 24 \approx 16`$ pointers and 15 keys per node on average.

| Level | Nodes | Keys (entries) | Tree pointers |
|---|---|---|---|
| Root | 1 | 15 | 16 |
| Level 1 | 16 | 240 | 256 |
| Level 2 | 256 | 3,840 | 4,096 |
| Level 3 | 4,096 | 61,440 | – |
| **Total** | | **65,535 entries** | |

### B+-tree

**Internal node:** $`p`$ tree pointers and $`p-1`$ keys (no data pointers):

```math
p \cdot P + (p-1) V \le B \;\Rightarrow\; 6p + 9(p-1) \le 512 \;\Rightarrow\; 15p \le 521 \;\Rightarrow\; p = 34
```

**Leaf node:** $`p_{leaf}`$ ⟨key, data pointer⟩ pairs + 1 next-leaf pointer:

```math
p_{leaf}(Pr + V) + P \le B \;\Rightarrow\; 16\,p_{leaf} + 6 \le 512 \;\Rightarrow\; p_{leaf} = 31
```

At 69% full: internal nodes have $`0.69 \times 34 \approx 23`$ pointers (22 keys), and leaves have $`0.69 \times 31 \approx 21`$ data pointers.

| Level | Nodes | Keys | Pointers |
|---|---|---|---|
| Root | 1 | 22 | 23 |
| Level 1 | 23 | 506 | 529 |
| Level 2 | 529 | 11,638 | 12,167 |
| Leaf level | 12,167 | – | **255,507 data pointers** |

➡ With the **same 4 levels**, the B+-tree indexes **255,507 records** against **65,535** for the B-tree. That is almost **4× more**, because internal B+-tree nodes don't waste space on data pointers. **This is why every major DBMS uses B+-trees.**

## 5.6 Search cost in a dynamic multilevel index

```math
\text{height} \approx \left\lceil \log_{\lceil p/2 \rceil} N \right\rceil \text{ (worst case)}, \qquad
\text{search cost} = \text{height} \;(+1 \text{ to read the data block})
```

For $`N = 1{,}000{,}000`$ keys and $`p = 100`$: $`\lceil \log_{50} 10^6 \rceil = \lceil 3.53 \rceil = 4`$ levels in the worst case. Typically it's 3, since nodes are fuller than half. A record is found in about **4–5 block accesses**, compared with $`\lceil \log_2 b \rceil`$ ≈ 17 for binary search on a sorted file of 100,000 blocks.

## 5.7 Summary: static vs. dynamic multilevel index

| | Static multilevel (ISAM) | Dynamic multilevel (B / B+-tree) |
|---|---|---|
| Structure | Levels are ordered files | Levels are tree nodes (blocks) with free space |
| Insert/Delete | Overflow chains; periodic full reorganisation | Local split/merge; always balanced |
| Performance over time | Degrades | Stable, $`O(\log_{fo} N)`$ |
| Space | Compact (full blocks) | Nodes 50–100% full (≈ 69% average) |
| Used for | Read-mostly, static files | Almost all DBMS indexes |

---

# 6. Tuple Relational Calculus

## 6.1 What is relational calculus?

- A **formal, declarative** query language for the relational model. You state **what** the result should satisfy, **not how** to compute it.
- Compare **relational algebra**, which is **procedural**: a sequence of operations that says *how*.
- **SQL is based largely on tuple relational calculus** (`SELECT … FROM … WHERE` ≈ "tuples $`t`$ such that …").
- Two forms:
  - **Tuple Relational Calculus (TRC)**, where variables range over **tuples**
  - **Domain Relational Calculus (DRC)**, where variables range over **attribute values** (domains)

## 6.2 Syntax of a TRC query

```math
\{\, t_1.A_1, t_2.A_2, \dots, t_n.A_n \mid \text{COND}(t_1, t_2, \dots, t_n, t_{n+1}, \dots, t_{n+m}) \,\}
```

- $`t_i`$ are **tuple variables**; $`A_i`$ are attributes.
- The part left of $`\mid`$ is the **target list**: what to return.
- $`\text{COND}`$ is a **formula** (well-formed formula, WFF) of the calculus.
- The result is the set of all value combinations for which COND is **TRUE**.

Simplest form, returning whole tuples:

```math
\{\, t \mid \text{COND}(t) \,\}
```

## 6.3 Atoms (the building blocks of formulas)

| Atom | Meaning | Example |
|---|---|---|
| $`R(t_i)`$ | $`t_i`$ is a tuple of relation $`R`$ (called the **range relation** of $`t_i`$) | $`EMPLOYEE(t)`$ |
| $`t_i.A \;\theta\; t_j.B`$ | Compare attributes of two tuple variables, with $`\theta \in \{=, \ne, <, \le, >, \ge\}`$ | $`e.Dno = d.Dnumber`$ |
| $`t_i.A \;\theta\; c`$ or $`c \;\theta\; t_j.B`$ | Compare an attribute with a constant | $`t.Salary > 50000`$ |

## 6.4 Formulas

Built recursively:
1. Every **atom** is a formula.
2. If $`F_1`$ and $`F_2`$ are formulas, then so are $`(F_1 \wedge F_2)`$, $`(F_1 \vee F_2)`$, $`\neg F_1`$ and $`\neg F_2`$.
3. If $`F`$ is a formula and $`t`$ a tuple variable, then so are $`(\exists t)(F)`$ and $`(\forall t)(F)`$.

**Quantifiers:**
- **Existential** $`(\exists t)(F)`$ is TRUE if **at least one** tuple $`t`$ makes $`F`$ TRUE ("there exists").
- **Universal** $`(\forall t)(F)`$ is TRUE if **every** tuple $`t`$ in the universe makes $`F`$ TRUE ("for all").

**Free and bound variables:**
- A variable is **bound** if it is quantified by $`\exists`$ or $`\forall`$ in that part of the formula; otherwise it is **free**.
- **Only free variables may appear in the target list** (left of $`\mid`$).

Example: in $`\{ t.Fname \mid EMPLOYEE(t) \wedge (\exists d)(DEPARTMENT(d) \wedge d.Dnumber = t.Dno) \}`$, $`t`$ is **free** and $`d`$ is **bound**.

## 6.5 Useful transformation rules

```math
(\forall x)(P(x)) \equiv \neg(\exists x)(\neg P(x))
```
```math
(\exists x)(P(x)) \equiv \neg(\forall x)(\neg P(x))
```
```math
\neg(P \wedge Q) \equiv \neg P \vee \neg Q, \qquad \neg(P \vee Q) \equiv \neg P \wedge \neg Q
```
```math
P \Rightarrow Q \;\equiv\; \neg P \vee Q
```

The last one is essential for "for all" queries: "for every $`x`$, **if** $`x`$ is a project of dept 5 **then** …" becomes $`(\forall x)(\neg(PROJECT(x) \wedge x.Dnum = 5) \vee \dots)`$.

## 6.6 Example database 1 (small, so results can be checked)

**Student**

| sid | name | dept |
|---|---|---|
| 1 | Asha | CSE |
| 2 | Bala | ECE |
| 3 | Chen | CSE |

**Course**

| cid | title |
|---|---|
| C1 | DBMS |
| C2 | OS |

**Enroll**

| sid | cid | grade |
|---|---|---|
| 1 | C1 | A |
| 1 | C2 | B |
| 2 | C1 | A |
| 3 | C2 | C |

### Type 1 — Selection (whole tuples)

*All CSE students.*

```math
\{\, s \mid Student(s) \wedge s.dept = \text{'CSE'} \,\}
```

Result: (1, Asha, CSE), (3, Chen, CSE). RA: $`\sigma_{dept='CSE'}(Student)`$.

### Type 2 — Projection (target list)

*Names of CSE students.*

```math
\{\, s.name \mid Student(s) \wedge s.dept = \text{'CSE'} \,\}
```

Result: Asha, Chen. RA: $`\pi_{name}(\sigma_{dept='CSE'}(Student))`$.

### Type 3 — Join using $`\exists`$

*Names of students who got an 'A' in some course.*

```math
\{\, s.name \mid Student(s) \wedge (\exists e)(Enroll(e) \wedge e.sid = s.sid \wedge e.grade = \text{'A'}) \,\}
```

Check: Asha (1, C1, A) ✓, Bala (2, C1, A) ✓, Chen (only a C) ✗. **Result: Asha, Bala.**
RA: $`\pi_{name}(Student \bowtie \sigma_{grade='A'}(Enroll))`$.

### Type 4 — Join with attributes from two relations

*Student name with the title of each course they take.*

```math
\{\, s.name, c.title \mid Student(s) \wedge Course(c) \wedge (\exists e)(Enroll(e) \wedge e.sid = s.sid \wedge e.cid = c.cid) \,\}
```

Result: (Asha, DBMS), (Asha, OS), (Bala, DBMS), (Chen, OS).

### Type 5 — Negation / difference using $`\neg \exists`$

*Students who are **not** enrolled in C1.*

```math
\{\, s.name \mid Student(s) \wedge \neg(\exists e)(Enroll(e) \wedge e.sid = s.sid \wedge e.cid = \text{'C1'}) \,\}
```

Asha takes C1 ✗, Bala takes C1 ✗, Chen doesn't ✓. **Result: Chen.**
RA: $`\pi_{name}(Student \bowtie (\pi_{sid}(Student) - \pi_{sid}(\sigma_{cid='C1'}(Enroll))))`$.

### Type 6 — Division using $`\forall`$ ("enrolled in ALL courses")

```math
\{\, s.name \mid Student(s) \wedge (\forall c)\big(\neg Course(c) \vee (\exists e)(Enroll(e) \wedge e.sid = s.sid \wedge e.cid = c.cid)\big) \,\}
```

Read it as: *for every tuple $`c`$, either $`c`$ is not a course, or $`s`$ is enrolled in $`c`$.*

| Student | C1? | C2? | All? |
|---|---|---|---|
| Asha | ✓ | ✓ | ✓ |
| Bala | ✓ | ✗ | ✗ |
| Chen | ✗ | ✓ | ✗ |

**Result: Asha.** RA: $`\pi_{sid,cid}(Enroll) \div \pi_{cid}(Course)`$, then join to get the name.

The same query **without $`\forall`$**, as "there is no course that $`s`$ is not enrolled in":

```math
\{\, s.name \mid Student(s) \wedge \neg(\exists c)\big(Course(c) \wedge \neg(\exists e)(Enroll(e) \wedge e.sid = s.sid \wedge e.cid = c.cid)\big) \,\}
```

(This double-negation form is exactly how "for all" is written in SQL with `NOT EXISTS … NOT EXISTS`.)

### Type 7 — Union using $`\vee`$

*Sids of students who are in CSE **or** enrolled in C1.*

```math
\{\, s.sid \mid Student(s) \wedge \big(s.dept = \text{'CSE'} \vee (\exists e)(Enroll(e) \wedge e.sid = s.sid \wedge e.cid = \text{'C1'})\big) \,\}
```

Result: 1 (CSE), 2 (C1), 3 (CSE) → **{1, 2, 3}**. RA: $`\pi_{sid}(\sigma_{dept='CSE'}(Student)) \cup \pi_{sid}(\sigma_{cid='C1'}(Enroll))`$.

### Type 8 — Self-join (two variables over the same relation)

*Pairs of different students in the same department.*

```math
\{\, s_1.name, s_2.name \mid Student(s_1) \wedge Student(s_2) \wedge s_1.dept = s_2.dept \wedge s_1.sid < s_2.sid \,\}
```

Result: **(Asha, Chen)**. (`s1.sid < s2.sid` avoids pairing a student with themself and listing each pair twice.)

### Type 9 — Intersection using $`\wedge`$ of two existentials

*Students enrolled in **both** C1 and C2.*

```math
\{\, s.name \mid Student(s) \wedge (\exists e_1)(Enroll(e_1) \wedge e_1.sid = s.sid \wedge e_1.cid = \text{'C1'}) \wedge (\exists e_2)(Enroll(e_2) \wedge e_2.sid = s.sid \wedge e_2.cid = \text{'C2'}) \,\}
```

**Result: Asha.**

### Type 10 — Empty result

*Courses nobody is enrolled in.*

```math
\{\, c.title \mid Course(c) \wedge \neg(\exists e)(Enroll(e) \wedge e.cid = c.cid) \,\}
```

Both C1 and C2 have enrollments, so the **result is ∅** (the empty set).

## 6.7 Example database 2: COMPANY schema (textbook queries)

Schema: EMPLOYEE(Fname, Minit, Lname, **Ssn**, Bdate, Address, Sex, Salary, Super_ssn, Dno), DEPARTMENT(Dname, **Dnumber**, Mgr_ssn, Mgr_start_date), PROJECT(Pname, **Pnumber**, Plocation, Dnum), WORKS_ON(**Essn, Pno**, Hours), DEPENDENT(**Essn, Dependent_name**, Sex, Bdate, Relationship).

**Q0.** Birth date and address of employee 'John B. Smith':

```math
\{\, t.Bdate, t.Address \mid EMPLOYEE(t) \wedge t.Fname = \text{'John'} \wedge t.Minit = \text{'B'} \wedge t.Lname = \text{'Smith'} \,\}
```

**Q1.** Name and address of all employees who work for the 'Research' department:

```math
\{\, t.Fname, t.Lname, t.Address \mid EMPLOYEE(t) \wedge (\exists d)(DEPARTMENT(d) \wedge d.Dname = \text{'Research'} \wedge d.Dnumber = t.Dno) \,\}
```

**Q2.** For every project located in 'Stafford': the project number, the controlling department number, and the manager's last name, birth date and address:

```math
\{\, p.Pnumber, p.Dnum, m.Lname, m.Bdate, m.Address \mid PROJECT(p) \wedge EMPLOYEE(m) \wedge p.Plocation = \text{'Stafford'} \wedge (\exists d)(DEPARTMENT(d) \wedge p.Dnum = d.Dnumber \wedge d.Mgr\_ssn = m.Ssn) \,\}
```

(Free variables $`p`$ and $`m`$ appear in the target list; $`d`$ is only used for the join, so it is bound.)

**Q8.** Each employee's name with the name of their immediate supervisor (self-join):

```math
\{\, e.Fname, e.Lname, s.Fname, s.Lname \mid EMPLOYEE(e) \wedge EMPLOYEE(s) \wedge e.Super\_ssn = s.Ssn \,\}
```

**Q3′.** Employees who work on **some** project controlled by department 5:

```math
\{\, e.Lname, e.Fname \mid EMPLOYEE(e) \wedge (\exists x)(\exists w)(PROJECT(x) \wedge WORKS\_ON(w) \wedge x.Dnum = 5 \wedge w.Essn = e.Ssn \wedge x.Pnumber = w.Pno) \,\}
```

**Q3.** Employees who work on **all** projects controlled by department 5 (universal quantifier):

```math
\{\, e.Lname, e.Fname \mid EMPLOYEE(e) \wedge (\forall x)\Big(\neg PROJECT(x) \vee \neg(x.Dnum = 5) \vee (\exists w)\big(WORKS\_ON(w) \wedge w.Essn = e.Ssn \wedge x.Pnumber = w.Pno\big)\Big) \,\}
```

Reading: for every tuple $`x`$, **either** it's not a project, **or** it's not controlled by dept 5, **or** employee $`e`$ works on it. (These are the three ways the implication "if $`x`$ is a dept-5 project then $`e`$ works on it" can be true.)

The same query using only $`\exists`$:

```math
\{\, e.Lname, e.Fname \mid EMPLOYEE(e) \wedge \neg(\exists x)\Big(PROJECT(x) \wedge x.Dnum = 5 \wedge \neg(\exists w)\big(WORKS\_ON(w) \wedge w.Essn = e.Ssn \wedge x.Pnumber = w.Pno\big)\Big) \,\}
```

**Q6.** Names of employees who have **no** dependents:

```math
\{\, e.Fname, e.Lname \mid EMPLOYEE(e) \wedge \neg(\exists d)(DEPENDENT(d) \wedge e.Ssn = d.Essn) \,\}
```

Equivalently, with $`\forall`$: $`\{ e.Fname, e.Lname \mid EMPLOYEE(e) \wedge (\forall d)(\neg DEPENDENT(d) \vee \neg(e.Ssn = d.Essn)) \}`$.

**Q7.** Names of managers who have **at least one** dependent:

```math
\{\, e.Fname, e.Lname \mid EMPLOYEE(e) \wedge (\exists d)(\exists \rho)(DEPARTMENT(d) \wedge DEPENDENT(\rho) \wedge e.Ssn = d.Mgr\_ssn \wedge \rho.Essn = e.Ssn) \,\}
```

## 6.8 Safe and unsafe expressions

An expression is **safe** if it is guaranteed to produce a **finite** result. More precisely, every value in the result must come from the **domain of the expression**: the values that appear in the relations it mentions, or constants in it.

| Expression | Safe? | Why |
|---|---|---|
| $`\{ t \mid EMPLOYEE(t) \wedge t.Salary > 50000 \}`$ | ✅ Safe | $`t`$ ranges over EMPLOYEE |
| $`\{ t \mid \neg EMPLOYEE(t) \}`$ | ❌ **Unsafe** | Every tuple in the universe that isn't an employee: infinitely many |
| $`\{ t \mid EMPLOYEE(t) \vee t.Salary > 50000 \}`$ | ❌ **Unsafe** | The second disjunct lets $`t`$ be *any* tuple with Salary > 50000, not only employees |
| $`\{ s.name \mid Student(s) \wedge \neg(\exists e)(Enroll(e) \wedge e.sid = s.sid) \}`$ | ✅ Safe | Negation is fine because $`s`$ is already restricted to Student |

**Rule of thumb:** every free variable must be **"anchored" by a positive range atom** such as $`R(t)`$, joined with $`\wedge`$ to the rest of the condition.

## 6.9 Mapping relational algebra ↔ TRC

| RA | TRC |
|---|---|
| $`\sigma_{c}(R)`$ | $`\{ t \mid R(t) \wedge c(t) \}`$ |
| $`\pi_{A,B}(R)`$ | $`\{ t.A, t.B \mid R(t) \}`$ |
| $`R \cup S`$ (compatible) | $`\{ t \mid R(t) \vee S(t) \}`$ |
| $`R \cap S`$ | $`\{ t \mid R(t) \wedge S(t) \}`$ |
| $`R - S`$ | $`\{ t \mid R(t) \wedge \neg S(t) \}`$ |
| $`R \times S`$ | $`\{ r, s \mid R(r) \wedge S(s) \}`$ |
| $`R \bowtie_{R.A = S.B} S`$ | $`\{ r, s \mid R(r) \wedge S(s) \wedge r.A = s.B \}`$ |
| $`R(A,B) \div S(B)`$ | $`\{ r.A \mid R(r) \wedge (\forall s)(\neg S(s) \vee (\exists u)(R(u) \wedge u.A = r.A \wedge u.B = s.B)) \}`$ |

**Expressive power (Codd's theorem):** **safe** TRC, **safe** DRC and **basic relational algebra** can express exactly the same queries. A language that can express every RA query is called **relationally complete**. SQL is relationally complete (and more, with aggregates and ordering).

## 6.10 TRC ↔ SQL

The SQL `SELECT–FROM–WHERE` block reads almost like TRC:

| TRC | SQL |
|---|---|
| Tuple variable with range $`EMPLOYEE(e)`$ | `FROM EMPLOYEE e` |
| Target list $`e.Fname, e.Lname`$ | `SELECT e.Fname, e.Lname` |
| Condition | `WHERE …` |
| $`(\exists d)(\dots)`$ | `EXISTS (SELECT * FROM … d WHERE …)` |
| $`\neg(\exists d)(\dots)`$ | `NOT EXISTS (…)` |
| $`(\forall x)(P)`$ | `NOT EXISTS (… WHERE NOT P)` |

**Q6 in SQL** (employees with no dependents):

```sql
SELECT e.Fname, e.Lname
FROM   EMPLOYEE e
WHERE  NOT EXISTS (SELECT * FROM DEPENDENT d WHERE d.Essn = e.Ssn);
```

**"Enrolled in all courses" in SQL** (Type 6 above):

```sql
SELECT s.name
FROM   Student s
WHERE  NOT EXISTS (SELECT * FROM Course c
                   WHERE NOT EXISTS (SELECT * FROM Enroll e
                                     WHERE e.sid = s.sid AND e.cid = c.cid));
```

## 6.11 Domain relational calculus (brief comparison)

In DRC, variables range over **single attribute values**, one variable per attribute:

```math
\{\, x_1, x_2, \dots, x_n \mid \text{COND}(x_1, \dots, x_n, x_{n+1}, \dots, x_{n+m}) \,\}
```

For example, the names of CSE students, with Student(sid, name, dept):

```math
\{\, n \mid (\exists i)(\exists d)(Student(i, n, d) \wedge d = \text{'CSE'}) \,\}
```

| | TRC | DRC |
|---|---|---|
| Variables range over | Whole tuples | Attribute values |
| Atom for membership | $`R(t)`$ | $`R(x_1, \dots, x_n)`$ |
| Practical language based on it | SQL | QBE (Query-By-Example) |

---

# 7. Quick Revision

### File operations and organisations
- **Record-at-a-time:** Open, Reset, Find, Read, FindNext, Delete, Modify, Insert, Close. **Set-at-a-time:** Scan, FindAll, Find n, FindOrdered, Reorganize.
- $`bfr = \lfloor B/R \rfloor`$, $`b = \lceil r/bfr \rceil`$. Unspanned wastes $`B - bfr \cdot R`$ per block.
- **Heap:** insert ≈ 2 accesses; search $`b/2`$ (avg) or $`b`$ (worst/absent); delete leaves holes or uses a deletion marker, so it needs reorganisation.
- **Ordered:** binary search $`\lceil \log_2 b \rceil`$ on the ordering field only; cheap ordered/range reads; **expensive insert/delete** (use an overflow file + periodic merge).

### Linear hashing
- No directory. Buckets are split in **round-robin order** using split pointer $`n`$, whichever bucket overflowed.
- Split bucket $`n`$ with $`h_{j+1}(K) = K \bmod 2^{j+1}M`$ into buckets $`n`$ and $`n + 2^j M`$, then $`n{+}{+}`$.
- Address: $`a = h_j(K)`$; **if $`a < n`$ then $`a = h_{j+1}(K)`$**.
- When $`n = 2^j M`$: $`n = 0`$, $`j{+}{+}`$.

### Dynamic multilevel indexing
- Static multilevel: $`fo = \lfloor B/R_i \rfloor`$, $`t = \lceil \log_{fo} r_1 \rceil`$, cost $`t + 1`$, but inserts/deletes are painful.
- **B-tree:** data pointers in every node, each key once, median moves up on a split.
- **B+-tree:** data pointers only in the leaves, leaves linked, higher fan-out → shallower. The leaf split **copies** up and the internal split **moves** up.
- B-tree order: $`pP + (p-1)(Pr+V) \le B`$. B+ internal: $`pP + (p-1)V \le B`$. B+ leaf: $`p_{leaf}(Pr+V) + P \le B`$.

### Tuple relational calculus
- $`\{ t \mid COND(t) \}`$: declarative. Atoms: $`R(t)`$, $`t.A \,\theta\, s.B`$, $`t.A \,\theta\, c`$.
- $`\exists`$ = "some" (join/semi-join); $`\neg\exists`$ = "none" (difference); $`\forall`$ = "all" (division).
- $`\forall x\, P \equiv \neg \exists x\, \neg P`$; $`P \Rightarrow Q \equiv \neg P \vee Q`$.
- Only **free** variables go left of $`\mid`$. Only **safe** expressions (finite results) are allowed.
- Safe TRC = safe DRC = relational algebra in expressive power (**relational completeness**).
