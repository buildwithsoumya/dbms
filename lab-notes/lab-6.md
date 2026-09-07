# Oracle SQL — Joins (Complete Guide)

Based on your **Joins slide deck** (the EasyShop retail examples) and **Exercise 6**. The deck teaches joins with `item`/`retailstock`/`employee`/`retailoutlet`/`customer` examples — this guide keeps those short illustrative examples where they explain a concept best, but reworks every query against your actual **FLIGHT / PASSENGER / BOOKING** schema so you can run them directly.

> **Reminder from Exercise 6:** suffix your table names with your registration number, and — importantly — the exercise tells you to **add a few more sample rows first**: some flights with no bookings, some passengers with multiple bookings, some passengers with zero bookings, a few pending/cancelled bookings, and some NULL seat numbers/fares. Outer joins only get interesting when there's actually a "missing side" to reveal.

---

## 0. Why joins exist

A relational database deliberately splits information across multiple tables (one row per flight, one per passenger, one per booking) instead of one giant flat table — this avoids repeating airline/passenger details on every single row. But that means to answer a real question like *"which passengers booked which flights?"*, you need to **join** the tables back together based on the columns that link them (`BOOKING.FlightID` ↔ `FLIGHT.FlightID`, `BOOKING.PassengerID` ↔ `PASSENGER.PassengerID`).

---

## 1. The schema and its relationships

```sql
FLIGHT(FlightID, Airline, Source, Destination, Fare, Departure_Date)
PASSENGER(PassengerID, Name, Gender, Age, Email, Phone)
BOOKING(BookingID, FlightID, PassengerID, BookingDate, SeatNo, Status)
```
`BOOKING` sits in the middle — it's a **linking table** with two foreign keys, one to each parent. Most of your join queries will connect `PASSENGER` → `BOOKING` → `FLIGHT` (or a subset of that chain), because that's the only path that relates a passenger to a flight.

---

## 2. INNER JOIN — only matching rows from both tables

**Syntax:**
```sql
SELECT columns
FROM tableA a
INNER JOIN tableB b ON a.common_column = b.common_column;
```

**List passenger names along with their booked flight IDs (Exercise Q1):**
```sql
SELECT p.Name, b.FlightID
FROM PASSENGER p
INNER JOIN BOOKING b ON p.PassengerID = b.PassengerID;
```
Only passengers who **have** a matching booking row appear. A passenger with zero bookings is silently dropped — that's the defining trait of `INNER JOIN`.

**Joining all three tables (flight details + passenger + booking):**
```sql
SELECT p.Name, f.Airline, f.Source, f.Destination, b.Status
FROM PASSENGER p
INNER JOIN BOOKING b ON p.PassengerID = b.PassengerID
INNER JOIN FLIGHT f  ON b.FlightID    = f.FlightID
WHERE b.Status = 'Confirmed';
```
This answers Exercise Q3 — *"flight details for passengers who have confirmed bookings."* Note the rule from the deck: **joining N tables needs at least N−1 join conditions** — here 3 tables, 2 `ON` conditions.

**Display booking ID, passenger name, and airline for Pending bookings (Q2):**
```sql
SELECT b.BookingID, p.Name, f.Airline
FROM BOOKING b
INNER JOIN PASSENGER p ON b.PassengerID = p.PassengerID
INNER JOIN FLIGHT f    ON b.FlightID    = f.FlightID
WHERE b.Status = 'Pending';
```

---

## 3. LEFT OUTER JOIN — everything from the left table, matched or not

**Syntax:**
```sql
SELECT columns
FROM tableA a
LEFT OUTER JOIN tableB b ON a.common_column = b.common_column;
```
Every row from `tableA` (the "left" table) appears **at least once**, even if there's no match in `tableB` — in that case, all of `tableB`'s columns come back as `NULL` for that row.

**List all flights and their passengers — including flights with no bookings (Q5):**
```sql
SELECT f.FlightID, f.Airline, p.Name
FROM FLIGHT f
LEFT OUTER JOIN BOOKING b  ON f.FlightID    = b.FlightID
LEFT OUTER JOIN PASSENGER p ON b.PassengerID = p.PassengerID;
```
`FLIGHT` is on the left, so every flight shows up. A flight nobody has booked yet still appears, with `p.Name` as `NULL`.

**Deck's classic illustration (employee/outlet), for comparison:**
```sql
SELECT e.empid, e.empname, r.retailoutletid, r.retailoutletlocation
FROM employee e
LEFT OUTER JOIN retailoutlet r ON e.worksin = r.retailoutletid;
```
Employee "Allen", who isn't assigned to any outlet, still shows up with `retailoutletid`/`retailoutletlocation` as `NULL` — that's exactly the gap `INNER JOIN` would have silently dropped.

---

## 4. RIGHT OUTER JOIN — everything from the right table, matched or not

Mirror image of `LEFT OUTER JOIN` — every row from the table **after** `RIGHT OUTER JOIN` is guaranteed to appear.

```sql
SELECT f.FlightID, f.Airline, p.Name
FROM BOOKING b
RIGHT OUTER JOIN FLIGHT f ON b.FlightID = f.FlightID
LEFT OUTER JOIN PASSENGER p ON b.PassengerID = p.PassengerID;
```
Here `FLIGHT` (the right side of the first join) is guaranteed to be fully represented — functionally identical to writing `FLIGHT LEFT OUTER JOIN BOOKING`. In practice, most people avoid `RIGHT OUTER JOIN` entirely and just swap the table order to use `LEFT OUTER JOIN` instead, since it reads more naturally — but you should recognize `RIGHT OUTER JOIN` when you see it, since Exercise 6 explicitly asks you to practice it.

---

## 5. FULL OUTER JOIN — everything from both tables

Combines `LEFT` and `RIGHT` — every row from **both** tables appears at least once; unmatched rows on either side get `NULL`s for the other table's columns.

**Find all passengers and flights, even if there is no booking made (Q6):**
```sql
SELECT p.Name, f.FlightID
FROM PASSENGER p
FULL OUTER JOIN BOOKING b ON p.PassengerID = b.PassengerID
FULL OUTER JOIN FLIGHT f  ON b.FlightID    = f.FlightID;
```
This surfaces passengers with zero bookings **and** flights with zero bookings, all in the same result — the most permissive join type.

**Show all bookings along with passenger names and flight details, where either side may be missing (Q7):**
```sql
SELECT b.BookingID, p.Name, f.Airline, f.Source, f.Destination
FROM BOOKING b
FULL OUTER JOIN PASSENGER p ON b.PassengerID = p.PassengerID
FULL OUTER JOIN FLIGHT f    ON b.FlightID    = f.FlightID;
```

### Outer joins at a glance

| Join type | Guarantees every row from... | NULLs appear on... |
|---|---|---|
| `LEFT OUTER JOIN` | the table before the keyword | the right table's columns, when unmatched |
| `RIGHT OUTER JOIN` | the table after the keyword | the left table's columns, when unmatched |
| `FULL OUTER JOIN` | both tables | either side's columns, when unmatched |

---

## 6. CROSS JOIN — the Cartesian product

**Syntax:**
```sql
SELECT * FROM tableA CROSS JOIN tableB;
```
No `ON` condition at all — every row of `tableA` is paired with **every** row of `tableB`. If `tableA` has *m* rows and `tableB` has *n* rows, the result has *m × n* rows.

```sql
SELECT * FROM PASSENGER CROSS JOIN FLIGHT;
```
With 6 passengers and 6 flights, this produces 36 rows — every possible passenger-flight combination, whether or not that passenger actually booked that flight. As the deck's manager scenario shows: this is almost never what you actually want for reporting — it doesn't answer any real question, since the pairing is meaningless. It's mostly useful for generating combinations (e.g. every possible seat × row for a seating chart) or as a teaching example of what joins *without* a matching condition explode into.

---

## 7. SELF JOIN — a table joined with itself

Not a distinct SQL keyword — it's any join (usually `INNER JOIN`) where **both sides are the same table**, given two different aliases so you can distinguish "this row" from "that other row in the same table."

**Deck's example — matching husbands to wives within one `customer` table:**
```sql
SELECT h.customerid, h.customername AS husband, w.customername AS wife
FROM customer h
INNER JOIN customer w ON h.spouse = w.customerid AND h.gender = 'M';
```
`h` and `w` are two independent "views" into the same table — every row of `customer` is compared against every other row (including itself) via the `spouse` column, filtered down to just the male-to-spouse pairings.

**Applied to your schema — find pairs of flights with the same source but different destinations (Q11):**
```sql
SELECT f1.FlightID AS Flight1, f2.FlightID AS Flight2, f1.Source
FROM FLIGHT f1
INNER JOIN FLIGHT f2
  ON f1.Source = f2.Source
  AND f1.Destination <> f2.Destination
  AND f1.FlightID < f2.FlightID;
```
`f1` and `f2` are the same `FLIGHT` table aliased twice. The `f1.FlightID < f2.FlightID` condition is a common self-join trick to avoid getting each pair twice (once as A-B, once as B-A) and to avoid a flight pairing with itself.

---

## 8. NATURAL JOIN and the USING clause

Two shortcuts for when the join columns share the **exact same name** in both tables.

**NATURAL JOIN** — automatically joins on every column with a matching name in both tables, no `ON` needed:
```sql
SELECT * FROM BOOKING NATURAL JOIN FLIGHT;
```
This works cleanly here because `FlightID` is the shared column name. **Caution:** if the tables happen to share *any other* same-named column you didn't intend to join on, `NATURAL JOIN` silently includes it too — this makes it risky on wider tables, and most Oracle developers avoid it for that reason.

**USING clause** — you name explicitly which shared column to join on, keeping the rest of the automatic behavior:
```sql
SELECT BookingID, Airline, Fare
FROM BOOKING JOIN FLIGHT USING (FlightID);
```
Safer than `NATURAL JOIN` since you control exactly which column drives the match, but still saves you from writing out `BOOKING.FlightID = FLIGHT.FlightID`. One quirk: with `USING`, you refer to the shared column as plain `FlightID` (no table prefix) in the rest of the query — Oracle treats it as a single merged column.

---

## 9. Worked examples for the remaining exercise questions

**Passengers who booked more than one flight (Q8) — join + GROUP BY/HAVING:**
```sql
SELECT p.Name, COUNT(*) AS Booking_Count
FROM PASSENGER p
INNER JOIN BOOKING b ON p.PassengerID = b.PassengerID
GROUP BY p.Name
HAVING COUNT(*) > 1;
```

**Passengers (name, age) who booked flights from Delhi (Q9):**
```sql
SELECT p.Name, p.Age
FROM PASSENGER p
INNER JOIN BOOKING b ON p.PassengerID = b.PassengerID
INNER JOIN FLIGHT f  ON b.FlightID    = f.FlightID
WHERE f.Source = 'Delhi';
```

**Passenger who booked the cheapest available flight (Q10) — join + subquery together:**
```sql
SELECT p.Name, f.Fare
FROM PASSENGER p
INNER JOIN BOOKING b ON p.PassengerID = b.PassengerID
INNER JOIN FLIGHT f  ON b.FlightID    = f.FlightID
WHERE f.Fare = (SELECT MIN(Fare) FROM FLIGHT);
```
This combines what you already know from the subqueries guide (single-row `=` subquery for the minimum) with a join to bring in the passenger's name — a good example of how joins and subqueries aren't competing techniques, they're often used together.

**Passenger names with their booking dates and flight departure dates (Q12):**
```sql
SELECT p.Name, b.BookingDate, f.Departure_Date
FROM PASSENGER p
INNER JOIN BOOKING b ON p.PassengerID = b.PassengerID
INNER JOIN FLIGHT f  ON b.FlightID    = f.FlightID;
```

**Booking details of passengers who booked Akasa Air (Q13):**
```sql
SELECT b.*, p.Name
FROM BOOKING b
INNER JOIN PASSENGER p ON b.PassengerID = p.PassengerID
INNER JOIN FLIGHT f    ON b.FlightID    = f.FlightID
WHERE f.Airline = 'Akasa Air';
```

**Passengers not yet assigned a seat number (Q14) — NULL check, not a join trick:**
```sql
SELECT p.Name, b.BookingID
FROM PASSENGER p
INNER JOIN BOOKING b ON p.PassengerID = b.PassengerID
WHERE b.SeatNo IS NULL;
```

**All cancelled bookings with passenger details and flight routes (Q15):**
```sql
SELECT b.BookingID, p.Name, f.Source, f.Destination, b.Status
FROM BOOKING b
INNER JOIN PASSENGER p ON b.PassengerID = p.PassengerID
INNER JOIN FLIGHT f    ON b.FlightID    = f.FlightID
WHERE b.Status = 'Cancelled';
```

---

## Quick reference — which join for which question phrasing

| Question says... | Use |
|---|---|
| "…for passengers/flights **who have a booking**" | `INNER JOIN` |
| "…**include** flights/passengers with **no** booking" | `LEFT` / `RIGHT OUTER JOIN` (whichever side must always show) |
| "…**even if** either side is missing" | `FULL OUTER JOIN` |
| "…every possible combination" | `CROSS JOIN` |
| "…compare rows **within the same table**" (pairs, hierarchies, matching by attribute) | `SELF JOIN` |

---

## Quick self-test

1. Add 2–3 new rows to `FLIGHT`, `PASSENGER`, and `BOOKING` as the exercise instructs — at least one flight with zero bookings, one passenger with two bookings, one passenger with zero bookings.
2. Run the same `INNER JOIN` and `LEFT OUTER JOIN` query on `FLIGHT`/`BOOKING` before and after adding that data — confirm the row counts differ once there's an unmatched flight to reveal.
3. Write the self-join for "same source, different destination" (Q11) and verify it doesn't return a flight paired with itself.
4. Try `NATURAL JOIN` between `BOOKING` and `FLIGHT`, then rewrite it with `USING (FlightID)` and again with an explicit `ON` — confirm all three give the same result here.
