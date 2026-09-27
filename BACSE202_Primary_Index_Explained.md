# BACSE202 — Primary Index Explained

> Module 3 · Indexing. Based on the course slides (Module 3 main deck and *Types of Indexes in DBMS*) and Elmasri & Navathe, Ch. 17.

---

## 1. Definition

A **primary index** is an index built on the **ordering key field** of an **ordered (sorted) data file**.

For a primary index to exist, **both** of these must be true:

1. The data file is **physically sorted** on some field (the *ordering field*).
2. That field is a **key**, meaning its value is **unique** for every record (e.g. `RollNo`, `Ssn`, `StudentID`).

> If the file is sorted on a field that is **not unique** (e.g. `Department`), the index is a **clustering index**, not a primary index.
> If the index is on a field the file is **not sorted** by, it is a **secondary index**.

---

## 2. Structure of the index file

The index file is itself an **ordered file** of **fixed-length entries with two fields**:

```math
\langle\, K(i),\; P(i) \,\rangle
```

| Field | Contains |
|---|---|
| $`K(i)`$ | A **primary key value**: the key of the **first record in block $`i`$** of the data file |
| $`P(i)`$ | A **pointer to disk block $`i`$** of the data file |

- There is **one index entry per data block**, not one per record.
- The first record of each block is called the **anchor record** (or **block anchor**), and its key is what goes into the index.

---

## 3. Example

**Data file** sorted on `StudentID` (the primary key), 4 records per block:

```
   Index file                       Data file (sorted on StudentID)
 ┌────────┬───────┐
 │ Anchor │ Ptr   │
 ├────────┼───────┤
 │  1001  │  ●────┼──────►  Block 1: [1001 | 1002 | 1003 | 1004]
 │  1005  │  ●────┼──────►  Block 2: [1005 | 1006 | 1007 | 1008]
 │  1009  │  ●────┼──────►  Block 3: [1009 | 1010 | 1011 | 1012]
 │  1013  │  ●────┼──────►  Block 4: [1013 | 1014 |  …   |  …  ]
 └────────┴───────┘
```

Each index entry holds the **anchor** (first key) of one block: 1001, 1005, 1009, 1013.

**Textbook example** (file ordered on `Name`):

| Index entry (block anchor) | Data block contents |
|---|---|
| Aaron, Ed | Aaron, Ed · Abbot, Diane · … · Acosta, Marc |
| Adams, John | Adams, John · Adams, Robin · … · Akers, Jan |
| Alexander, Ed | Alexander, Ed · Alfred, Bob · … · Allen, Sam |
| Allen, Troy | Allen, Troy · Anders, Keith · … · Anderson, Rob |
| Anderson, Zach | Anderson, Zach · Angel, Joe · … · Archer, Sue |
| Arnold, Mack | Arnold, Mack · Arnold, Steven · … · Atkins, Timothy |

---

## 4. Dense or sparse?

A primary index in the textbook sense is **sparse (non-dense)**. It has entries for only **some** key values (one per block), not for every record.

| | Number of index entries |
|---|---|
| Data file | $`r`$ records in $`b`$ blocks |
| Primary index | $`b`$ entries (**one per block**) |
| Dense index | $`r`$ entries (one per record) |

Since $`b \ll r`$, the primary index is **much smaller** than the data file.

> The slides also show a primary index drawn as **dense** (one entry per key: 101 → Arun, 102 → Banu, …).
> - **Sparse primary index** = the classic textbook form: one entry per block.
> - **Dense primary index** = one entry per record.
>
> A sparse index is possible **only because the file is sorted**. Once you know the right block, you can find the record inside it.

---

## 5. How to search using a primary index

To find the record with key $`K`$:

1. **Binary search the index file** for the entry $`i`$ such that

```math
K(i) \le K < K(i+1)
```

   i.e. the **largest anchor ≤ K**.

2. Follow pointer $`P(i)`$ and read **that one data block**.
3. Search inside the block (in memory) for $`K`$.

### Worked search

Using the `StudentID` example, find **1011**:

| Step | Action |
|---|---|
| 1 | Index anchors: 1001, 1005, 1009, 1013. The largest anchor ≤ 1011 is **1009** |
| 2 | Follow its pointer → read **Block 3** |
| 3 | Block 3 = {1009, 1010, **1011**, 1012} → found |

Find **1016** (does not exist): the largest anchor ≤ 1016 is 1013 → read Block 4 → 1016 is not in it → **not found**. Note that a sparse index **cannot** say "not found" without reading the data block.

---

## 6. Cost calculation (the exam favourite)

### Given

- $`r = 300{,}000`$ records, fixed length $`R = 100`$ bytes, block size $`B = 4096`$ bytes
- Ordering key field $`V = 9`$ bytes, block pointer $`P = 6`$ bytes

### Step 1 — Data file

```math
bfr = \left\lfloor \frac{B}{R} \right\rfloor = \left\lfloor \frac{4096}{100} \right\rfloor = 40 \text{ records/block}
```

```math
b = \left\lceil \frac{r}{bfr} \right\rceil = \left\lceil \frac{300000}{40} \right\rceil = 7500 \text{ blocks}
```

**Without an index**, a binary search on the data file costs:

```math
\lceil \log_2 b \rceil = \lceil \log_2 7500 \rceil = 13 \text{ block accesses}
```

### Step 2 — Primary index

Index entry size:

```math
R_i = V + P = 9 + 6 = 15 \text{ bytes}
```

Index blocking factor (entries per index block):

```math
bfr_i = \left\lfloor \frac{4096}{15} \right\rfloor = 273
```

Number of index entries = number of data blocks = $`r_i = 7500`$, so the index needs:

```math
b_i = \left\lceil \frac{7500}{273} \right\rceil = 28 \text{ blocks}
```

**With the primary index:**

```math
\underbrace{\lceil \log_2 28 \rceil}_{\text{binary search on index}} + \underbrace{1}_{\text{data block}} = 5 + 1 = 6 \text{ block accesses}
```

### Step 3 — Multilevel version (bonus)

Build a second level on the 28 index blocks. The fan-out is $`fo = bfr_i = 273`$:

```math
\text{Level 2 blocks} = \left\lceil \frac{28}{273} \right\rceil = 1
```

That gives 2 index levels, so search cost $`= 2 + 1 = 3`$ block accesses.

### Summary

| Method | Block accesses |
|---|---|
| Linear search (average) | $`7500/2 = 3750`$ |
| Binary search on the sorted data file | **13** |
| Single-level primary index | **6** |
| Two-level primary index | **3** |

---

## 7. Insertion and deletion — the main disadvantage

Because the data file is **sorted** and the index stores **block anchors**:

- **Inserting** a record in its correct position may require **shifting records** into later blocks. That can **change the anchor records** of many blocks, so many **index entries change** too.
- **Deleting** a record has the same problem (records move, anchors change).

**Solutions:**

| Technique | Idea |
|---|---|
| **Unordered overflow file** | Put new records in a separate overflow area; merge periodically |
| **Linked list of overflow records** per block | Each block keeps a chain of its extra records, so anchors don't change |
| **Deletion markers** | Mark records deleted instead of physically removing them |
| **Use a B+-tree instead** | The dynamic multilevel index handles inserts/deletes locally (the modern solution) |

---

## 8. Advantages and disadvantages

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Very **small** index (one entry per block) → often fits in memory | Data file **must be sorted** on the key |
| Fast lookups: $`\log_2 b_i + 1`$ block accesses | Only **one** primary index per file (a file can be sorted only one way) |
| Efficient **range queries** (records are physically in order) | **Insert/delete are expensive** (records shift, anchors change) |
| Enforces/uses **uniqueness** of the key | Can't directly answer "does key K exist?" without reading the block (sparse) |

---

## 9. Primary vs. clustering vs. secondary index

| | Primary | Clustering | Secondary |
|---|---|---|---|
| File sorted on the indexed field? | **Yes** | **Yes** | **No** |
| Indexed field unique? | **Yes (key)** | **No** (non-key) | Either |
| Index entries | 1 per **block** | 1 per **distinct value** | 1 per **record** (or per value + bucket) |
| Dense / sparse | Sparse | Sparse | **Dense** |
| How many per file | At most 1 | At most 1 | Many |
| Example | `RollNo` in a file sorted by `RollNo` | `Department` in a file sorted by `Department` | `Name` in a file sorted by `RollNo` |

> A file can have **either** a primary index **or** a clustering index, never both, because it can only be physically sorted on one field. It can have **any number** of secondary indexes.

---

## 10. Primary index in SQL

Most DBMSs **automatically create an index** when a `PRIMARY KEY` is declared. It enforces uniqueness and makes point lookups fast:

```sql
CREATE TABLE Student (
    StudentID INT PRIMARY KEY,   -- primary index created automatically
    Name      VARCHAR(50),
    Age       INT
);

SELECT * FROM Student WHERE StudentID = 1011;               -- point lookup via the index
SELECT * FROM Student WHERE StudentID BETWEEN 1005 AND 1012; -- range query
```

> In real systems (e.g. MySQL InnoDB, SQL Server) the primary key index is usually a **clustered B+-tree**. That is the dynamic, multilevel version of the same idea: data ordered by the primary key, with an index on top.

---

## 11. Quick revision

- **Primary index** = index on the **ordering key field** of a **sorted file**.
- Entry = ⟨**block anchor key**, **block pointer**⟩, **one per data block** → **sparse**.
- Search: find the **largest anchor ≤ K**, read that block. Cost = $`\lceil \log_2 b_i \rceil + 1`$.
- Formulas: $`bfr_i = \lfloor B / (V + P) \rfloor`$ and $`b_i = \lceil b / bfr_i \rceil`$.
- Example: 300,000 records → 7,500 data blocks → 28 index blocks → **6** accesses (vs 13 without an index).
- Problem: inserts and deletes shift records and change anchors → use overflow files, or a B+-tree.
- Only **one** primary index per file. It can't coexist with a clustering index.
