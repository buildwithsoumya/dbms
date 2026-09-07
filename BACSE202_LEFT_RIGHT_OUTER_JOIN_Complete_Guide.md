# BACSE202 DBMS --- LEFT OUTER JOIN & RIGHT OUTER JOIN

## Complete Practical Guide with Proper Examples

## 1. What is a JOIN?

A **JOIN** combines rows from two or more tables using a related column.

Example:

### STUDENT

    StudentID StudentName     CourseID
  ----------- ------------- ----------
          101 Rahul                  1
          102 Priya                  2
          103 Karthik                3
          104 Anjali                 5

### COURSE

    CourseID CourseName
  ---------- ------------
           1 DBMS
           2 Java
           3 Python
           4 AI

The common relationship is:

``` sql
STUDENT.CourseID = COURSE.CourseID
```

------------------------------------------------------------------------

# 2. Why do we need OUTER JOINs?

A normal **INNER JOIN** returns only matching rows.

Here:

-   Anjali has `CourseID = 5`, but Course 5 does not exist.
-   AI has `CourseID = 4`, but no student is enrolled in it.

An OUTER JOIN lets us keep unmatched rows.

The two joins covered here are:

1.  LEFT OUTER JOIN
2.  RIGHT OUTER JOIN

------------------------------------------------------------------------

# 3. LEFT OUTER JOIN

A **LEFT OUTER JOIN** returns:

> **All rows from the LEFT table + matching rows from the RIGHT table.**

If a left-table row has no match, the right-table columns become `NULL`.

### Syntax

``` sql
SELECT columns
FROM table1
LEFT OUTER JOIN table2
ON table1.column = table2.column;
```

`LEFT JOIN` is the short form of `LEFT OUTER JOIN`.

------------------------------------------------------------------------

# 4. LEFT JOIN Example

``` sql
SELECT s.StudentID,
       s.StudentName,
       c.CourseName
FROM STUDENT s
LEFT OUTER JOIN COURSE c
ON s.CourseID = c.CourseID;
```

### Result

    StudentID StudentName   CourseName
  ----------- ------------- ------------
          101 Rahul         DBMS
          102 Priya         Java
          103 Karthik       Python
          104 Anjali        NULL

Why is Anjali included?

Because `STUDENT` is the **left table**:

``` text
STUDENT s
LEFT JOIN
COURSE c
```

A LEFT JOIN must keep every STUDENT row.

Anjali's `CourseID = 5` has no match, so the COURSE columns become
`NULL`.

### Golden rule

> **LEFT JOIN keeps ALL rows of the table on the LEFT side of JOIN.**

------------------------------------------------------------------------

# 5. RIGHT OUTER JOIN

A **RIGHT OUTER JOIN** returns:

> **All rows from the RIGHT table + matching rows from the LEFT table.**

If a right-table row has no match, the left-table columns become `NULL`.

### Syntax

``` sql
SELECT columns
FROM table1
RIGHT OUTER JOIN table2
ON table1.column = table2.column;
```

`RIGHT JOIN` is the short form of `RIGHT OUTER JOIN`.

------------------------------------------------------------------------

# 6. RIGHT JOIN Example

``` sql
SELECT s.StudentID,
       s.StudentName,
       c.CourseName
FROM STUDENT s
RIGHT OUTER JOIN COURSE c
ON s.CourseID = c.CourseID;
```

### Result

    StudentID StudentName   CourseName
  ----------- ------------- ------------
          101 Rahul         DBMS
          102 Priya         Java
          103 Karthik       Python
         NULL NULL          AI

Why is AI included?

Because `COURSE` is the **right table**:

``` text
STUDENT s
RIGHT JOIN
COURSE c
```

A RIGHT JOIN must keep every COURSE row.

AI has no matching student, so the STUDENT columns become `NULL`.

### Golden rule

> **RIGHT JOIN keeps ALL rows of the table on the RIGHT side of JOIN.**

------------------------------------------------------------------------

# 7. LEFT vs RIGHT --- The Easiest Way to Remember

Look at the word before `JOIN`.

### LEFT JOIN

``` sql
FROM STUDENT
LEFT JOIN COURSE
```

Keep everything from:

``` text
STUDENT
```

### RIGHT JOIN

``` sql
FROM STUDENT
RIGHT JOIN COURSE
```

Keep everything from:

``` text
COURSE
```

Remember:

> **LEFT = keep left table**
>
> **RIGHT = keep right table**

------------------------------------------------------------------------

# 8. INNER vs LEFT vs RIGHT

### INNER JOIN

Only matching rows:

``` sql
SELECT s.StudentName, c.CourseName
FROM STUDENT s
INNER JOIN COURSE c
ON s.CourseID = c.CourseID;
```

  StudentName   CourseName
  ------------- ------------
  Rahul         DBMS
  Priya         Java
  Karthik       Python

Both unmatched rows disappear.

------------------------------------------------------------------------

### LEFT JOIN

All students + matching courses:

``` sql
SELECT s.StudentName, c.CourseName
FROM STUDENT s
LEFT JOIN COURSE c
ON s.CourseID = c.CourseID;
```

  StudentName   CourseName
  ------------- ------------
  Rahul         DBMS
  Priya         Java
  Karthik       Python
  Anjali        NULL

------------------------------------------------------------------------

### RIGHT JOIN

All courses + matching students:

``` sql
SELECT s.StudentName, c.CourseName
FROM STUDENT s
RIGHT JOIN COURSE c
ON s.CourseID = c.CourseID;
```

  StudentName   CourseName
  ------------- ------------
  Rahul         DBMS
  Priya         Java
  Karthik       Python
  NULL          AI

------------------------------------------------------------------------

# 9. Real-World Example: Employee and Department

### EMPLOYEE

    EmpID EmpName     DeptID
  ------- --------- --------
        1 Ravi            10
        2 Priya           20
        3 Amit            30
        4 Neha            50

### DEPARTMENT

    DeptID DeptName
  -------- -----------
        10 HR
        20 Finance
        30 IT
        40 Marketing

## Question

> Display all employees and their department names, including employees
> without a matching department.

We need **ALL employees**.

Therefore:

``` sql
SELECT e.EmpID,
       e.EmpName,
       d.DeptName
FROM EMPLOYEE e
LEFT JOIN DEPARTMENT d
ON e.DeptID = d.DeptID;
```

Result:

    EmpID EmpName   DeptName
  ------- --------- ----------
        1 Ravi      HR
        2 Priya     Finance
        3 Amit      IT
        4 Neha      NULL

Neha remains because EMPLOYEE is the left table.

------------------------------------------------------------------------

# 10. RIGHT JOIN Real-World Example

Question:

> Display all departments and employees working in them, including
> departments with no employees.

We need **ALL departments**.

Put DEPARTMENT on the right:

``` sql
SELECT e.EmpName,
       d.DeptName
FROM EMPLOYEE e
RIGHT JOIN DEPARTMENT d
ON e.DeptID = d.DeptID;
```

Result:

  EmpName   DeptName
  --------- -----------
  Ravi      HR
  Priya     Finance
  Amit      IT
  NULL      Marketing

Marketing remains because DEPARTMENT is the right table.

------------------------------------------------------------------------

# 11. RIGHT JOIN Can Be Rewritten as LEFT JOIN

These are logically equivalent:

``` sql
SELECT *
FROM STUDENT s
RIGHT JOIN COURSE c
ON s.CourseID = c.CourseID;
```

and:

``` sql
SELECT *
FROM COURSE c
LEFT JOIN STUDENT s
ON c.CourseID = s.CourseID;
```

Why?

Because:

``` text
STUDENT RIGHT JOIN COURSE
```

means:

> Keep all COURSE rows.

We can instead make COURSE the left table:

``` text
COURSE LEFT JOIN STUDENT
```

and get the same logical result.

------------------------------------------------------------------------

# 12. Finding Unmatched Rows

This is one of the most useful applications of OUTER JOINs.

## Find students with no matching course

``` sql
SELECT s.StudentID,
       s.StudentName
FROM STUDENT s
LEFT JOIN COURSE c
ON s.CourseID = c.CourseID
WHERE c.CourseID IS NULL;
```

Result:

    StudentID StudentName
  ----------- -------------
          104 Anjali

Logic:

1.  LEFT JOIN keeps all students.
2.  Anjali has no matching course.
3.  COURSE columns become `NULL`.
4.  `WHERE c.CourseID IS NULL` finds the unmatched student.

------------------------------------------------------------------------

## Find courses with no students

``` sql
SELECT c.CourseID,
       c.CourseName
FROM STUDENT s
RIGHT JOIN COURSE c
ON s.CourseID = c.CourseID
WHERE s.StudentID IS NULL;
```

Result:

    CourseID CourseName
  ---------- ------------
           4 AI

Logic:

1.  RIGHT JOIN keeps all courses.
2.  AI has no matching student.
3.  STUDENT columns become `NULL`.
4.  `WHERE s.StudentID IS NULL` finds the unmatched course.

------------------------------------------------------------------------

# 13. Important: WHERE Can Change an OUTER JOIN

Consider:

``` sql
SELECT s.StudentName,
       c.CourseName
FROM STUDENT s
LEFT JOIN COURSE c
ON s.CourseID = c.CourseID
WHERE c.CourseName = 'DBMS';
```

The `WHERE` condition removes rows where `c.CourseName` is `NULL`.

So unmatched students such as Anjali disappear.

This can make the result behave like an INNER JOIN for that condition.

If you want to keep all students and only match DBMS, put the condition
in `ON`:

``` sql
SELECT s.StudentName,
       c.CourseName
FROM STUDENT s
LEFT JOIN COURSE c
ON s.CourseID = c.CourseID
AND c.CourseName = 'DBMS';
```

This preserves the LEFT JOIN behavior.

------------------------------------------------------------------------

# 14. SQL\*Plus Practical Example

Create COURSE:

``` sql
CREATE TABLE COURSE (
    CourseID NUMBER PRIMARY KEY,
    CourseName VARCHAR2(30)
);
```

Insert data:

``` sql
INSERT INTO COURSE VALUES (1, 'DBMS');
INSERT INTO COURSE VALUES (2, 'Java');
INSERT INTO COURSE VALUES (3, 'Python');
INSERT INTO COURSE VALUES (4, 'AI');
```

Create STUDENT:

``` sql
CREATE TABLE STUDENT (
    StudentID NUMBER PRIMARY KEY,
    StudentName VARCHAR2(30),
    CourseID NUMBER
);
```

Insert data:

``` sql
INSERT INTO STUDENT VALUES (101, 'Rahul', 1);
INSERT INTO STUDENT VALUES (102, 'Priya', 2);
INSERT INTO STUDENT VALUES (103, 'Karthik', 3);
INSERT INTO STUDENT VALUES (104, 'Anjali', 5);
```

Save changes:

``` sql
COMMIT;
```

------------------------------------------------------------------------

# 15. LEFT OUTER JOIN --- Practical Query

``` sql
SELECT s.StudentID,
       s.StudentName,
       c.CourseName
FROM STUDENT s
LEFT OUTER JOIN COURSE c
ON s.CourseID = c.CourseID;
```

Important output:

``` text
101  Rahul     DBMS
102  Priya     Java
103  Karthik   Python
104  Anjali    NULL
```

------------------------------------------------------------------------

# 16. RIGHT OUTER JOIN --- Practical Query

``` sql
SELECT s.StudentID,
       s.StudentName,
       c.CourseName
FROM STUDENT s
RIGHT OUTER JOIN COURSE c
ON s.CourseID = c.CourseID;
```

Important output:

``` text
101   Rahul      DBMS
102   Priya      Java
103   Karthik    Python
NULL  NULL       AI
```

------------------------------------------------------------------------

# 17. Exam Question Templates

## If the question says:

> Display all students and their course names, including students who
> are not enrolled in a valid course.

Use:

``` sql
SELECT s.StudentID,
       s.StudentName,
       c.CourseName
FROM STUDENT s
LEFT JOIN COURSE c
ON s.CourseID = c.CourseID;
```

Because we need **all STUDENTS**.

------------------------------------------------------------------------

## If the question says:

> Display all courses and the students enrolled in them, including
> courses with no students.

Use:

``` sql
SELECT s.StudentName,
       c.CourseName
FROM STUDENT s
RIGHT JOIN COURSE c
ON s.CourseID = c.CourseID;
```

Because we need **all COURSES**.

------------------------------------------------------------------------

# 18. Common Mistakes

### Mistake 1 --- Choosing based on table importance

Do not think:

> "Student is the main table, so I must use LEFT JOIN."

Instead ask:

> **Which table's ALL rows do I need?**

------------------------------------------------------------------------

### Mistake 2 --- Confusing NULL

If you see:

``` text
Anjali   NULL
```

it does NOT mean Anjali does not exist.

It means:

> Anjali exists, but there is no matching COURSE row.

------------------------------------------------------------------------

### Mistake 3 --- Confusing INNER and OUTER JOIN

``` text
INNER JOIN
→ only matches

LEFT JOIN
→ all LEFT + matches

RIGHT JOIN
→ all RIGHT + matches
```

------------------------------------------------------------------------

# 19. Quick Comparison

  Join               All rows preserved from   Unmatched side becomes NULL
  ------------------ ------------------------- -----------------------------
  INNER JOIN         Neither                   N/A
  LEFT OUTER JOIN    Left table                Right table
  RIGHT OUTER JOIN   Right table               Left table
  FULL OUTER JOIN    Both tables               Opposite side

------------------------------------------------------------------------

# 20. One-Minute Revision

## LEFT OUTER JOIN

``` sql
A LEFT JOIN B
```

means:

``` text
ALL A
+
matching B
```

No matching B:

``` text
B columns = NULL
```

------------------------------------------------------------------------

## RIGHT OUTER JOIN

``` sql
A RIGHT JOIN B
```

means:

``` text
matching A
+
ALL B
```

No matching A:

``` text
A columns = NULL
```

------------------------------------------------------------------------

# 21. Golden Rule for the Practical Exam

Before writing an OUTER JOIN, ask:

> **"Which table do I want ALL rows from?"**

If the answer is:

``` text
STUDENT
```

write:

``` sql
FROM STUDENT
LEFT JOIN COURSE
```

If the answer is:

``` text
COURSE
```

you can write:

``` sql
FROM STUDENT
RIGHT JOIN COURSE
```

or equivalently:

``` sql
FROM COURSE
LEFT JOIN STUDENT
```

The single most important rule is:

> **LEFT JOIN keeps everything from the LEFT table. RIGHT JOIN keeps
> everything from the RIGHT table.**

------------------------------------------------------------------------

# 22. Practice Questions

### Q1

Display all students and their course names, including students who do
not have a matching course.

**Hint:** Keep all STUDENT rows.

### Q2

Display all courses and the students enrolled in them, including courses
with no students.

**Hint:** Keep all COURSE rows.

### Q3

Find students who do not have a matching course.

**Hint:** Use `LEFT JOIN` + `IS NULL`.

### Q4

Find courses that do not have any students.

**Hint:** Use `RIGHT JOIN` + `IS NULL`.

### Q5

Explain why these two queries are logically equivalent:

``` sql
FROM STUDENT s
RIGHT JOIN COURSE c
ON s.CourseID = c.CourseID;
```

and:

``` sql
FROM COURSE c
LEFT JOIN STUDENT s
ON c.CourseID = s.CourseID;
```

------------------------------------------------------------------------

# Final Cheat Sheet

``` text
LEFT JOIN
    ↓
Keep ALL rows from LEFT table
    ↓
Add matching rows from RIGHT table
    ↓
No match → NULL on RIGHT


RIGHT JOIN
    ↓
Keep ALL rows from RIGHT table
    ↓
Add matching rows from LEFT table
    ↓
No match → NULL on LEFT
```

### Syntax

``` sql
SELECT ...
FROM A
LEFT OUTER JOIN B
ON A.id = B.id;
```

``` sql
SELECT ...
FROM A
RIGHT OUTER JOIN B
ON A.id = B.id;
```

### Remember

**LEFT = ALL left rows**

**RIGHT = ALL right rows**
