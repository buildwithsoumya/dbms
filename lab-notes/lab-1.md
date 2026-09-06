# Oracle SQL — Complete Query Guide (Week 1 Lab)

Based on your BACSE202 Introduction deck. This covers every SQL command category shown: **DDL, DML, DCL, TCL**, and **constraints** — with syntax and worked examples you can run directly in your Oracle 26ai Free SQL*Plus.

---

## 0. The Four SQL Sub-languages

| Type | Full Form | Purpose | Commands |
|---|---|---|---|
| **DDL** | Data Definition Language | Define/modify structure | CREATE, ALTER, DROP, TRUNCATE, RENAME |
| **DML** | Data Manipulation Language | Work with data | SELECT, INSERT, UPDATE, DELETE |
| **DCL** | Data Control Language | Control access | GRANT, REVOKE |
| **TCL** | Transaction Control Language | Manage transactions | COMMIT, ROLLBACK, SAVEPOINT |

---

## 1. DDL — Data Definition Language

### 1.1 CREATE TABLE

Creates a new relation (table) with named attributes and data types.

```sql
create table student(name varchar2(15), rollno number(8), dept varchar2(5), doj date);
```

Common data types you'll use: `VARCHAR2(n)` (variable-length text), `CHAR(n)` (fixed-length text), `NUMBER(p,q)` (p = total digits, q = decimal digits), `DATE`.

### 1.2 DESCRIBE (DESC)

Shows a table's structure — column names, nullability, and data types. Not technically DDL/DML, but essential.

```sql
SQL> desc student;

Name                       Null?    Type
-------------------------- -------- -------------
NAME                                VARCHAR2(15)
ROLLNO                              NUMBER(8)
DEPT                                VARCHAR2(5)
DOJ                                 DATE
```

### 1.3 ALTER TABLE

Modifies an existing table's structure. Four operations:

**Add a column:**
```sql
SQL> alter table student add regno number(15);
Table altered.
```

**Modify a column's data type or size:**
```sql
SQL> alter table student modify rollno varchar2(7);   -- change type
SQL> alter table student modify rollno varchar2(8);   -- change size
Table altered.
```

**Drop a column:**
```sql
SQL> alter table student drop column regno;
Table altered.
```

**Rename a column** (separate clause, same command family):
```sql
alter table table_name rename column old_name to new_name;
-- example:
alter table student rename column branch to course;
```

### 1.4 RENAME TABLE

```sql
alter table table_name rename to new_name;
-- example:
alter table student rename to students;
```

### 1.5 COPY A TABLE (structure + data)

```sql
create table table_name as select * from table_name;
```

### 1.6 TRUNCATE TABLE

Deletes **all rows** but keeps the table structure intact. Cannot be rolled back.

```sql
SQL> truncate table student;
Table truncated.

SQL> select * from student;
no rows selected
```

### 1.7 DROP TABLE

Deletes the table's data **and** its structure entirely.

```sql
SQL> drop table student;
Table dropped.

SQL> select * from student;
ERROR at line 1:
ORA-00942: table or view does not exist
```

**Quick memory aid:** `TRUNCATE` empties the house but keeps it standing; `DROP` demolishes the house.

---

## 2. DML — Data Manipulation Language

### 2.1 INSERT — three methods

Setup table used in the deck:
```sql
create table player(name varchar2(20), country varchar2(15), matches number(3), runs number(4), jersy number(2));
```

**Method 1 — single record, all columns in order:**
```sql
SQL> insert into player values('sachin','india',98,999,9);
1 row created.
```

**Method 2 — using substitution variables (`&`)** — SQL*Plus prompts you for each value, handy for repeated inserts:
```sql
SQL> insert into player values('&name','&country',&matches,&runs,&jersy);
Enter value for name: shewag
Enter value for country: india
Enter value for matches: 15
...
1 row created.
```
Typing `/` re-runs the previous statement, prompting again — useful for adding several rows quickly.

**Method 3 — specific columns only** (unspecified columns become NULL):
```sql
SQL> insert into player(name,country,matches) values('pathani','india',2);
1 row created.
```

### 2.2 SELECT — retrieving data

**All columns, all rows:**
```sql
select * from player;
```

**Specific columns (domain values):**
```sql
select name, country from player;
```

**Remove duplicates:**
```sql
select distinct country from player;
```

**Filter with WHERE:**
```sql
select * from player where matches >= 40;
select name, runs from player where runs >= 1000;
```

**Sort with ORDER BY:**
```sql
select * from player order by matches;             -- ascending by default
select * from player order by matches asc;
select * from player order by matches desc;
select * from player order by matches asc, country desc;   -- multi-column sort
```

**Pattern matching with LIKE** (`%` = any number of characters, `_` = exactly one character):
```sql
select * from player where name like 's%';      -- starts with 's'
select * from player where name like 'd___i';   -- 5-char name: 'd', 3 any chars, 'i'
select * from player where name like '%c%';     -- contains 'c' anywhere
```

**Rename output column headers (alias):**
```sql
select name player, country nation from player;
```

### 2.3 UPDATE

Modifies existing rows that match a condition.

```sql
SQL> update player set runs=10 where jersy=10;
SQL> update player set runs=10 where jersy is null;
3 rows updated.
```
> Note: use `IS NULL` / `IS NOT NULL` for null comparisons — `= NULL` never matches in SQL.

### 2.4 DELETE

Removes rows (structure stays).

```sql
delete from player where jersy is null;   -- conditional delete
delete from player;                        -- deletes all rows
select * from player;                       -- no rows selected
```

---

## 3. DCL — Data Control Language

Controls who can access what.

```sql
GRANT SELECT ON students TO user123;   -- give a privilege
REVOKE SELECT ON students FROM user123; -- take it back
```

- **GRANT** — give another user (or yourself, to another user) permission to access an object, or permission to grant further permissions.
- **REVOKE** — withdraw a previously granted privilege.

---

## 4. TCL — Transaction Control Language

Manages the "unit of work" around DML changes so data stays consistent.

```sql
COMMIT;                 -- makes all changes since the last commit permanent
ROLLBACK;                -- undoes changes since the last commit
SAVEPOINT sp1;            -- marks a point to roll back to, without undoing everything
SET TRANSACTION ...;     -- sets transaction properties
```

---

## 5. Constraints

Constraints enforce rules on the data a column/table can hold.

### 5.1 NOT NULL

```sql
create table acct(acctno number(5) not null, balance number(12,2) not null);
```
Inserting NULL into either column throws:
```
ORA-01400: cannot insert NULL into ("...")
```
Add to an existing table:
```sql
ALTER TABLE table_name MODIFY column_name datatype NOT NULL;
```

### 5.2 UNIQUE

```sql
create table studs1(regno number(12), name varchar2(10), city varchar2(7),
  constraint mynuique unique(name));
```
Inserting a duplicate name throws:
```
ORA-00001: unique constraint (...) violated
```
Add / drop on an existing table:
```sql
ALTER TABLE table_name ADD CONSTRAINT MyUniqueConstraint UNIQUE(column1, column2...);
ALTER TABLE table_name DROP CONSTRAINT MyUniqueConstraint;
```

### 5.3 DEFAULT

Supplies a value automatically when none is given.

```sql
CREATE TABLE CUSTOMERS(
  ID INT NOT NULL,
  NAME VARCHAR(20) NOT NULL,
  AGE INT NOT NULL,
  ADDRESS CHAR(25),
  SALARY DECIMAL(18,2) DEFAULT 5000.00,
  PRIMARY KEY (ID)
);
```
Add to an existing table:
```sql
ALTER TABLE CUSTOMERS MODIFY SALARY DECIMAL(18,2) DEFAULT 5000.00;
```

### 5.4 CHECK

Restricts a column to a set of valid values or conditions.

```sql
create table stu(name varchar2(10) not null, studid varchar2(5),
  degreelevel varchar2(15), primary key(studid),
  check(degreelevel in ('Bachelors','Masters','Doctorate')));
```
A value outside the list, e.g. `'Mastres'` (typo), throws:
```
ORA-02290: check constraint (...) violated
```

### 5.5 PRIMARY KEY

Uniquely identifies each row; implicitly `NOT NULL` + `UNIQUE`.

```sql
-- inline
create table student(regno number(5) primary key, name varchar2(15), rollno number(8), dept varchar2(5), doj date);

-- as a separate clause
create table student(regno number(5), name varchar2(15), rollno number(8), dept varchar2(5), doj date, primary key(regno));
```
Add / drop on an existing table:
```sql
alter table table_name add constraint pk primary key(attribute_name);
alter table table_name drop primary key;   -- or: drop constraint pk;
```

### 5.6 FOREIGN KEY

Links a column to a primary key in another table, enforcing referential integrity.

```sql
CREATE TABLE CUSTOMERS(
  ID INT NOT NULL,
  NAME VARCHAR(20) NOT NULL,
  AGE INT NOT NULL,
  ADDRESS CHAR(25),
  SALARY DECIMAL(18,2),
  PRIMARY KEY (ID)
);

CREATE TABLE ORDERS (
  ID INT NOT NULL,
  DATE_ DATETIME,
  CUSTOMER_ID INT REFERENCES CUSTOMERS(ID),
  AMOUNT DOUBLE,
  PRIMARY KEY (ID)
);
```
`CUSTOMER_ID` in `ORDERS` can only hold values that already exist as an `ID` in `CUSTOMERS`.

### 5.7 General constraint syntax (add/drop on existing tables)

```sql
alter table table_name add constraint constraint_name constraint_type(attribute_name);
alter table table_name drop constraint constraint_name;
```

---

## 6. SQL*Plus environment commands (not SQL itself, but you'll use these constantly)

| Command | What it does |
|---|---|
| `CONNECT username/password@dbname` | Log in to a database |
| `EXIT` / `QUIT` | Leave SQL*Plus |
| `SHOW USER` | Show current logged-in user |
| `SET LINESIZE 100` | Characters per output line |
| `SET PAGESIZE 50` | Lines per page |
| `SET HEADING OFF/ON` | Hide/show column headers |
| `SET FEEDBACK OFF/ON` | Hide/show "n rows selected" |
| `COLUMN col_name FORMAT A20` | Fix a column's display width |

---

## Quick self-test

Try these against a table you create yourself, in order:
1. `CREATE` a table with 4 columns, one `PRIMARY KEY`.
2. `INSERT` 3 rows (try all three insert methods).
3. `SELECT` with a `WHERE` and an `ORDER BY`.
4. `UPDATE` one row based on a condition.
5. `ALTER` the table to add a `NOT NULL` column.
6. `DELETE` one row, then `ROLLBACK` before committing — confirm it comes back.
