# Oracle SQL — Data Retrieval, Transactions & Single-Row Functions

Based on your Week 4 tutorial (Part A + Part B), Exercise 4, and the Single Row Functions deck. All examples run against the **FLIGHT / PASSENGER / BOOKING** schema from your `table_creation.txt`, so you can paste them straight into SQL*Plus.

> **Reminder from Exercise 4:** when you create these tables yourself for submission, suffix the table name with your registration number — e.g. `FLIGHT_23BCE1587`, `PASSENGER_23BCE1587`, `BOOKING_23BCE1587`.

---

## 0. The dataset we're working with

```sql
CREATE TABLE FLIGHT (
    FlightID       NUMBER(5) PRIMARY KEY,
    Airline        VARCHAR2(20),
    Source         VARCHAR2(20),
    Destination    VARCHAR2(20),
    Fare           NUMBER(8,2),
    Departure_Date DATE
);

CREATE TABLE PASSENGER (
    PassengerID NUMBER(5) PRIMARY KEY,
    Name        VARCHAR2(30),
    Gender      CHAR(1),
    Age         NUMBER(3),
    Email       VARCHAR2(40),
    Phone       NUMBER(10)
);

CREATE TABLE BOOKING (
    BookingID   NUMBER(5) PRIMARY KEY,
    FlightID    NUMBER(5),
    PassengerID NUMBER(5),
    BookingDate DATE,
    SeatNo      VARCHAR2(5),
    Status      VARCHAR2(15),
    CONSTRAINT fk_booking_flight    FOREIGN KEY (FlightID)    REFERENCES FLIGHT(FlightID),
    CONSTRAINT fk_booking_passenger FOREIGN KEY (PassengerID) REFERENCES PASSENGER(PassengerID)
);
```
Notice `BOOKING` is a **child table of two parents** — a common real-world pattern (a "junction"/linking table) where each row connects one flight to one passenger. Several rows deliberately contain `NULL` (missing `Fare`, missing `Departure_Date`, missing `Email`, missing `SeatNo`, etc.) so you can practice NULL-handling functions later.

---

## 1. Transaction Control — COMMIT, ROLLBACK, SAVEPOINT

Every `INSERT`/`UPDATE`/`DELETE` in Oracle happens inside an implicit transaction. Nothing is permanent until you `COMMIT` — and until then, you can undo it.

```sql
INSERT INTO FLIGHT VALUES (107,'Emirates','Chennai','Dubai',15000,SYSDATE);
-- not yet permanent

COMMIT;   -- makes it permanent — cannot be undone after this
```

**ROLLBACK — undo everything since the last commit:**
```sql
UPDATE PASSENGER SET Age = Age + 1 WHERE PassengerID = 201;
-- realize this was a mistake
ROLLBACK;   -- the age change is undone
```

**SAVEPOINT — undo only part of a transaction:**
```sql
UPDATE PASSENGER SET Phone = 9999999999 WHERE PassengerID = 202;
SAVEPOINT sp1;                              -- mark this point

UPDATE PASSENGER SET Phone = 8888888888 WHERE PassengerID = 203;
-- this second update turns out to be wrong

ROLLBACK TO sp1;   -- undoes only the second update; the first one (202) stays

COMMIT;            -- finalize everything up to sp1
```

**Mental model:** `COMMIT` = save game. `ROLLBACK` = reload last save. `SAVEPOINT` = a checkpoint mid-level you can rewind to without restarting the whole level.

---

## 2. The SELECT toolkit

### Full clause order (this order is fixed — Oracle parses it this way even though you write SELECT first)
```sql
SELECT col1, col2
FROM   table
WHERE  condition        -- filters rows BEFORE grouping
GROUP BY col1
HAVING condition        -- filters groups AFTER grouping
ORDER BY col1;
```

| Clause | Purpose |
|---|---|
| SELECT | Choose columns to display |
| FROM | Which table(s) |
| WHERE | Filter individual rows |
| GROUP BY | Collapse rows sharing a value into groups |
| HAVING | Filter groups (only usable after GROUP BY) |
| ORDER BY | Sort the final result |
| DISTINCT | Remove duplicate rows from the result |

### DISTINCT — unique values only
```sql
SELECT DISTINCT Source FROM FLIGHT;
```
Returns each source city once, no matter how many flights depart from it.

### WHERE + BETWEEN — range filter (inclusive on both ends)
```sql
SELECT * FROM FLIGHT WHERE Fare BETWEEN 4000 AND 8000;
```
Matches fares from 4000 to 8000, **including** both boundaries.

### IN — match against a list of values
```sql
SELECT * FROM FLIGHT WHERE Source IN ('Chennai','Mumbai','Delhi');
```
Shorthand for `Source='Chennai' OR Source='Mumbai' OR Source='Delhi'` — cleaner to write and read.

### LIKE — pattern matching
```sql
SELECT * FROM PASSENGER WHERE Name LIKE 'A%';
```
| Wildcard | Meaning |
|---|---|
| `%` | zero or more characters |
| `_` | exactly one character |

`'A%'` → starts with A. `'_ohn'` → exactly 4 letters ending in "ohn" (e.g. "John"). `'%a%'` → contains "a" anywhere.

### ORDER BY — sorting
```sql
SELECT * FROM FLIGHT ORDER BY Fare DESC;              -- highest fare first
SELECT * FROM PASSENGER ORDER BY Age ASC;              -- youngest first
SELECT * FROM FLIGHT ORDER BY Airline ASC, Fare DESC;   -- multi-column sort
```

### Column aliasing — renaming output headers
```sql
SELECT FlightID AS Flight_Number, Airline AS Airline_Name, Fare AS Ticket_Fare
FROM FLIGHT;
```
`AS` is optional in Oracle (`Airline Airline_Name` works too) but including it makes queries easier to read.

---

## 3. Aggregate functions

These collapse many rows into **one summary value**.

```sql
SELECT COUNT(*)        FROM FLIGHT;   -- total number of flights
SELECT AVG(Fare)        FROM FLIGHT;   -- average fare (NULLs ignored automatically)
SELECT MAX(Fare)        FROM FLIGHT;   -- highest fare
SELECT MIN(Fare)        FROM FLIGHT;   -- lowest fare
SELECT SUM(Fare)        FROM FLIGHT;   -- total of all fares
```
**Important:** all of these except `COUNT(*)` silently skip `NULL` values — they don't treat NULL as zero. So `AVG(Fare)` on 6 rows where 1 fare is NULL divides by 5, not 6.

**All five together, with aliases:**
```sql
SELECT COUNT(*) AS Total_Flights,
       AVG(Fare) AS Avg_Fare,
       MAX(Fare) AS Max_Fare,
       MIN(Fare) AS Min_Fare,
       SUM(Fare) AS Total_Fare
FROM FLIGHT;
```

---

## 4. GROUP BY and HAVING

**GROUP BY** buckets rows that share a value, so aggregate functions run **per bucket** instead of over the whole table.

```sql
SELECT Status, COUNT(*) AS Booking_Count
FROM BOOKING
GROUP BY Status;
```
This gives one row per distinct `Status` ('Confirmed', 'Pending', 'Cancelled', …) with how many bookings fall into each.

**HAVING** filters those groups — like `WHERE`, but applied *after* grouping (so it can reference aggregate functions, which `WHERE` cannot):

```sql
SELECT Status, COUNT(*) AS Booking_Count
FROM BOOKING
GROUP BY Status
HAVING COUNT(*) > 1;
```
Only statuses with more than one booking survive.

**Why not just use WHERE?** Because `WHERE COUNT(*) > 1` is invalid — `WHERE` runs before grouping happens, so aggregate values don't exist yet. `HAVING` exists specifically to filter after the aggregation.

**Combined real example (from Part A):**
```sql
SELECT department, COUNT(*) AS total
FROM employees
WHERE salary > 50000
GROUP BY department
HAVING COUNT(*) > 3
ORDER BY total DESC;
```
Reading it in execution order: filter individual employees earning over 50,000 → group survivors by department → keep only departments with more than 3 people → sort by count, highest first.

---

## 5. Table maintenance

**UPDATE — modify existing rows:**
```sql
UPDATE FLIGHT SET Fare = Fare + 500 WHERE FlightID = 104;
```

**RENAME — rename a whole table:**
```sql
RENAME BOOKING TO FLIGHT_BOOKING;
```

**Backup a table with CREATE TABLE AS SELECT (CTAS):**
```sql
CREATE TABLE FLIGHT_BACKUP AS SELECT * FROM FLIGHT;
```
As covered earlier, this copies structure **and** data in one shot — useful right before you run risky `UPDATE`/`DELETE` statements in a lab session.

---

## 6. Single-Row Functions

These run **once per row** and return one value per row (as opposed to aggregate functions, which collapse many rows into one value). They work in `SELECT`, `WHERE`, and `ORDER BY`.

### 6.1 Character functions

| Function | What it does | Example |
|---|---|---|
| `UPPER(str)` | Converts to uppercase | `UPPER('oracle')` → `'ORACLE'` |
| `LOWER(str)` | Converts to lowercase | `LOWER('ORACLE')` → `'oracle'` |
| `INITCAP(str)` | Capitalizes first letter of each word | `INITCAP('oracle database')` → `'Oracle Database'` |
| `CONCAT(a,b)` | Joins two strings (only 2 args) | `CONCAT('Hello','World')` → `'HelloWorld'` |
| `\|\|` | Joins strings (any number, more common than CONCAT) | `'Hello' \|\| ' ' \|\| 'World'` |
| `LENGTH(str)` | Number of characters | `LENGTH('Oracle')` → `6` |
| `SUBSTR(str,start,len)` | Extract part of a string | `SUBSTR('Oracle',1,3)` → `'Ora'` |
| `INSTR(str,sub)` | Position where a substring first appears | `INSTR('Oracle','a')` → `3` |
| `LPAD(str,len,ch)` | Left-pad to a length | `LPAD('123',5,'0')` → `'00123'` |
| `RPAD(str,len,ch)` | Right-pad to a length | `RPAD('abc',5,'.')` → `'abc..'` |
| `TRIM(ch FROM str)` | Strip leading/trailing characters (default: spaces) | `TRIM(' O ')` → `'O'` |
| `REPLACE(str,old,new)` | Replace all occurrences of a substring | `REPLACE('JACK','J','BL')` → `'BLACK'` |

**Applied to your PASSENGER table:**
```sql
SELECT Name, UPPER(Name), LOWER(Name), INITCAP(Name) FROM PASSENGER;

SELECT Name, Email, Name || ' - ' || Email AS Contact_Info FROM PASSENGER;

SELECT Name, LENGTH(Name) AS Name_Length FROM PASSENGER;

SELECT Airline, SUBSTR(Airline,1,3) AS Airline_Short FROM FLIGHT;
```

### 6.2 Number functions

| Function | What it does | Example |
|---|---|---|
| `ROUND(n, d)` | Rounds to `d` decimal places | `ROUND(123.456,1)` → `123.5` |
| `TRUNC(n, d)` | Cuts off (no rounding) at `d` decimal places | `TRUNC(123.456,1)` → `123.4` |
| `MOD(n1, n2)` | Remainder of division | `MOD(10,3)` → `1` |
| `ABS(n)` | Absolute value | `ABS(-50)` → `50` |
| `CEIL(n)` | Rounds up to next integer | `CEIL(4.2)` → `5` |
| `FLOOR(n)` | Rounds down to previous integer | `FLOOR(4.8)` → `4` |
| `SQRT(n)` | Square root | `SQRT(25)` → `5` |

**Applied to FLIGHT/PASSENGER:**
```sql
SELECT Fare, ROUND(Fare,-1) AS Rounded_To_10 FROM FLIGHT;  -- negative d rounds LEFT of decimal
SELECT Fare, CEIL(Fare) AS Fare_Ceil, FLOOR(Fare) AS Fare_Floor FROM FLIGHT;
SELECT Fare, Fare + 500 AS Fare_Plus_500 FROM FLIGHT;
SELECT PassengerID, MOD(PassengerID,2) AS Is_Odd FROM PASSENGER;  -- 0 = even, 1 = odd
SELECT ABS(-50) AS Absolute_Value FROM DUAL;
```
> `DUAL` is Oracle's built-in one-row, one-column dummy table — used when you want to evaluate an expression that doesn't come from real table data.

### 6.3 Date functions

| Function | What it does | Example |
|---|---|---|
| `SYSDATE` | Current system date/time | `SELECT SYSDATE FROM DUAL;` |
| `ADD_MONTHS(d,n)` | Adds `n` months to a date | `ADD_MONTHS(SYSDATE,2)` |
| `MONTHS_BETWEEN(d1,d2)` | Number of months between two dates | `MONTHS_BETWEEN(SYSDATE,DOB)` |
| `NEXT_DAY(d,'DAY')` | Next occurrence of a given weekday | `NEXT_DAY(SYSDATE,'MONDAY')` |
| `LAST_DAY(d)` | Last day of that date's month | `LAST_DAY(SYSDATE)` |
| `ROUND(d,'fmt')` | Rounds a date to the nearest unit | `ROUND(SYSDATE,'MONTH')` |
| `TRUNC(d,'fmt')` | Truncates a date to the start of a unit | `TRUNC(SYSDATE,'MONTH')` |

**Applied to FLIGHT/BOOKING:**
```sql
SELECT FlightID, Departure_Date, Departure_Date + 10 AS Ten_Days_Later FROM FLIGHT;
-- dates support plain arithmetic: date + number = date shifted by that many days

SELECT BookingID, MONTHS_BETWEEN(SYSDATE, BookingDate) AS Months_Since_Booking FROM BOOKING;

SELECT FlightID, Departure_Date,
       TO_CHAR(Departure_Date,'MM') AS Dep_Month,
       TO_CHAR(Departure_Date,'YYYY') AS Dep_Year
FROM FLIGHT;

SELECT SYSDATE FROM DUAL;

SELECT BookingID, Departure_Date, BookingDate,
       (Departure_Date - BookingDate) AS Days_Between
FROM BOOKING JOIN FLIGHT ON BOOKING.FlightID = FLIGHT.FlightID;
-- subtracting two DATEs gives a plain number of days
```

### 6.4 Conversion functions

Oracle is strict about types — you often need to explicitly convert between text, numbers, and dates.

| Function | What it does | Example |
|---|---|---|
| `TO_CHAR(date, fmt)` | Date/number → formatted string | `TO_CHAR(Departure_Date,'DD-Mon-YYYY')` |
| `TO_NUMBER(str)` | String → number | `TO_NUMBER('1234.56')` → `1234.56` |
| `TO_DATE(str, fmt)` | String → date (must match the format mask) | `TO_DATE('2025-08-05','YYYY-MM-DD')` |

```sql
SELECT TO_CHAR(Departure_Date,'DD-Mon-YYYY') AS Formatted_Date FROM FLIGHT;
SELECT TO_NUMBER('1234.56') AS Number_Val FROM DUAL;
SELECT TO_DATE('2025-08-05','YYYY-MM-DD') AS Converted_Date FROM DUAL;
```
This is exactly what your `table_creation.txt` inserts use: `TO_DATE('20-Aug-2025','DD-Mon-YYYY')` — the second argument tells Oracle exactly how to read the text.

---

## 7. NULL-handling functions

Your dataset has NULLs everywhere on purpose (missing `Fare`, `Email`, `SeatNo`, `Airline`) — these functions are exactly for that.

### NVL(expr, replacement) — the simplest: substitute one fixed value for NULL
```sql
SELECT FlightID, NVL(Fare,0) AS Final_Fare FROM FLIGHT;
-- flight 102's NULL fare becomes 0; every other fare is unchanged

SELECT FlightID, NVL(Airline,'Not Assigned') AS Airline_Name FROM FLIGHT;
-- flight 106's NULL airline becomes 'Not Assigned'
```

### NVL2(expr, value_if_not_null, value_if_null) — branches both ways
```sql
SELECT Name, Email,
       NVL2(Email,'Provided','Not Provided') AS Email_Status
FROM PASSENGER;
```
Unlike `NVL`, this doesn't just fill in a default — it returns a **different** value depending on whether the expression is NULL or not.

### NULLIF(expr1, expr2) — returns NULL if the two are equal, else returns expr1
```sql
SELECT SeatNo, NULLIF(SeatNo,'12A') AS Result FROM BOOKING;
-- wherever SeatNo = '12A', the result becomes NULL; every other seat number is returned unchanged
```
Useful for flagging/blanking out a specific "known bad" or placeholder value.

### COALESCE(expr1, expr2, ..., default) — first non-NULL value in a list
```sql
SELECT Name,
       COALESCE(Email, TO_CHAR(Phone), 'No Contact Info') AS Contact
FROM PASSENGER;
```
Checks `Email` first; if that's NULL, checks `Phone`; if that's also NULL, falls back to the literal string. This is `NVL` generalized to more than one fallback — think of it as a chain of "else try this instead."

---

## 8. CASE and DECODE — conditional logic in a query

Both let you produce different output values based on conditions, but `CASE` is standard SQL (works everywhere) and reads like an if/elif/else; `DECODE` is Oracle-only and reads like a lookup table.

### CASE — general conditional logic, supports ranges and comparisons
```sql
SELECT Name, Age,
  CASE
    WHEN Age IS NULL THEN 'Unknown'
    WHEN Age < 18     THEN 'Minor'
    WHEN Age <= 60    THEN 'Adult'
    ELSE 'Senior'
  END AS Age_Category
FROM PASSENGER;
```
Conditions are checked **top to bottom**, and the first one that's true wins — order matters.

### DECODE — Oracle's shorthand equivalence-based IF/ELSE
```sql
SELECT Gender, DECODE(Gender,'M','Male','F','Female','Unspecified') AS Gender_Full
FROM PASSENGER;
```
Reads as: if `Gender = 'M'` return `'Male'`, else if `'F'` return `'Female'`, else return `'Unspecified'` (the final unpaired argument is the default). `DECODE` can only test for **equality**, not ranges like `CASE` can — for the age-bracket example above you'd need `CASE`, not `DECODE`.

**Worked seat-type example (from your Exercise 4, Part C):**
```sql
SELECT BookingID, SeatNo,
  DECODE(SUBSTR(SeatNo,-1,1),
    'A','Window','B','Window',
    'C','Middle','D','Middle',
    'E','Aisle','F','Aisle',
    'Unknown') AS Seat_Type
FROM BOOKING;
```
`SUBSTR(SeatNo,-1,1)` grabs the **last character** of the seat number (negative start counts from the end of the string), then `DECODE` maps it to a seat type.

---

## Quick self-test (mirrors your Exercise 4)

1. `SELECT DISTINCT` the source cities in `FLIGHT`.
2. Find flights with fare `BETWEEN` 4000 and 8000.
3. Use `IN` to get flights from Chennai, Mumbai, or Delhi.
4. Use `LIKE` to find passengers whose names start with 'A'.
5. `COUNT`, `AVG`, `MAX`, `MIN`, `SUM` the fares in one query.
6. `GROUP BY Status` on `BOOKING`, then add `HAVING COUNT(*) > 1`.
7. Replace NULL fares with `NVL`, NULL airlines with `NVL`, and build an email-status column with `NVL2`.
8. Write the age-classification `CASE` and the seat-type `DECODE` from Part C — these show up often in exams because they combine several functions at once.
