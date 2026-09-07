# Oracle SQL — Operators, Subqueries & Set Operators

Based on your **Operators and Subqueries tutorial** and **Exercise 5**. All examples run against the **MOVIE / THEATER_SHOW / CUSTOMER / BOOKING** schema — a movie-ticket-booking system where `THEATER_SHOW` links a movie to a specific screening, and `BOOKING` links a customer to a specific show.

> **Reminder from Exercise 5:** suffix your table names with your registration number (`MOVIE_23BCE1587`, etc.), turn on `SPOOL` before you start, and if any of your `SELECT` queries returns zero rows, your output should explicitly show a message like `No records found` rather than a blank result.

---

## 0. The dataset we're working with

```sql
CREATE TABLE MOVIE (
    MovieID       NUMBER PRIMARY KEY,
    Title         VARCHAR2(50),
    Genre         VARCHAR2(20),
    MovieLanguage VARCHAR2(20),
    Duration      NUMBER,
    ReleaseDate   DATE
);

CREATE TABLE THEATER_SHOW (
    ShowID      NUMBER PRIMARY KEY,
    MovieID     NUMBER REFERENCES MOVIE(MovieID),
    TheaterName VARCHAR2(50),
    Location    VARCHAR2(30),
    ShowDate    DATE,
    ShowTime    VARCHAR2(20),
    Price       NUMBER
);

CREATE TABLE CUSTOMER (
    CustomerID NUMBER PRIMARY KEY,
    Name       VARCHAR2(50),
    Email      VARCHAR2(50),
    Phone      VARCHAR2(15)
);

CREATE TABLE BOOKING (
    BookingID     NUMBER PRIMARY KEY,
    CustomerID    NUMBER REFERENCES CUSTOMER(CustomerID),
    ShowID        NUMBER REFERENCES THEATER_SHOW(ShowID),
    Seats         NUMBER,
    BookingDate   DATE,
    PaymentStatus VARCHAR2(15)
);
```

**Relationship chain:** `MOVIE` → `THEATER_SHOW` (which movie, at which theater/location/price) → `BOOKING` (which customer booked which show). Most subquery questions in this exercise involve walking two or three tables deep along this chain — that's the whole point of nesting subqueries.

---

## 1. What is a subquery?

A **subquery** (or inner/nested query) is a `SELECT` placed inside another query's `WHERE` (or `FROM`/`SELECT`) clause. The inner query runs first, its result feeds into the outer query, and only then does the outer query evaluate.

**Basic shape:**
```sql
SELECT column_list FROM table_name
WHERE column_name operator
  (SELECT column_list FROM another_table WHERE condition);
```

Subqueries split into two families based on **how many rows the inner query returns** — this determines which operator you're allowed to use with it.

| Type | Inner query returns | Operators |
|---|---|---|
| **Single-row subquery** | Exactly one value | `=`, `>`, `<`, `>=`, `<=`, `<>`, `BETWEEN` |
| **Multiple-row subquery** | A list of values | `IN`, `ANY`, `SOME`, `ALL`, `EXISTS`, `NOT IN`, `NOT EXISTS` |

Using a single-row operator (like `=`) with a subquery that returns more than one row throws:
```
ORA-01427: single-row subquery returns more than one row
```
That error is your signal to switch to `IN` or `ANY`/`ALL` instead.

---

## 2. Part A — Single-Row Subqueries

### 2.1 `=` — exact match against one value

**Find the customer with the earliest booking date:**
```sql
SELECT Name
FROM CUSTOMER
WHERE CustomerID = (
  SELECT CustomerID
  FROM BOOKING
  WHERE BookingDate = (SELECT MIN(BookingDate) FROM BOOKING)
);
```
Read from the inside out:
1. `SELECT MIN(BookingDate) FROM BOOKING` → one date value (the earliest booking).
2. `SELECT CustomerID FROM BOOKING WHERE BookingDate = <that date>` → one CustomerID (assuming only one booking happened on that date).
3. Outer query looks up that customer's name.

> **Risk:** if *two* bookings happen to share the earliest date, step 2 returns two rows and the whole query throws `ORA-01427`. That's exactly why the tutorial's next example switches to `IN`.

### 2.2 Comparison operators (`>`, `<`, `>=`, `<=`) against a single value

**Find the movie title with the longest duration:**
```sql
SELECT Title
FROM MOVIE
WHERE Duration = (SELECT MAX(Duration) FROM MOVIE);
```
The inner query collapses to one number (the max duration across all movies); the outer query returns whichever movie(s) match it exactly.

### 2.3 `BETWEEN` with two single-row subqueries

**Find movies released between the oldest and newest Comedy movie release dates:**
```sql
SELECT Title, ReleaseDate
FROM MOVIE
WHERE ReleaseDate BETWEEN
  (SELECT MIN(ReleaseDate) FROM MOVIE WHERE Genre = 'Comedy')
  AND
  (SELECT MAX(ReleaseDate) FROM MOVIE WHERE Genre = 'Comedy');
```
Both subqueries independently return one date each (the earliest and latest Comedy release), and `BETWEEN` uses them as the lower and upper bound — same as `BETWEEN` with two literal dates, just computed instead of typed by hand.

---

## 3. Part B — Multiple-Row Subqueries

### 3.1 `IN` — matches any value in a returned list

**Find all customers tied for the earliest booking (fixes the `=` risk from 2.1):**
```sql
SELECT Name
FROM CUSTOMER
WHERE CustomerID IN (
  SELECT CustomerID FROM BOOKING
  WHERE BookingDate = (SELECT MIN(BookingDate) FROM BOOKING)
);
```
Even if several customers share the earliest booking date, `IN` happily matches against the whole list — no `ORA-01427` risk.

**List customers who booked shows in Chennai (three levels deep):**
```sql
SELECT Name
FROM CUSTOMER
WHERE CustomerID IN (
  SELECT CustomerID FROM BOOKING
  WHERE ShowID IN (
    SELECT ShowID FROM THEATER_SHOW
    WHERE Location = 'Chennai'
  )
);
```
Innermost query: which ShowIDs are in Chennai → middle query: which CustomerIDs booked any of those shows → outer query: those customers' names.

### 3.2 `ALL` — condition must hold against every value returned

**Find movies longer than all Action movies:**
```sql
SELECT Title
FROM MOVIE
WHERE Duration > ALL (
  SELECT Duration FROM MOVIE WHERE Genre = 'Action'
);
```
The movie's duration must beat the **longest** Action movie to satisfy `> ALL` — it's effectively comparing against the maximum of the list.

### 3.3 `ANY` / `SOME` — condition must hold against at least one value

`ANY` and `SOME` are exact synonyms in Oracle — use whichever reads better.

**Find shows priced higher than any show in Bangalore:**
```sql
SELECT TheaterName, Price
FROM THEATER_SHOW
WHERE Price > ANY (
  SELECT Price FROM THEATER_SHOW WHERE Location = 'Bangalore'
);
```
This only needs to beat the **cheapest** Bangalore show to qualify — `> ANY` is effectively comparing against the minimum of the list. (Note this differs from Exercise 5's Question 2, "priced higher than *all* shows in Bangalore" — that one needs `> ALL` instead, matching against the *maximum* Bangalore price. Read the wording carefully; `ANY` and `ALL` give very different result sets.)

**Find movies shorter than some Sci-Fi movies:**
```sql
SELECT Title
FROM MOVIE
WHERE Duration < SOME (
  SELECT Duration FROM MOVIE WHERE Genre = 'Sci-Fi'
);
```
True as long as the movie beats at least one Sci-Fi movie's duration in the comparison direction shown.

> **Quick mental model:** `> ALL` = greater than the biggest one (hardest to satisfy). `> ANY` = greater than the smallest one (easiest to satisfy). Mixing these up is the single most common subquery mistake.

### 3.4 `EXISTS` — true if the subquery returns at least one row (contents don't matter)

**Return all movies if there is any unpaid booking in the system:**
```sql
SELECT Title
FROM MOVIE
WHERE EXISTS (
  SELECT 1 FROM BOOKING WHERE PaymentStatus = 'Unpaid'
);
```
`EXISTS` doesn't care *what* the subquery returns — only *whether* it returns anything. `SELECT 1` is a convention meaning "I just want to test for existence, the actual column values are irrelevant." If even one unpaid booking exists anywhere in the table, the condition is `TRUE` for **every** row in `MOVIE` (since the inner query doesn't reference `MOVIE` at all here) — so all movies come back. If no unpaid booking exists, zero rows come back.

> This particular example is an **uncorrelated** subquery (it doesn't reference the outer table). `EXISTS` is far more commonly used as a **correlated** subquery, referencing the outer row — e.g. "find movies that have at least one show scheduled":
> ```sql
> SELECT Title FROM MOVIE m
> WHERE EXISTS (
>   SELECT 1 FROM THEATER_SHOW t WHERE t.MovieID = m.MovieID
> );
> ```
> Here the inner query re-runs once per outer row, checking each movie individually.

### 3.5 `NOT IN` / `NOT EXISTS` — the negations

```sql
-- Customers who have never made any booking:
SELECT Name FROM CUSTOMER
WHERE CustomerID NOT IN (SELECT CustomerID FROM BOOKING);
```
> **Careful with `NOT IN` and NULLs:** if the subquery's result list contains even a single `NULL`, `NOT IN` returns **no rows at all** (not an error, just silently empty) — because `x <> NULL` is unknown, not true, for every comparison. If the referenced column can contain NULLs, prefer `NOT EXISTS` instead, which doesn't have this trap:
```sql
SELECT Name FROM CUSTOMER c
WHERE NOT EXISTS (
  SELECT 1 FROM BOOKING b WHERE b.CustomerID = c.CustomerID
);
```

### 3.6 `UNIQUE` — Oracle-specific, deduplicates a subquery's rows

```sql
SELECT UNIQUE Price FROM THEATER_SHOW;
```
Functionally identical to `SELECT DISTINCT Price FROM THEATER_SHOW;` — `UNIQUE` is Oracle's older, less commonly used synonym for `DISTINCT`.

---

## 4. Part C — Set Operators (UNION, INTERSECT, MINUS)

These combine the results of **two separate SELECT statements** — not by joining columns side-by-side, but by stacking/comparing whole result sets. Both queries must return the **same number of columns** with **compatible data types**.

### 4.1 UNION — combines two results, removing duplicates

**List all show IDs that were booked by customers and all shows scheduled:**
```sql
SELECT ShowID FROM BOOKING
UNION
SELECT ShowID FROM THEATER_SHOW;
```
Returns every distinct ShowID that appears in *either* table — no repeats, sorted automatically.

### 4.2 UNION ALL — same, but keeps duplicates
```sql
SELECT ShowDate FROM THEATER_SHOW
UNION ALL
SELECT BookingDate FROM BOOKING;
```
Stacks both lists exactly as-is, including repeats — faster than `UNION` since Oracle skips the duplicate-removal step, and appropriate here since you actually *want* to see how many times each date shows up across both tables.

### 4.3 INTERSECT — only rows present in both results

**List shows that were both scheduled and booked:**
```sql
SELECT ShowID FROM THEATER_SHOW
INTERSECT
SELECT ShowID FROM BOOKING;
```
Returns only the ShowIDs that appear in **both** queries — this is essentially asking "which scheduled shows actually got at least one booking?"

### 4.4 MINUS — rows in the first result that are NOT in the second

**List shows that were scheduled but never booked:**
```sql
SELECT ShowID FROM THEATER_SHOW
MINUS
SELECT ShowID FROM BOOKING;
```
`MINUS` is order-sensitive — this returns ShowIDs from `THEATER_SHOW` that don't appear anywhere in `BOOKING`. Flipping the order (`BOOKING MINUS THEATER_SHOW`) would instead look for booked ShowIDs missing from the schedule, which shouldn't exist given the foreign key — a useful sanity check that your data integrity holds.

### Set operators at a glance

| Operator | Keeps | Use when you want... |
|---|---|---|
| `UNION` | Everything in either, duplicates removed | A combined distinct list from two sources |
| `UNION ALL` | Everything in either, duplicates kept | A combined list where counts matter |
| `INTERSECT` | Only rows in both | What overlaps between two sets |
| `MINUS` | Rows in the first, absent from the second | What's "left over" / unmatched |

---

## 5. Mapping onto Exercise 5's question types

Since your exercise lists specific question categories, here's which technique each maps to:

- **"Find the X with the [longest/highest/earliest] Y"** → single-row subquery with `=` and `MAX`/`MIN`.
- **"Find X where condition holds against all/any of Y"** → `> ALL` / `> ANY` (double-check which one the wording actually needs — see the ALL vs ANY warning in Section 3.3).
- **"List X who have at least one Y"** → `IN`, or `EXISTS` if it needs to be correlated to the outer row.
- **"Find X that don't have any Y"** → `NOT IN` (only safe if the subquery column can't be NULL) or `NOT EXISTS` (always safe).
- **"List all distinct A and B from two tables"** → `UNION`.
- **"List A and B including duplicates"** → `UNION ALL`.
- **"Find values common to both"** → `INTERSECT`.
- **"Find values in A that don't appear in B"** → `MINUS`.

---

## Quick self-test

1. Find the theater name hosting the most expensive show — single-row `=` subquery.
2. Find the theater location with the cheapest show — same pattern, `MIN`.
3. List movie titles that have at least one show scheduled — `IN` or correlated `EXISTS`.
4. Find customers whose bookings are all `'Paid'` — think about what `NOT EXISTS` combined with `PaymentStatus <> 'Paid'` would express (a customer with *no* unpaid booking).
5. Run all three set operators (`UNION`, `INTERSECT`, `MINUS`) between `THEATER_SHOW.ShowDate` and `BOOKING.BookingDate`, and explain in your own words what each result physically represents.
