# BACSE202 --- SQL Integrity Constraints: Complete Practical Guide

## 1. What is a Constraint?

A **constraint** is a rule enforced by the database to control what data
can be stored.

The BACSE202 Exercise 3 specifically covers:

-   PRIMARY KEY
-   FOREIGN KEY
-   UNIQUE
-   NOT NULL
-   CHECK
-   DEFAULT
-   Column-level and table-level constraints
-   Named and unnamed constraints
-   ALTER TABLE
-   Referential integrity
-   ON DELETE SET NULL
-   ON DELETE CASCADE

Source: the uploaded **Lab-3_25BCE1983** exercise, whose aim explicitly
lists these topics.

------------------------------------------------------------------------

## 2. The Six Main Constraints

  Constraint      Purpose
  --------------- ------------------------------------------
  `PRIMARY KEY`   Uniquely identifies every row
  `FOREIGN KEY`   Connects a child table to a parent table
  `UNIQUE`        Prevents duplicate values
  `NOT NULL`      Makes a value mandatory
  `CHECK`         Restricts values using a condition
  `DEFAULT`       Supplies a value when one is omitted

### Memory trick

``` text
PRIMARY KEY → Who is this row?
FOREIGN KEY → Which parent does it belong to?
UNIQUE      → No duplicates
NOT NULL    → Must have a value
CHECK       → Must satisfy a condition
DEFAULT     → Use this if nothing is supplied
```

------------------------------------------------------------------------

# 3. PRIMARY KEY

A primary key uniquely identifies every row.

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER PRIMARY KEY,
    StudentName VARCHAR2(30)
);
```

A primary-key value must be:

-   unique
-   NOT NULL

### Valid

``` text
101 Rahul
102 Priya
103 Karthik
```

### Invalid

``` text
101 Rahul
101 Arun
```

because `101` is already present.

This is also invalid:

``` text
NULL Deepa
```

because a primary key cannot be NULL.

### Lab example

The named primary-key constraint is:

``` sql
StudentID NUMBER CONSTRAINT STUD_PK PRIMARY KEY
```

So the constraint name is:

``` text
STUD_PK
```

The lab demonstrates duplicate and NULL primary-key inserts. A duplicate
produces a unique-constraint violation, while NULL violates the
primary-key requirement.

------------------------------------------------------------------------

# 4. UNIQUE Constraint

`UNIQUE` prevents duplicate values.

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER PRIMARY KEY,
    PhoneNumber VARCHAR2(10) UNIQUE
);
```

This is valid:

``` text
9876543210
9876543211
9876543212
```

This is not:

``` text
9876543210
9876543210
```

## PRIMARY KEY vs UNIQUE

  PRIMARY KEY                     UNIQUE
  ------------------------------- ---------------------------------------
  Identifies each row             Prevents duplicate values
  Cannot contain NULL             NULL is allowed in Oracle
  One primary key per table       Multiple UNIQUE constraints can exist
  Implies uniqueness + non-null   Mainly enforces uniqueness

### Important lab example

This fails if `9876543210` already exists:

``` sql
INSERT INTO STUDENT
VALUES(104,'Ramesh','9876543210','A');
```

But this succeeds under the lab's UNIQUE constraint:

``` sql
INSERT INTO STUDENT
VALUES(105,'Suresh',NULL,'B');
```

Why?

Because Oracle allows NULL in a normal UNIQUE column.

If the column must be both unique and mandatory:

``` sql
PhoneNumber VARCHAR2(10) UNIQUE NOT NULL
```

------------------------------------------------------------------------

# 5. NOT NULL

`NOT NULL` means a value is mandatory.

``` sql
StudentName VARCHAR2(30) NOT NULL
```

Therefore:

``` sql
INSERT INTO STUDENT
VALUES(104,NULL,'9876543216','A');
```

fails.

Use NOT NULL for fields that must always be supplied, such as a required
name or course name.

------------------------------------------------------------------------

# 6. CHECK Constraint

`CHECK` restricts values according to a condition.

The lab requires:

``` text
Grade must be A, B, C or D
```

So:

``` sql
Grade CHAR(1)
    CHECK (Grade IN ('A','B','C','D'))
```

Valid:

``` text
A
B
C
D
```

Invalid:

``` text
E
F
X
```

Therefore:

``` sql
INSERT INTO STUDENT
VALUES(106,'Mohan','9876543217','E');
```

fails.

Typical Oracle error:

``` text
ORA-02290: check constraint (...) violated
```

Other examples:

``` sql
Age NUMBER CHECK (Age >= 18)

Salary NUMBER CHECK (Salary > 0)

Gender CHAR(1) CHECK (Gender IN ('M','F'))

Credit NUMBER CHECK (Credit BETWEEN 1 AND 6)
```

------------------------------------------------------------------------

# 7. DEFAULT Constraint

`DEFAULT` supplies a value when the user does not provide one.

The lab uses:

``` text
Credit = 3
```

Example:

``` sql
CREATE TABLE COURSE (
    CourseID NUMBER,
    CourseName VARCHAR2(30),
    Credit NUMBER DEFAULT 3
);
```

Then:

``` sql
INSERT INTO COURSE(COURSEID,COURSENAME)
VALUES(101,'Database Systems');
```

results in:

``` text
101 | Database Systems | 3
```

But if you explicitly provide a value:

``` sql
INSERT INTO COURSE(COURSEID,COURSENAME,CREDIT)
VALUES(102,'Operating Systems',4);
```

the value is `4`, not `3`.

### Lab ALTER TABLE version

``` sql
ALTER TABLE COURSE
MODIFY Credit NUMBER DEFAULT 3;
```

------------------------------------------------------------------------

# 8. FOREIGN KEY

A foreign key creates a relationship between two tables.

Consider:

``` text
COURSE
----------------
CourseID
CourseName
Credit
```

and:

``` text
STUDENT
----------------
StudentID
StudentName
CourseID
```

The relationship is:

``` text
STUDENT.CourseID
       ↓
COURSE.CourseID
```

Therefore:

``` text
COURSE  = Parent
STUDENT = Child
```

The child stores a reference to the parent's key.

------------------------------------------------------------------------

# 9. Creating a Foreign Key

Parent:

``` sql
CREATE TABLE COURSE (
    CourseID NUMBER PRIMARY KEY,
    CourseName VARCHAR2(30),
    Credit NUMBER
);
```

Child:

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER PRIMARY KEY,
    StudentName VARCHAR2(30),
    CourseID NUMBER,

    CONSTRAINT STUD_COURSE_FK
        FOREIGN KEY (CourseID)
        REFERENCES COURSE(CourseID)
);
```

Read the syntax as:

``` text
FOREIGN KEY (child column)
REFERENCES parent table(parent column)
```

------------------------------------------------------------------------

# 10. Foreign-Key Example

Suppose COURSE contains:

``` text
101 Database Systems
102 Operating Systems
103 Computer Networks
```

Then:

``` sql
INSERT INTO STUDENT
VALUES(104,'Anand','9876543220','A',101);
```

is valid because course `101` exists.

But:

``` sql
INSERT INTO STUDENT
VALUES(104,'Anand','9876543220','A',110);
```

fails because course `110` does not exist.

Typical error:

``` text
ORA-02291: integrity constraint (...) violated -
parent key not found
```

### Exam memory

``` text
ORA-02291
     ↓
Parent key not found
     ↓
Foreign-key reference problem
```

------------------------------------------------------------------------

# 11. Can a Foreign Key be NULL?

Yes, unless it is also declared NOT NULL.

This is valid:

``` sql
INSERT INTO STUDENT
VALUES(104,'Anand','9876543220','A',NULL);
```

The lab specifically asks why NULL is accepted.

The reason is that NULL means no course is currently specified; the FK
constraint does not require a value unless NOT NULL is also imposed.

------------------------------------------------------------------------

# 12. Can a Foreign Key Repeat?

Yes.

Suppose:

``` text
COURSE
101
102
```

Students can be:

``` text
Rahul    101
Priya    101
Karthik  101
Anand    102
```

This is completely valid.

Therefore:

``` text
PRIMARY KEY → unique
FOREIGN KEY → may repeat
```

This is a very common exam question.

------------------------------------------------------------------------

# 13. Referential Integrity

Referential integrity keeps parent-child relationships valid.

If:

``` sql
STUDENT.CourseID
REFERENCES COURSE.CourseID
```

then a student cannot normally reference a course that does not exist.

Invalid:

``` text
COURSE
101
102

STUDENT
StudentID  CourseID
1          101
2          999   ← invalid
```

The foreign key prevents the second row.

------------------------------------------------------------------------

# 14. Column-Level Constraints

A constraint is column-level when it appears beside the column
definition.

Example:

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER PRIMARY KEY,
    StudentName VARCHAR2(30) NOT NULL,
    PhoneNumber VARCHAR2(10) UNIQUE,
    Grade CHAR(1)
        CHECK (Grade IN ('A','B','C','D'))
);
```

Each constraint is attached directly to a column.

------------------------------------------------------------------------

# 15. Named Column-Level Constraints

The lab uses:

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER
        CONSTRAINT STUD_PK PRIMARY KEY,

    StudentName VARCHAR2(30)
        CONSTRAINT STUD_NAME_NN NOT NULL,

    PhoneNumber VARCHAR2(10)
        CONSTRAINT STUD_PHONE_UK UNIQUE,

    Grade CHAR(1)
        CONSTRAINT STUD_GRADE_CHK
        CHECK (Grade IN ('A','B','C','D'))
);
```

Constraint names:

``` text
STUD_PK
STUD_NAME_NN
STUD_PHONE_UK
STUD_GRADE_CHK
```

------------------------------------------------------------------------

# 16. Why Name Constraints?

Named constraints are easier to identify, modify, and drop.

For example:

``` sql
ALTER TABLE STUDENT
DROP CONSTRAINT STUD_PHONE_UK;
```

If you do not name a constraint, Oracle may generate a system name such
as:

``` text
SYS_C008329
```

Compare:

``` text
STUD_PHONE_UK
```

with:

``` text
SYS_C008329
```

The first is much easier to understand.

Useful naming convention:

``` text
_PK   → Primary Key
_UK   → Unique Key
_NN   → Not Null
_CHK  → Check
_FK   → Foreign Key
```

------------------------------------------------------------------------

# 17. Table-Level Constraints

A table-level constraint is written separately from the column
definitions.

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER,
    StudentName VARCHAR2(30) NOT NULL,
    PhoneNumber VARCHAR2(10),
    Grade CHAR(1),

    CONSTRAINT STUD_PK PRIMARY KEY (StudentID),
    CONSTRAINT STUD_PHONE_UK UNIQUE (PhoneNumber),
    CONSTRAINT STUD_GRADE_CHK
        CHECK (Grade IN ('A','B','C','D'))
);
```

Compare:

### Column-level

``` sql
StudentID NUMBER PRIMARY KEY
```

### Table-level

``` sql
StudentID NUMBER,

CONSTRAINT STUD_PK PRIMARY KEY (StudentID)
```

The lab specifically asks you to recreate STUDENT using table-level
PRIMARY KEY, UNIQUE and CHECK constraints while keeping NOT NULL at
column level.

------------------------------------------------------------------------

# 18. Column-Level vs Table-Level

  -----------------------------------------------------------------------
  Column-Level                        Table-Level
  ----------------------------------- -----------------------------------
  Written beside a column             Written separately

  Convenient for simple constraints   Useful for more complex constraints

  Very readable for basic rules       Better for multi-column constraints

  Example:                            Example:
  `Age NUMBER CHECK (Age >= 18)`      `CONSTRAINT PK PRIMARY KEY (A,B)`
  -----------------------------------------------------------------------

A composite key is a classic table-level example:

``` sql
CREATE TABLE ENROLLMENT (
    StudentID NUMBER,
    CourseID NUMBER,

    CONSTRAINT ENROLL_PK
        PRIMARY KEY (StudentID, CourseID)
);
```

Here the **combination** must be unique.

------------------------------------------------------------------------

# 19. Named vs Unnamed Constraints

### Unnamed

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER PRIMARY KEY
);
```

Oracle generates a system constraint name.

### Named

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER
        CONSTRAINT STUD_PK PRIMARY KEY
);
```

Now the constraint has the meaningful name:

``` text
STUD_PK
```

The lab asks you to compare error messages from named and unnamed
constraints.

------------------------------------------------------------------------

# 20. ALTER TABLE --- Add Primary Key

You can create a table first and add its constraint later.

``` sql
CREATE TABLE COURSE (
    CourseID NUMBER,
    CourseName VARCHAR2(30),
    Credit NUMBER
);
```

Then:

``` sql
ALTER TABLE COURSE
ADD CONSTRAINT COURSE_PK PRIMARY KEY (CourseID);
```

------------------------------------------------------------------------

# 21. ALTER TABLE --- Add Foreign Key

The lab uses:

``` sql
ALTER TABLE STUDENT
ADD CONSTRAINT STUD_COURSE_FK
FOREIGN KEY (CourseID)
REFERENCES COURSE(CourseID);
```

Break it down:

``` text
ALTER TABLE STUDENT
        ↓
target table

ADD CONSTRAINT STUD_COURSE_FK
        ↓
constraint name

FOREIGN KEY (CourseID)
        ↓
child column

REFERENCES COURSE(CourseID)
        ↓
parent table + parent key
```

------------------------------------------------------------------------

# 22. ALTER TABLE --- NOT NULL

The lab uses:

``` sql
ALTER TABLE COURSE
MODIFY CourseName NOT NULL;
```

Now this fails:

``` sql
INSERT INTO COURSE(CourseID, Credit)
VALUES(107, 3);
```

because `CourseName` is mandatory.

------------------------------------------------------------------------

# 23. ALTER TABLE --- DEFAULT

The lab uses:

``` sql
ALTER TABLE COURSE
MODIFY Credit NUMBER DEFAULT 3;
```

Then:

``` sql
INSERT INTO COURSE(CourseID, CourseName)
VALUES(101, 'Database Systems');
```

automatically uses:

``` text
Credit = 3
```

------------------------------------------------------------------------

# 24. ALTER TABLE --- Drop Constraint

If the UNIQUE constraint is named:

``` text
STUD_PHONE_UK
```

drop it with:

``` sql
ALTER TABLE STUDENT
DROP CONSTRAINT STUD_PHONE_UK;
```

After this, duplicate phone values are allowed.

The lab then demonstrates that:

``` sql
INSERT INTO STUDENT
VALUES(106,'Rakesh','9876543210','A',102);
```

can succeed after the UNIQUE constraint has been dropped.

------------------------------------------------------------------------

# 25. Verify Constraints with USER_CONSTRAINTS

The lab asks you to execute:

``` sql
SELECT *
FROM USER_CONSTRAINTS
WHERE TABLE_NAME='COURSE';
```

This lets you inspect the constraints defined on the table.

Oracle stores normal unquoted table names in uppercase, so:

``` sql
WHERE TABLE_NAME='COURSE'
```

is the usual form.

Useful constraint type codes include:

  Type   Meaning
  ------ -------------------------------------
  `P`    Primary key
  `U`    Unique
  `R`    Referential integrity / Foreign key
  `C`    Check

------------------------------------------------------------------------

# 26. Parent and Child --- Never Confuse Them

For:

``` sql
STUDENT.CourseID
REFERENCES COURSE.CourseID
```

remember:

``` text
COURSE  = PARENT
STUDENT = CHILD
```

Why?

Because:

``` text
COURSE.CourseID
```

is the referenced key.

``` text
STUDENT.CourseID
```

is the referencing key.

Memory:

``` text
Parent = referenced table
Child  = referencing table
```

------------------------------------------------------------------------

# 27. Default Foreign-Key Behavior --- NO ACTION

Suppose:

``` text
COURSE
101 Database Systems

STUDENT
105 Deepa  CourseID=101
```

Now:

``` sql
DELETE FROM COURSE
WHERE COURSEID=101;
```

The deletion fails because a child row still references course 101.

Typical error:

``` text
ORA-02292: integrity constraint (...) violated -
child record found
```

Meaning:

> You tried to delete a parent row that is still referenced by a child.

------------------------------------------------------------------------

# 28. Why Does NO ACTION Prevent the Delete?

Before:

``` text
COURSE 101
    ↑
    |
STUDENT CourseID=101
```

If the parent disappeared while the child still contained 101:

``` text
COURSE
(no 101)

STUDENT
CourseID=101
```

the reference would become invalid.

Therefore Oracle protects referential integrity.

Memory:

``` text
NO ACTION → Don't delete a referenced parent
```

------------------------------------------------------------------------

# 29. ON DELETE SET NULL

Sometimes you want to delete the parent but keep the child.

Use:

``` sql
FOREIGN KEY (CourseID)
REFERENCES COURSE(CourseID)
ON DELETE SET NULL
```

Suppose:

``` text
COURSE
101 Database Systems

STUDENT
105 Deepa  CourseID=101
```

Run:

``` sql
DELETE FROM COURSE
WHERE COURSEID=101;
```

After deletion:

``` text
COURSE
101 → deleted

STUDENT
105 Deepa  CourseID=NULL
```

The student remains.

Only the foreign-key value becomes NULL.

------------------------------------------------------------------------

# 30. Important SET NULL Requirement

`ON DELETE SET NULL` means:

``` text
parent deleted
      ↓
child FK becomes NULL
```

Therefore the child FK must be able to contain NULL.

For example:

``` sql
CourseID NUMBER
```

works.

But:

``` sql
CourseID NUMBER NOT NULL
```

would conflict with the operation because the database cannot set it to
NULL.

------------------------------------------------------------------------

# 31. Recreate the Foreign Key with SET NULL

The lab drops the existing FK:

``` sql
ALTER TABLE STUDENT
DROP CONSTRAINT STUD_COURSE_FK;
```

Then recreates it:

``` sql
ALTER TABLE STUDENT
ADD CONSTRAINT STUD_COURSE_FK
FOREIGN KEY (CourseID)
REFERENCES COURSE(CourseID)
ON DELETE SET NULL;
```

Then:

``` sql
DELETE FROM COURSE
WHERE COURSEID=101;
```

The affected student rows remain and their `CourseID` becomes NULL.

------------------------------------------------------------------------

# 32. ON DELETE CASCADE

`ON DELETE CASCADE` means:

> Delete the parent and automatically delete matching child rows.

Syntax:

``` sql
FOREIGN KEY (CourseID)
REFERENCES COURSE(CourseID)
ON DELETE CASCADE
```

Suppose:

``` text
COURSE
102 Operating Systems

STUDENT
106 Rakesh  CourseID=102
```

Then:

``` sql
DELETE FROM COURSE
WHERE COURSEID=102;
```

causes both the course and referencing student row to be deleted.

------------------------------------------------------------------------

# 33. SET NULL vs CASCADE vs NO ACTION

Memorize this table:

  Behavior               Parent deletion         Child row   Child FK
  ---------------------- ----------------------- ----------- ----------------
  Default / NO ACTION    Blocked if referenced   Remains     Remains
  `ON DELETE SET NULL`   Allowed                 Remains     Becomes NULL
  `ON DELETE CASCADE`    Allowed                 Deleted     Row disappears

### Memory trick

``` text
NO ACTION → ERROR
SET NULL  → KEEP CHILD
CASCADE   → DELETE CHILD
```

------------------------------------------------------------------------

# 34. Complete Practical Example

## COURSE

``` sql
CREATE TABLE COURSE (
    CourseID NUMBER,
    CourseName VARCHAR2(30),
    Credit NUMBER
);

ALTER TABLE COURSE
ADD CONSTRAINT COURSE_PK PRIMARY KEY (CourseID);

ALTER TABLE COURSE
MODIFY CourseName NOT NULL;

ALTER TABLE COURSE
MODIFY Credit NUMBER DEFAULT 3;
```

Insert:

``` sql
INSERT INTO COURSE(COURSEID,COURSENAME)
VALUES(101,'Database Systems');

INSERT INTO COURSE(COURSEID,COURSENAME,CREDIT)
VALUES(102,'Operating Systems',4);

INSERT INTO COURSE(COURSEID,COURSENAME)
VALUES(103,'Computer Networks');

INSERT INTO COURSE VALUES(104,'Data Mining',4);
INSERT INTO COURSE VALUES(105,'Machine Learning',3);
INSERT INTO COURSE VALUES(106,'Cloud Computing',2);
```

------------------------------------------------------------------------

# 35. STUDENT with Major Constraints

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER
        CONSTRAINT STUD_PK PRIMARY KEY,

    StudentName VARCHAR2(30)
        CONSTRAINT STUD_NAME_NN NOT NULL,

    PhoneNumber VARCHAR2(10)
        CONSTRAINT STUD_PHONE_UK UNIQUE,

    Grade CHAR(1)
        CONSTRAINT STUD_GRADE_CHK
        CHECK (Grade IN ('A','B','C','D')),

    CourseID NUMBER,

    CONSTRAINT STUD_COURSE_FK
        FOREIGN KEY (CourseID)
        REFERENCES COURSE(CourseID)
);
```

This one table demonstrates:

``` text
PRIMARY KEY
NOT NULL
UNIQUE
CHECK
FOREIGN KEY
```

The COURSE table demonstrates:

``` text
PRIMARY KEY
NOT NULL
DEFAULT
```

------------------------------------------------------------------------

# 36. Valid and Invalid Data

### Valid

``` sql
INSERT INTO STUDENT
VALUES(104,'Anand','9876543220','A',NULL);
```

### Valid

``` sql
INSERT INTO STUDENT
VALUES(105,'Deepa','9876543221','B',101);
```

### Invalid --- duplicate primary key

``` sql
INSERT INTO STUDENT
VALUES(101,'Arun','9876543213','A',101);
```

### Invalid --- NULL primary key

``` sql
INSERT INTO STUDENT
VALUES(NULL,'Deepa','9876543214','B',101);
```

### Invalid --- duplicate UNIQUE value

``` sql
INSERT INTO STUDENT
VALUES(106,'Rakesh','9876543210','A',102);
```

if that phone already exists and the UNIQUE constraint is active.

### Invalid --- CHECK

``` sql
INSERT INTO STUDENT
VALUES(107,'Mohan','9876543225','E',101);
```

### Invalid --- foreign key

``` sql
INSERT INTO STUDENT
VALUES(108,'Arun','9876543226','A',110);
```

if course 110 does not exist.

------------------------------------------------------------------------

# 37. Important Oracle Error Codes

  Error         Think
  ------------- ------------------------
  `ORA-00001`   Duplicate value
  `ORA-02290`   CHECK condition failed
  `ORA-02291`   Parent key not found
  `ORA-02292`   Child record exists

### ORA-00001

Usually:

``` text
PRIMARY KEY or UNIQUE violation
```

### ORA-02290

``` text
CHECK constraint violated
```

### ORA-02291

``` text
Foreign key points to a missing parent
```

### ORA-02292

``` text
Trying to delete a referenced parent
```

------------------------------------------------------------------------

# 38. Lab Exercise 3 --- Question Map

## Q1 --- Create STUDENT

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER,
    StudentName VARCHAR2(30),
    PhoneNumber VARCHAR2(10),
    Grade CHAR(1)
);
```

## Q2 --- Column-Level Constraints

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER PRIMARY KEY,
    StudentName VARCHAR2(30) NOT NULL,
    PhoneNumber VARCHAR2(10) UNIQUE,
    Grade CHAR(1)
        CHECK (Grade IN ('A','B','C','D'))
);
```

## Q3 --- Primary Key

Duplicate:

``` sql
INSERT INTO STUDENT
VALUES(101,'Arun','9876543213','A');
```

NULL:

``` sql
INSERT INTO STUDENT
VALUES(NULL,'Deepa','9876543214','B');
```

## Q4 --- Named Constraints

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER CONSTRAINT STUD_PK PRIMARY KEY,
    StudentName VARCHAR2(30) CONSTRAINT STUD_NAME_NN NOT NULL,
    PhoneNumber VARCHAR2(10) CONSTRAINT STUD_PHONE_UK UNIQUE,
    Grade CHAR(1)
        CONSTRAINT STUD_GRADE_CHK
        CHECK (Grade IN ('A','B','C','D'))
);
```

## Q5 --- NOT NULL

``` sql
INSERT INTO STUDENT
VALUES(104,NULL,'9876543216','A');
```

## Q6 --- UNIQUE

``` sql
INSERT INTO STUDENT
VALUES(104,'Ramesh','9876543210','A');

INSERT INTO STUDENT
VALUES(105,'Suresh',NULL,'B');
```

## Q7 --- CHECK

``` sql
INSERT INTO STUDENT
VALUES(106,'Mohan','9876543217','E');
```

## Q8 --- Table-Level Constraints

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER,
    StudentName VARCHAR2(30) NOT NULL,
    PhoneNumber VARCHAR2(10),
    Grade CHAR(1),

    CONSTRAINT STUD_PK PRIMARY KEY (StudentID),
    CONSTRAINT STUD_PHONE_UK UNIQUE (PhoneNumber),
    CONSTRAINT STUD_GRADE_CHK
        CHECK (Grade IN ('A','B','C','D'))
);
```

## Q9 --- COURSE

``` sql
CREATE TABLE COURSE (
    CourseID NUMBER,
    CourseName VARCHAR2(30),
    Credit NUMBER
);
```

## Q10 --- ALTER TABLE

``` sql
ALTER TABLE COURSE
ADD CONSTRAINT COURSE_PK PRIMARY KEY (CourseID);

ALTER TABLE COURSE
MODIFY CourseName NOT NULL;
```

Verify:

``` sql
SELECT *
FROM USER_CONSTRAINTS
WHERE TABLE_NAME='COURSE';
```

## Q11 --- DEFAULT

``` sql
ALTER TABLE COURSE
MODIFY Credit NUMBER DEFAULT 3;
```

Then:

``` sql
INSERT INTO COURSE(COURSEID,COURSENAME)
VALUES(101,'Database Systems');
```

Credit becomes 3.

## Q12 --- Additional Records

``` sql
INSERT INTO COURSE VALUES(104,'Data Mining',4);
INSERT INTO COURSE VALUES(105,'Machine Learning',3);
INSERT INTO COURSE VALUES(106,'Cloud Computing',2);
```

## Q13 --- FOREIGN KEY

``` sql
ALTER TABLE STUDENT
ADD CourseID NUMBER;

ALTER TABLE STUDENT
ADD CONSTRAINT STUD_COURSE_FK
FOREIGN KEY (CourseID)
REFERENCES COURSE(CourseID);
```

Invalid:

``` sql
INSERT INTO STUDENT
VALUES(104,'Anand','9876543220','A',110);
```

Valid:

``` sql
INSERT INTO STUDENT
VALUES(104,'Anand','9876543220','A',NULL);
```

Valid:

``` sql
INSERT INTO STUDENT
VALUES(105,'Deepa','9876543221','B',101);
```

## Q14 --- Drop UNIQUE

``` sql
ALTER TABLE STUDENT
DROP CONSTRAINT STUD_PHONE_UK;
```

Now duplicate phone values can be inserted.

## Q15 --- Referential Integrity

### Default

``` sql
DELETE FROM COURSE
WHERE COURSEID=101;
```

Fails if a student references 101.

### SET NULL

``` sql
ALTER TABLE STUDENT
DROP CONSTRAINT STUD_COURSE_FK;

ALTER TABLE STUDENT
ADD CONSTRAINT STUD_COURSE_FK
FOREIGN KEY (CourseID)
REFERENCES COURSE(CourseID)
ON DELETE SET NULL;
```

Then:

``` sql
DELETE FROM COURSE
WHERE COURSEID=101;
```

The student remains, but its CourseID becomes NULL.

### CASCADE

``` sql
ALTER TABLE STUDENT
DROP CONSTRAINT STUD_COURSE_FK;

ALTER TABLE STUDENT
ADD CONSTRAINT STUD_COURSE_FK
FOREIGN KEY (CourseID)
REFERENCES COURSE(CourseID)
ON DELETE CASCADE;
```

Then:

``` sql
DELETE FROM COURSE
WHERE COURSEID=102;
```

Students referencing course 102 are automatically deleted.

------------------------------------------------------------------------

# 39. Constraint Decision Tree

When the question says:

``` text
"Every row must have a unique identity"
        ↓
PRIMARY KEY

"No duplicate values"
        ↓
UNIQUE

"Value must be provided"
        ↓
NOT NULL

"Value must satisfy a condition"
        ↓
CHECK

"Use 3 if no value is supplied"
        ↓
DEFAULT

"Value must exist in another table"
        ↓
FOREIGN KEY
```

For deletion:

``` text
"Do not allow parent deletion if children exist"
        ↓
DEFAULT / NO ACTION

"Delete parent and keep child but remove reference"
        ↓
ON DELETE SET NULL

"Delete parent and children together"
        ↓
ON DELETE CASCADE
```

------------------------------------------------------------------------

# 40. Common Practical-Exam Mistakes

### Mistake 1 --- Thinking UNIQUE and PRIMARY KEY are identical

Remember:

``` text
PRIMARY KEY = unique + NOT NULL
UNIQUE      = duplicate prevention; NULL allowed in Oracle
```

### Mistake 2 --- Thinking foreign keys must be unique

They can repeat.

### Mistake 3 --- Thinking foreign keys cannot be NULL

They can, unless NOT NULL is added.

### Mistake 4 --- Reversing parent and child

For:

``` sql
STUDENT.CourseID
REFERENCES COURSE.CourseID
```

the answer is:

``` text
STUDENT = child
COURSE = parent
```

### Mistake 5 --- Confusing SET NULL and CASCADE

``` text
SET NULL → child stays
CASCADE  → child disappears
```

### Mistake 6 --- Forgetting the parent row

Before inserting:

``` sql
CourseID = 110
```

check whether:

``` sql
SELECT *
FROM COURSE
WHERE CourseID=110;
```

exists.

### Mistake 7 --- Dropping the wrong constraint

To change the foreign-key behavior, drop:

``` sql
ALTER TABLE STUDENT
DROP CONSTRAINT STUD_COURSE_FK;
```

not the COURSE primary key.

------------------------------------------------------------------------

# 41. 30-Second Revision

``` text
PK  = unique + NOT NULL
UK  = no duplicate non-NULL values; NULL allowed in Oracle
NN  = mandatory
CHK = condition
DEF = automatic value if omitted
FK  = child → parent

NO ACTION  = block referenced parent deletion
SET NULL   = child survives, FK becomes NULL
CASCADE    = child is deleted

ORA-00001 = duplicate
ORA-02290 = CHECK failed
ORA-02291 = parent not found
ORA-02292 = child exists
```

## Final Mental Model

``` text
             COURSE
        +----------------+
        | CourseID (PK)  |
        | CourseName     |
        | Credit DEFAULT3|
        +----------------+
                ↑
                │
                │ FOREIGN KEY
                │
        +----------------+
        | STUDENT        |
        | StudentID (PK) |
        | StudentName NN |
        | PhoneNumber UK |
        | Grade CHECK     |
        | CourseID FK    |
        +----------------+
```

Think of constraints as the database's **rules of admission**:

``` text
PRIMARY KEY → "You need a unique identity."
UNIQUE      → "Don't duplicate this."
NOT NULL    → "You must provide this."
CHECK       → "Your value must obey this rule."
DEFAULT     → "I'll provide a value if you don't."
FOREIGN KEY → "Your reference must point somewhere valid."
```

This is the complete conceptual and practical map of the constraint
topics covered by the uploaded BACSE202 Exercise 3 material.
