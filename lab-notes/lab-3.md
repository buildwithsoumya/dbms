# Oracle SQL — Constraints & Referential Integrity (Deep Dive)

Based on your **SQL Integrity Constraints Manual** and **Exercise 3**. This covers every constraint type, how to declare them two different ways, how to inspect and modify them, and how foreign keys behave when parent data is deleted.

---

## 1. What is a constraint?

A **constraint** is a rule enforced on a column (or combination of columns) that protects the accuracy and integrity of your data. Oracle checks every `INSERT`/`UPDATE`/`DELETE` against active constraints and rejects anything that breaks the rule.

| Constraint | Allows NULL? | Allows Duplicates? | Purpose |
|---|---|---|---|
| **PRIMARY KEY** | ✗ No | ✗ No | Uniquely identifies each row |
| **FOREIGN KEY** | ✓ Yes | ✓ Yes | Links to a Primary Key in another table |
| **UNIQUE** | ✓ Yes | ✗ No | No duplicate values, but NULL is fine |
| **NOT NULL** | ✗ No | ✓ Yes | Value is mandatory, duplicates fine |
| **CHECK** | — | — | Restricts values by a condition |
| **DEFAULT** | — | — | Auto-fills a value if none supplied |

Constraints can be written in two styles: **column-level** or **table-level**.

---

## 2. Column-level constraints

Written directly after the column's data type — reads naturally, but you can only reference **one** column per constraint this way (except a multi-column PK/UNIQUE, which needs table-level).

```sql
CREATE TABLE EMPLOYEE
(
  emp_id  NUMBER PRIMARY KEY,
  name    VARCHAR2(20) NOT NULL,
  email   VARCHAR2(30) UNIQUE,
  gender  VARCHAR2(10) CHECK(gender IN ('m','f'))
);
```

**Test it:**
```sql
INSERT INTO EMPLOYEE VALUES(1,'Smith','smith@gmail.com','m');
-- 1 row created.

INSERT INTO EMPLOYEE VALUES(1,'John','john@gmail.com','m');
-- ERROR: unique constraint (...) violated
-- Fails because emp_id=1 already exists — PRIMARY KEY forbids duplicates.
```

---

## 3. Named vs unnamed constraints

If you don't name a constraint, Oracle auto-generates a system name like `SYS_C003208` — which tells you nothing when an error occurs. Naming your constraints makes errors self-explanatory.

**Unnamed (system-generated name):**
```sql
CREATE TABLE STUDENT(
  StudentID   NUMBER PRIMARY KEY,
  StudentName VARCHAR2(30) NOT NULL,
  PhoneNumber VARCHAR2(10) UNIQUE,
  Grade       CHAR(1) CHECK(Grade IN ('A','B','C','D'))
);
```
Inserting a duplicate `StudentID` gives an unhelpful message:
```
ORA-00001: unique constraint (SCHEMA.SYS_C0032xx) violated
```

**Named, using `CONSTRAINT constraint_name`:**
```sql
CREATE TABLE STUDENT(
  StudentID   NUMBER          CONSTRAINT STUD_PK       PRIMARY KEY,
  StudentName VARCHAR2(30)    CONSTRAINT STUD_NAME_NN  NOT NULL,
  PhoneNumber VARCHAR2(10)    CONSTRAINT STUD_PHONE_UK UNIQUE,
  Grade       CHAR(1)         CONSTRAINT STUD_GRADE_CHK CHECK(Grade IN ('A','B','C','D'))
);
```
Now the same violation reads:
```
ORA-00001: unique constraint (SCHEMA.STUD_PK) violated
```
You immediately know it's the primary key, without looking anything up — much easier to debug and to reference later when dropping the constraint.

---

## 4. Table-level constraints

Written as a separate line **after** all the columns, instead of attached to one column. This is the only way to build a constraint that spans multiple columns (composite key), and some people prefer it for readability.

```sql
CREATE TABLE EMPLOYEE
(
  emp_id  NUMBER,
  name    VARCHAR2(20) CONSTRAINT const_NN NOT NULL,   -- NOT NULL stays column-level
  email   VARCHAR2(30),
  gender  VARCHAR2(10),
  PRIMARY KEY(emp_id),
  UNIQUE(email),
  CHECK(gender IN ('m','f'))
);
```

> **Rule of thumb from the manual:** `NOT NULL` is always written at the column level (it has to attach to one specific column) — `PRIMARY KEY`, `UNIQUE`, `CHECK`, and `FOREIGN KEY` can go either way.

---

## 5. Each constraint, tested in detail

Using the `EMPLOYEE` table from Section 3 (named constraints):

### PRIMARY KEY — rejects duplicates and NULLs
```sql
INSERT INTO EMPLOYEE VALUES(1,'Smith','smith@gmail.com','m');   -- OK
INSERT INTO EMPLOYEE VALUES(1,'John','john@gmail.com','m');     -- ORA-00001, CONST_PK
INSERT INTO EMPLOYEE VALUES(NULL,'Peter','peter@gmail.com','m'); -- ORA-01400: cannot insert NULL
```

### NOT NULL — rejects missing values, allows duplicate values
```sql
INSERT INTO EMPLOYEE VALUES(3,NULL,'peter@gmail.com','m');   -- fails: name is NOT NULL
INSERT INTO EMPLOYEE VALUES(3,'John','peter@gmail.com','m'); -- succeeds — 'John' already
                                                                -- exists as a name, but
                                                                -- NOT NULL only blocks NULL,
                                                                -- not duplicates
```

### UNIQUE — rejects duplicate values, allows NULL
```sql
INSERT INTO EMPLOYEE VALUES(4,'Steven','peter@gmail.com','m'); -- fails: email already used
INSERT INTO EMPLOYEE VALUES(4,'Steven',NULL,'m');               -- succeeds — UNIQUE permits NULL
```
This is the key difference from PRIMARY KEY: **UNIQUE tolerates NULL, PRIMARY KEY never does.**

### CHECK — rejects values outside the allowed condition
```sql
INSERT INTO EMPLOYEE VALUES(5,'William','william@gmail.com','k'); -- fails: 'k' not in ('m','f')
```

### DEFAULT — fills a value automatically when omitted
```sql
ALTER TABLE DEPARTMENT ADD experience NUMBER DEFAULT 0;

INSERT INTO DEPARTMENT(dept_id,dept_name) VALUES(1,'Admin');
-- experience column is automatically set to 0
```

---

## 6. Viewing constraints — the data dictionary

Oracle records every constraint in a system view called `USER_CONSTRAINTS`. Query it instead of guessing:

```sql
SELECT * FROM USER_CONSTRAINTS;                              -- everything you own

SELECT * FROM USER_CONSTRAINTS WHERE TABLE_NAME = 'EMPLOYEE'; -- just one table
```
This shows you the constraint name, type (`P` = Primary Key, `U` = Unique, `C` = Check/Not Null, `R` = Foreign Key/References), and status.

---

## 7. Adding/modifying constraints on an existing table (ALTER TABLE)

You don't have to get everything right at `CREATE TABLE` time.

**Add a PRIMARY KEY:**
```sql
ALTER TABLE DEPARTMENT
ADD CONSTRAINT cons_dept_id_PK
PRIMARY KEY(dept_id);
```

**Add NOT NULL to an existing column (via MODIFY):**
```sql
ALTER TABLE DEPARTMENT
MODIFY dept_name
CONSTRAINT cons_dept_name_NN
NOT NULL;
```

**Drop any constraint by name:**
```sql
ALTER TABLE table_name
DROP CONSTRAINT constraint_name;

-- example from the exercise:
ALTER TABLE STUDENT DROP CONSTRAINT STUD_PHONE_UK;
```
After dropping the `UNIQUE` constraint on `PhoneNumber`, duplicate phone numbers are accepted again — the constraint no longer exists to block them.

---

## 8. FOREIGN KEY and Referential Integrity

A **Foreign Key** links a **child table** column to a **Parent table's Primary Key**, enforcing that every value in the child either:
- matches an existing value in the parent, **or**
- is `NULL` (if the column allows it).

```sql
CREATE TABLE DEPARTMENT(
  dept_id   NUMBER PRIMARY KEY,
  dept_name VARCHAR2(30) NOT NULL
);

CREATE TABLE EMPLOYEE(
  emp_id  NUMBER PRIMARY KEY,
  name    VARCHAR2(30),
  dept_id NUMBER,
  CONSTRAINT emp_fk FOREIGN KEY(dept_id) REFERENCES DEPARTMENT(dept_id)
);
```

**Behavior tested in the manual:**
```sql
INSERT INTO EMPLOYEE VALUES(5,'William','william@gmail.com','m',5);
-- ORA-02291: integrity constraint violated - parent key not found
-- (dept_id 5 doesn't exist in DEPARTMENT)

INSERT INTO EMPLOYEE VALUES(5,'William','william@gmail.com','m',NULL);
-- succeeds — NULL is always allowed in a foreign key column

INSERT INTO EMPLOYEE VALUES(6,'Nicolas','nicolas@gmail.com','m',3);
-- succeeds — duplicate foreign key values are fine; many employees
-- can belong to the same department
```

### What happens when you try to delete a referenced parent row?

By default, Oracle blocks it — this is the safest option and Oracle's default.

**Case 1 — Default behavior: NO ACTION / RESTRICT**
```sql
DELETE FROM DEPARTMENT WHERE dept_id = 1;
-- ORA-02292: integrity constraint violated - child record found
```
Oracle refuses to delete a parent row while child rows still point to it — this protects you from leaving "orphaned" foreign key values.

**Case 2 — `ON DELETE SET NULL`**

Declare it when creating the foreign key:
```sql
CONSTRAINT emp_fk FOREIGN KEY(dept_id) REFERENCES DEPARTMENT(dept_id) ON DELETE SET NULL
```
Now deleting the parent doesn't fail — instead:
```
Before:  101  Smith  1
DELETE FROM DEPARTMENT WHERE dept_id = 1;
After:   101  Smith  NULL
```
Only the foreign key column becomes NULL; the employee row itself stays.

**Case 3 — `ON DELETE CASCADE`**
```sql
CONSTRAINT emp_fk FOREIGN KEY(dept_id) REFERENCES DEPARTMENT(dept_id) ON DELETE CASCADE
```
```sql
DELETE FROM DEPARTMENT WHERE dept_id = 1;
```
Now every employee row with `dept_id = 1` is deleted automatically along with the department — no manual cleanup needed, but be careful: this deletes data, not just a reference.

### Referential actions — what Oracle actually supports

| Action | Effect on DELETE | Effect on UPDATE | Oracle support |
|---|---|---|---|
| **NO ACTION / RESTRICT** (default) | Blocks the delete if children exist | Blocks the update | ✓ Yes |
| **CASCADE** | Deletes matching child rows | Updates child FK values | Delete only |
| **SET NULL** | Sets child FK to NULL | Sets child FK to NULL | Delete only |
| **SET DEFAULT** | Sets child FK to its default value | Sets child FK to its default | ✗ Not supported |

> Oracle only lets you customize `ON DELETE` behavior (`CASCADE` or `SET NULL`) — there is **no `ON UPDATE CASCADE`/`SET NULL`/`SET DEFAULT`** in Oracle, unlike MySQL, PostgreSQL, or SQL Server which support all of these. If you need cascading updates in Oracle, you'd handle it with a trigger instead.

---

## 9. Worked lab example — STUDENT + COURSE (from Exercise 3)

Putting it all together, this mirrors your exercise's flow:

```sql
-- Parent table
CREATE TABLE COURSE(
  CourseID   NUMBER,
  CourseName VARCHAR2(30),
  Credit     NUMBER
);

ALTER TABLE COURSE ADD CONSTRAINT course_pk PRIMARY KEY(CourseID);
ALTER TABLE COURSE MODIFY CourseName NOT NULL;
ALTER TABLE COURSE ADD CONSTRAINT credit_default DEFAULT 3;  -- (conceptually — see note below)

INSERT INTO COURSE(CourseID,CourseName) VALUES(101,'Database Systems');
-- Credit defaults to 3 since it wasn't supplied

-- Child table with foreign key
ALTER TABLE STUDENT ADD CourseID NUMBER;
ALTER TABLE STUDENT ADD CONSTRAINT student_course_fk
  FOREIGN KEY(CourseID) REFERENCES COURSE(CourseID);

INSERT INTO STUDENT VALUES(104,'Anand','9876543220','A',110);
-- ORA-02291: parent key not found — CourseID 110 doesn't exist in COURSE

INSERT INTO STUDENT VALUES(104,'Anand','9876543220','A',NULL);
-- succeeds — NULL is always permitted in a FK column

INSERT INTO STUDENT VALUES(105,'Deepa','9876543221','B',101);
-- succeeds — duplicate FK values (multiple students, same course) are fine
```

> **Note:** the actual Oracle syntax for adding a `DEFAULT` to an existing column is via `MODIFY`, not `ADD CONSTRAINT`:
> ```sql
> ALTER TABLE COURSE MODIFY Credit NUMBER DEFAULT 3;
> ```

---

## Quick self-test

1. Create a parent `DEPARTMENT` table and a child `EMPLOYEE` table with a named foreign key.
2. Insert 2 departments and 2 employees, one per department.
3. Try deleting a department with an employee still assigned — confirm you get `ORA-02292`.
4. Recreate the FK with `ON DELETE SET NULL`, delete the department again, and check the employee's `dept_id` becomes NULL.
5. Recreate the FK once more with `ON DELETE CASCADE`, delete the department, and confirm the employee row itself disappears.
6. Query `USER_CONSTRAINTS WHERE TABLE_NAME = 'EMPLOYEE'` after each step to see the constraint name and type change.
