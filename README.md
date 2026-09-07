# BACSE202 --- Database Systems

> A complete learning and practical repository for **BACSE202: Database
> Systems**.

This repository follows the **BACSE202 Database Systems syllabus,
Version 1.0**, covering database fundamentals, ER/EER modeling,
relational design and normalization, SQL, query processing and
optimization, transaction management, concurrency control, recovery,
distributed databases, NoSQL, PL/SQL, database connectivity, and
practical database applications.

**Course:** BACSE202 --- Database Systems\
**Credits:** 4\
**Lecture Hours:** 45\
**Laboratory Hours:** 30\
**Prerequisite:** NIL\
**Syllabus Version:** 1.0

------------------------------------------------------------------------

## 📚 Course Objectives

The course is designed to:

1.  Familiarize students with database concepts and Entity-Relationship
    modeling.
2.  Develop knowledge of relational database design, normalization,
    physical database design, query optimization, and distributed
    databases.
3.  Equip students with strategies for transaction processing,
    concurrency control, and database recovery.

------------------------------------------------------------------------

## 🎯 Course Outcomes

After completing the course, you should be able to:

-   **CO1:** Understand the role of DBMS in organizations and design
    relational data models and their operations.
-   **CO2:** Analyze and apply indexing structures for efficient data
    access and query optimization.
-   **CO3:** Understand transaction management, concurrency control, and
    database recovery mechanisms.
-   **CO4:** Demonstrate knowledge of distributed databases and NoSQL
    databases.
-   **CO5:** Apply database design principles to create scalable and
    maintainable data models for real-world systems.

------------------------------------------------------------------------

# 🧠 Theory Syllabus

## Module 1 --- Database Systems and Data Models

**7 hours**

### Topics

-   Introduction to database systems
-   Characteristics of the database approach
-   Advantages of using the DBMS approach
-   Actors on the scene
-   Classification of DBMS
-   Data Models
-   Schemas and Instances
-   Three-Schema Architecture
-   Database System Environment
-   Introduction to:
    -   Relational databases
    -   Semi-structured databases
    -   Graph databases
    -   Vector databases
    -   Time-series databases
    -   Distributed databases
    -   Unstructured databases
-   Relational model concepts
-   Characteristics of relations
-   Relational model constraints
-   Relational database schemas
-   Domain constraints
-   Key constraints
-   Constraints on NULL values
-   Entity integrity
-   Referential integrity
-   Foreign keys
-   Handling constraint violations

### Practical Focus

-   Database creation
-   Tables and schemas
-   Primary keys
-   Foreign keys
-   UNIQUE, NOT NULL, CHECK and DEFAULT constraints
-   Referential integrity
-   SQL data types
-   Basic SQL operations

------------------------------------------------------------------------

# Module 2 --- ER Modeling and Relational Database Design

**10 hours**

## Entity-Relationship Model

### Topics

-   Types of attributes
-   Relationship types and sets
-   Roles
-   Structural constraints
-   Weak entities
-   Mapping ER models to relational schemas

## Extended ER Model

-   Generalization
-   Specialization
-   Aggregation

## Relational Schema Design

-   Guidelines for relational schema design

## Functional Dependencies

-   Functional dependencies
-   Inference rules
-   Equivalence
-   Minimal cover
-   Axioms on functional dependencies

## Normalization

-   First Normal Form --- **1NF**
-   Second Normal Form --- **2NF**
-   Third Normal Form --- **3NF**
-   Boyce-Codd Normal Form --- **BCNF**
-   Multivalued dependencies
-   Fourth Normal Form --- **4NF**
-   Join dependencies
-   Fifth Normal Form --- **5NF**

### Practical Focus

-   ER diagrams
-   EER diagrams
-   Converting ER diagrams to relational tables
-   Identifying primary and foreign keys
-   Functional dependencies
-   Attribute closure
-   Candidate keys
-   Minimal cover
-   Normalization from 1NF → 2NF → 3NF
-   BCNF
-   MVD and 4NF
-   Join dependency and 5NF

------------------------------------------------------------------------

# Module 3 --- Physical Database Design and Query Processing

**9 hours**

## File Structures

-   Operations on files
-   Files of unordered records
-   Files of ordered records

## Hashing

-   Static hashing
-   Dynamic hashing

## Indexing

-   Single-level indexing
-   Multi-level indexing
-   Dynamic multi-level indexing
-   B+ tree indexing
-   LSM trees and variants

## Relational Algebra

-   Relational algebra operators
-   Translating SQL queries into relational algebra

## Tuple Relational Calculus

-   Tuple Relational Calculus concepts

## Query Optimization

-   Query trees
-   Heuristic optimization of query trees

### Practical Focus

-   Relational algebra operations
-   SQL-to-relational-algebra translation
-   Query trees
-   Indexing concepts
-   B+ tree concepts
-   Query optimization basics

------------------------------------------------------------------------

# Module 4 --- Transaction Processing, Concurrency Control and Database Recovery

**10 hours**

## Transaction Processing

-   Introduction to transaction processing
-   Transaction concepts
-   ACID properties
-   Transaction states

## Schedules

-   Serial schedules
-   Serializable schedules
-   Recoverable schedules
-   Schedules based on recoverability
-   Schedules based on serializability
-   Conflict serializability

## Concurrent Transactions

-   Concurrent transactions
-   Need for concurrency control

## Concurrency Control

### Two-Phase Locking

-   Two-phase locking techniques
-   Guaranteeing serializability using 2PL
-   Deadlock
-   Starvation

### Timestamp Ordering

-   Timestamp ordering
-   Timestamp ordering algorithm

### Locking Granularity

-   Granularity of data items
-   Multiple granularity locking

## Database Recovery

-   Recovery concepts
-   Categorization of recovery algorithms
-   Deferred update
-   Immediate update

### Practical / Conceptual Focus

-   ACID
-   Transaction states
-   Serial vs concurrent schedules
-   Conflict serializability
-   Precedence graphs
-   Two-phase locking
-   Deadlocks
-   Starvation
-   Timestamp ordering
-   Recovery techniques

------------------------------------------------------------------------

# Module 5 --- Distributed and NoSQL Databases

**7 hours**

## Distributed Databases

-   Introduction
-   Distributed database architectures
-   Data fragmentation
-   Data replication
-   Data allocation techniques
-   Transaction management
-   Concurrency control
-   Recovery
-   Query processing
-   Query optimization

## NoSQL Databases

-   Introduction to NoSQL
-   Characteristics of NoSQL systems
-   CAP theorem
-   ACID vs BASE
-   NoSQL modeling techniques
-   Aggregate data models
-   Distribution models
-   MapReduce

## Types of NoSQL Databases

### Key-Value Databases

Data represented as:

``` text
key → value
```

### Document Databases

Data represented as documents, commonly JSON-like structures.

### Column-Based Databases

Data organized around columns/column families.

### Graph Databases

Data represented using:

``` text
Nodes + Relationships + Properties
```

### Practical Focus

-   NoSQL CRUD operations
-   Document databases
-   Scaling concepts
-   Replication
-   Backup and restore
-   Data sharing
-   Distributed database concepts

------------------------------------------------------------------------

# Module 6 --- Contemporary Issues

**2 hours**

The syllabus lists **Contemporary Issues** as the sixth module.

------------------------------------------------------------------------

# 🧪 Laboratory Syllabus

The laboratory component contains **30 hours** of practical work.

## 1. ER / EER Diagram

**2 hours**

Design an ER/EER diagram for a real-world requirement such as a Bank
Database using a suitable drawing tool.

------------------------------------------------------------------------

## 2. ER to Relational Database

**2 hours**

Translate an ER diagram into the corresponding relational database
schema.

Key skills:

-   Entities → tables
-   Attributes → columns
-   Primary keys
-   Foreign keys
-   Relationships
-   Cardinality

------------------------------------------------------------------------

## 3. Normalization

**2 hours**

Normalize a given relation into:

``` text
1NF
 ↓
2NF
 ↓
3NF
```

Also understand BCNF and higher normal forms from the theory syllabus.

------------------------------------------------------------------------

## 4. DML, DCL and TCL

**2 hours**

Populate a relational database using real-world Customer and Transaction
data.

Practice:

### DML

``` sql
INSERT
UPDATE
DELETE
SELECT
```

### DCL

``` sql
GRANT
REVOKE
```

### TCL

``` sql
COMMIT
ROLLBACK
SAVEPOINT
```

------------------------------------------------------------------------

## 5. Operators and Retrieval

**2 hours**

Retrieve data using:

-   Arithmetic operators
-   Comparison operators
-   Logical operators
-   LIKE operator

Examples:

``` sql
=
<>
>
<
>=
<=
```

``` sql
AND
OR
NOT
```

``` sql
LIKE
```

------------------------------------------------------------------------

## 6. Single-Row and Group Functions

**2 hours**

Apply functions to normalized tables.

### Single-row functions

Examples include functions for:

-   Character manipulation
-   Number manipulation
-   Date manipulation
-   Conversion

### Group functions

Important examples:

``` sql
COUNT()
SUM()
AVG()
MAX()
MIN()
```

Practice with:

``` sql
GROUP BY
HAVING
```

------------------------------------------------------------------------

## 7. Subqueries, Joins and Views

**4 hours**

Retrieve data using:

-   Subqueries
-   Joins
-   Views

### Subqueries

Practice:

-   Single-row subqueries
-   Multiple-row subqueries
-   Nested subqueries

Important operators:

``` sql
IN
ANY
ALL
EXISTS
NOT EXISTS
```

### Joins

Practice:

-   INNER JOIN
-   LEFT OUTER JOIN
-   RIGHT OUTER JOIN
-   FULL OUTER JOIN
-   Self joins

### Views

Practice:

``` sql
CREATE VIEW
```

and retrieving data through views.

------------------------------------------------------------------------

## 8. PL/SQL

**6 hours**

Use PL/SQL features to retrieve or modify data.

### Topics

-   Control structures
-   Conditional statements
-   User-defined functions
-   Procedures
-   Cursors
-   Triggers

### Important PL/SQL Structures

``` text
Anonymous Block
     ↓
Variables
     ↓
Conditions
     ↓
Loops
     ↓
Functions / Procedures
     ↓
Cursors
     ↓
Triggers
```

------------------------------------------------------------------------

## 9. ODBC and JDBC

**2 hours**

Interface the database with front-end applications using:

-   ODBC
-   JDBC

Perform:

-   Data retrieval
-   DML operations

from a front-end application.

------------------------------------------------------------------------

## 10. NoSQL Database Operations

**2 hours**

Implement:

-   Database creation
-   Database deletion
-   Document insertion
-   Document updates
-   Document retrieval
-   Views where supported
-   Scale-in
-   Scale-out

------------------------------------------------------------------------

## 11. NoSQL Replication, Backup and Restore

**2 hours**

Implement and demonstrate:

-   Database replication
-   Backup
-   Restore
-   Data sharing in a NoSQL environment

------------------------------------------------------------------------

## 12. Cloud Database Application

**2 hours**

Create and configure cloud resources and:

-   Perform CRUD operations
-   Integrate a database with a sample application
-   Demonstrate real-world database usage

------------------------------------------------------------------------

# 🗺️ Recommended Learning Path

A good order for learning the course is:

``` text
1. Database Fundamentals
        ↓
2. Relational Model
        ↓
3. Constraints
        ↓
4. ER / EER Modeling
        ↓
5. ER → Relational Mapping
        ↓
6. Functional Dependencies
        ↓
7. Normalization
        ↓
8. SQL Basics
        ↓
9. SQL Operators & Functions
        ↓
10. Subqueries
        ↓
11. Joins
        ↓
12. Views
        ↓
13. PL/SQL
        ↓
14. Relational Algebra
        ↓
15. Query Processing & Optimization
        ↓
16. Transactions
        ↓
17. Concurrency Control
        ↓
18. Recovery
        ↓
19. Distributed Databases
        ↓
20. NoSQL
        ↓
21. Database Connectivity
        ↓
22. Cloud Database Applications
```

------------------------------------------------------------------------

# 🛠️ SQL / Oracle Practical Toolkit

The practical portion of this repository can be organized around the
following SQL areas.

## Database Definition

``` sql
CREATE TABLE
ALTER TABLE
DROP TABLE
```

## Constraints

``` sql
PRIMARY KEY
FOREIGN KEY
UNIQUE
NOT NULL
CHECK
DEFAULT
```

## Data Manipulation

``` sql
INSERT
UPDATE
DELETE
SELECT
```

## Data Control

``` sql
GRANT
REVOKE
```

## Transaction Control

``` sql
COMMIT
ROLLBACK
SAVEPOINT
```

## Retrieval

``` sql
WHERE
ORDER BY
GROUP BY
HAVING
```

## Operators

``` sql
Arithmetic
Comparison
Logical
LIKE
BETWEEN
IN
ANY
ALL
EXISTS
```

## Functions

``` sql
Single-row functions
Group functions
```

## Advanced Retrieval

``` sql
Subqueries
Joins
Views
```

## PL/SQL

``` text
Blocks
Variables
IF / CASE
Loops
Functions
Procedures
Cursors
Triggers
```

------------------------------------------------------------------------

# 📐 Database Design Concepts

The repository should maintain a clear distinction between these stages:

``` text
Real-World Requirement
        ↓
ER / EER Model
        ↓
Relational Schema
        ↓
Functional Dependencies
        ↓
Normalization
        ↓
SQL Implementation
        ↓
Data Population
        ↓
Queries
        ↓
Application Integration
```

This represents the progression from **database design to implementation
and application usage**.

------------------------------------------------------------------------

# 🔑 Core Concepts to Master

Before moving to advanced topics, make sure you can confidently explain:

### Database Fundamentals

-   DBMS
-   Database approach
-   Database users / actors
-   Data models
-   Schema vs instance
-   Three-schema architecture

### Relational Model

-   Relation
-   Tuple
-   Attribute
-   Domain
-   Keys
-   Constraints
-   Entity integrity
-   Referential integrity

### Database Design

-   Entity
-   Attribute
-   Relationship
-   Cardinality
-   Participation
-   Weak entity
-   Generalization
-   Specialization
-   Aggregation

### Normalization

-   Functional dependency
-   Candidate key
-   Prime attribute
-   Partial dependency
-   Transitive dependency
-   1NF
-   2NF
-   3NF
-   BCNF
-   Multivalued dependency
-   4NF
-   Join dependency
-   5NF

### SQL

-   DDL
-   DML
-   DCL
-   TCL
-   Constraints
-   Operators
-   Functions
-   Subqueries
-   Joins
-   Views

### PL/SQL

-   Blocks
-   Variables
-   Conditions
-   Loops
-   Functions
-   Procedures
-   Cursors
-   Triggers

### Advanced DBMS

-   Indexing
-   B+ trees
-   LSM trees
-   Relational algebra
-   Query optimization
-   Transactions
-   Serializability
-   2PL
-   Timestamp ordering
-   Deadlocks
-   Recovery
-   Distributed databases
-   CAP theorem
-   ACID vs BASE
-   NoSQL models

------------------------------------------------------------------------

# 📁 Suggested Repository Structure

A clean structure for this DBMS repository could be:

``` text
DBMS/
│
├── README.md
│
├── Module-1/
│   ├── notes/
│   ├── sql/
│   └── examples/
│
├── Module-2/
│   ├── ER-EER/
│   ├── normalization/
│   ├── functional-dependencies/
│   └── examples/
│
├── Module-3/
│   ├── file-structures/
│   ├── hashing/
│   ├── indexing/
│   ├── relational-algebra/
│   └── query-optimization/
│
├── Module-4/
│   ├── transactions/
│   ├── serializability/
│   ├── concurrency-control/
│   └── recovery/
│
├── Module-5/
│   ├── distributed-databases/
│   └── nosql/
│
├── Module-6/
│   └── contemporary-issues/
│
├── SQL/
│   ├── ddl/
│   ├── dml/
│   ├── dcl/
│   ├── tcl/
│   ├── constraints/
│   ├── operators/
│   ├── functions/
│   ├── subqueries/
│   ├── joins/
│   └── views/
│
├── PL-SQL/
│   ├── functions/
│   ├── procedures/
│   ├── cursors/
│   └── triggers/
│
├── Labs/
│   ├── Lab-01/
│   ├── Lab-02/
│   └── ...
│
└── Projects/
    └── ...
```

This is a **recommended organization**, not a structure prescribed by
the syllabus.

------------------------------------------------------------------------

# 🧪 Practical Checklist

Use this checklist before the lab exam:

## SQL

-   [ ] Create tables
-   [ ] Insert records
-   [ ] Update records
-   [ ] Delete records
-   [ ] Alter tables
-   [ ] Add constraints
-   [ ] Rename constraints
-   [ ] Drop constraints
-   [ ] Use primary keys
-   [ ] Use foreign keys
-   [ ] Use CHECK / UNIQUE / NOT NULL / DEFAULT
-   [ ] Use arithmetic operators
-   [ ] Use comparison operators
-   [ ] Use logical operators
-   [ ] Use LIKE
-   [ ] Use GROUP BY
-   [ ] Use HAVING
-   [ ] Use aggregate functions
-   [ ] Write single-row subqueries
-   [ ] Write multiple-row subqueries
-   [ ] Write nested subqueries
-   [ ] Write INNER JOIN
-   [ ] Write LEFT OUTER JOIN
-   [ ] Write RIGHT OUTER JOIN
-   [ ] Create and use views

## PL/SQL

-   [ ] Anonymous blocks
-   [ ] Variables
-   [ ] IF / ELSE
-   [ ] CASE
-   [ ] Loops
-   [ ] Functions
-   [ ] Procedures
-   [ ] Cursors
-   [ ] Triggers

## Database Design

-   [ ] ER diagram
-   [ ] EER diagram
-   [ ] ER → relational schema
-   [ ] Functional dependencies
-   [ ] Candidate keys
-   [ ] 1NF
-   [ ] 2NF
-   [ ] 3NF
-   [ ] BCNF
-   [ ] 4NF
-   [ ] 5NF

## Advanced DBMS

-   [ ] File organization
-   [ ] Hashing
-   [ ] Indexing
-   [ ] B+ trees
-   [ ] LSM trees
-   [ ] Relational algebra
-   [ ] Query trees
-   [ ] Query optimization
-   [ ] ACID
-   [ ] Serializability
-   [ ] 2PL
-   [ ] Deadlock
-   [ ] Timestamp ordering
-   [ ] Recovery
-   [ ] Distributed databases
-   [ ] CAP theorem
-   [ ] ACID vs BASE
-   [ ] NoSQL database models

------------------------------------------------------------------------

# 📚 Recommended Textbooks

The official syllabus lists:

### Primary Textbook

**R. Elmasri and S. B. Navathe**\
*Fundamentals of Database Systems*, Pearson India Education, **7th
Edition, 2021**

### Reference Books

1.  **A. Silberschatz, H. F. Korth and S. Sudarshan**\
    *Database System Concepts*, McGraw Hill India, 7th Edition, 2021

2.  **Mark L. Gillenson and Pradeep Singh**\
    *Fundamentals of Database Management Systems --- An Indian
    Adaptation*, Wiley India, 3rd Edition, 2025

3.  **M. Tamer Özsu and Patrick Valduriez**\
    *Principles of Distributed Database Systems*, Springer Nature, 4th
    Edition, 2020

4.  **Gerardus Blokdyk**\
    *NoSQL Databases A Complete Guide*, 1st Edition, 2021

------------------------------------------------------------------------

# 📝 Assessment

The syllabus specifies the following modes of evaluation:

-   Continuous Assessment Test
-   Digital Assignment
-   Quiz
-   Final Assessment Test
-   Lab Continuous Assessment
-   Lab Final Assessment

------------------------------------------------------------------------

# 📊 Course at a Glance

  Component                       Hours
  ---------------------------- --------
  Module 1                            7
  Module 2                           10
  Module 3                            9
  Module 4                           10
  Module 5                            7
  Module 6                            2
  **Total Lecture Hours**        **45**
  **Total Laboratory Hours**     **30**

------------------------------------------------------------------------

# 🚀 Goal of This Repository

This repository is intended to serve as a **single reference point for
BACSE202 Database Systems**.

It should contain:

-   Theory notes
-   SQL scripts
-   PL/SQL programs
-   ER/EER diagrams
-   Normalization examples
-   Lab exercises
-   Query practice
-   Transaction and concurrency examples
-   NoSQL experiments
-   Database projects
-   Exam revision material

The goal is to move from:

``` text
Understand
    ↓
Design
    ↓
Implement
    ↓
Query
    ↓
Optimize
    ↓
Manage Transactions
    ↓
Scale
    ↓
Build Applications
```

------------------------------------------------------------------------

# ⭐ Quick Revision Map

``` text
                 DATABASE SYSTEMS
                        │
        ┌───────────────┴───────────────┐
        │                               │
   DATABASE DESIGN                 DATABASE USAGE
        │                               │
   ER / EER Model                     SQL
        │                               │
   Relational Schema              DDL / DML
        │                          DCL / TCL
 Functional Dependencies                │
        │                         Functions
 Normalization                     Subqueries
 1NF → 2NF → 3NF                    Joins
        │                            Views
      BCNF                             │
        │                           PL/SQL
    4NF / 5NF                          │
        │                               │
        └───────────────┬───────────────┘
                        │
                 ADVANCED DBMS
                        │
       ┌────────────────┼────────────────┐
       │                │                │
   Indexing        Transactions       NoSQL
       │                │                │
 B+ Trees / LSM    Concurrency       CAP / BASE
       │            Recovery        Distribution
       │                │                │
       └────────────────┼────────────────┘
                        │
                  APPLICATIONS
                        │
                  JDBC / ODBC
                        │
                  Cloud Database
```

------------------------------------------------------------------------

## 📌 Source

This README is based on the official **BACSE202 Database Systems,
Syllabus Version 1.0** document, which specifies the course objectives,
outcomes, six modules, 45 lecture hours, 30 laboratory hours, textbooks,
indicative experiments, and evaluation methods.

> **BACSE202 --- Database Systems \| Syllabus Version 1.0**
