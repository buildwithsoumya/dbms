# DBMS DA2 Review – Pharmacy Management System

**Course:** BACSE202 – Database Systems  
**Project:** Pharmacy Management System  
**Review:** 22 September 2026  
**Database:** Oracle SQL / PL/SQL

## 1. DA2 Requirements

- Table creation and insertion with appropriate constraints
- Advanced SQL – 5 queries
- PL/SQL – 5 queries/programs
- Complete database implementation demonstration
- Every team member should participate and explain their work

This implementation follows the finalized DA1 schema.

## 2. Finalized Tables

| Table | Primary Key |
|---|---|
| SUPPLIER | Supplier_ID |
| MEDICINE | Medicine_ID |
| CUSTOMER | Customer_ID |
| EMPLOYEE | Employee_ID |
| PRESCRIPTION | Prescription_ID |
| SALE | Sale_ID |
| SALE_ITEM | (Sale_ID, Medicine_ID) |

Relationships:
- Supplier 1:M Medicine
- Customer 1:M Prescription
- Customer 1:M Sale
- Employee 1:M Sale
- Sale 1:M Sale_Item
- Medicine 1:M Sale_Item

---

# 3. Table Creation

Run parent tables before child tables.

## SUPPLIER

```sql
CREATE TABLE Supplier (
    Supplier_ID NUMBER(5) PRIMARY KEY,
    Supplier_Name VARCHAR2(50) NOT NULL,
    Phone VARCHAR2(15)
);
```

## CUSTOMER

```sql
CREATE TABLE Customer (
    Customer_ID NUMBER(5) PRIMARY KEY,
    Customer_Name VARCHAR2(50) NOT NULL,
    Phone VARCHAR2(15)
);
```

## EMPLOYEE

```sql
CREATE TABLE Employee (
    Employee_ID NUMBER(5) PRIMARY KEY,
    Employee_Name VARCHAR2(50) NOT NULL,
    Phone VARCHAR2(15)
);
```

## MEDICINE

```sql
CREATE TABLE Medicine (
    Medicine_ID NUMBER(5) PRIMARY KEY,
    Medicine_Name VARCHAR2(60) NOT NULL,
    Category VARCHAR2(30),
    Price NUMBER(10,2) CHECK (Price >= 0),
    Stock NUMBER(6) CHECK (Stock >= 0),
    Expiry_Date DATE,
    Supplier_ID NUMBER(5),
    FOREIGN KEY (Supplier_ID) REFERENCES Supplier(Supplier_ID)
);
```

## PRESCRIPTION

```sql
CREATE TABLE Prescription (
    Prescription_ID NUMBER(5) PRIMARY KEY,
    Prescription_Date DATE NOT NULL,
    Doctor_Name VARCHAR2(60),
    Customer_ID NUMBER(5) NOT NULL,
    FOREIGN KEY (Customer_ID) REFERENCES Customer(Customer_ID)
);
```

## SALE

```sql
CREATE TABLE Sale (
    Sale_ID NUMBER(5) PRIMARY KEY,
    Sale_Date DATE NOT NULL,
    Customer_ID NUMBER(5) NOT NULL,
    Employee_ID NUMBER(5) NOT NULL,
    Payment_Mode VARCHAR2(20),
    Payment_Status VARCHAR2(20),
    FOREIGN KEY (Customer_ID) REFERENCES Customer(Customer_ID),
    FOREIGN KEY (Employee_ID) REFERENCES Employee(Employee_ID)
);
```

## SALE_ITEM

```sql
CREATE TABLE Sale_Item (
    Sale_ID NUMBER(5),
    Medicine_ID NUMBER(5),
    Quantity NUMBER(6) CHECK (Quantity > 0),
    PRIMARY KEY (Sale_ID, Medicine_ID),
    FOREIGN KEY (Sale_ID) REFERENCES Sale(Sale_ID),
    FOREIGN KEY (Medicine_ID) REFERENCES Medicine(Medicine_ID)
);
```

---

# 4. Inserting Values

## Suppliers

```sql
INSERT INTO Supplier VALUES (101, 'Apollo Suppliers', '9876543210');
INSERT INTO Supplier VALUES (102, 'MedPlus Distributors', '9876543211');
```

## Customers

```sql
INSERT INTO Customer VALUES (201, 'Rahul Sharma', '9876500001');
INSERT INTO Customer VALUES (202, 'Ananya Rao', '9876500002');
```

## Employees

```sql
INSERT INTO Employee VALUES (301, 'Arjun Kumar', '9876500011');
INSERT INTO Employee VALUES (302, 'Priya Singh', '9876500012');
```

## Medicines

```sql
INSERT INTO Medicine
VALUES (401, 'Paracetamol', 'Analgesic', 25.00, 100, DATE '2027-12-31', 101);

INSERT INTO Medicine
VALUES (402, 'Amoxicillin', 'Antibiotic', 120.00, 50, DATE '2027-08-31', 102);

INSERT INTO Medicine
VALUES (403, 'Cetirizine', 'Antihistamine', 45.00, 75, DATE '2028-03-31', 101);
```

## Prescriptions

```sql
INSERT INTO Prescription
VALUES (501, DATE '2026-09-10', 'Dr. Mehta', 201);

INSERT INTO Prescription
VALUES (502, DATE '2026-09-11', 'Dr. Rao', 202);
```

## Sales

```sql
INSERT INTO Sale
VALUES (601, DATE '2026-09-15', 201, 301, 'Cash', 'Paid');

INSERT INTO Sale
VALUES (602, DATE '2026-09-16', 202, 302, 'UPI', 'Paid');
```

## Sale Items

```sql
INSERT INTO Sale_Item VALUES (601, 401, 2);
INSERT INTO Sale_Item VALUES (601, 403, 1);
INSERT INTO Sale_Item VALUES (602, 402, 2);

COMMIT;
```

---

# 5. Basic Verification

```sql
SELECT * FROM Supplier;
SELECT * FROM Medicine;
SELECT * FROM Customer;
SELECT * FROM Employee;
SELECT * FROM Prescription;
SELECT * FROM Sale;
SELECT * FROM Sale_Item;
```

---

# 6. Advanced SQL – 5 Queries

## Query 1 – Medicine and Supplier JOIN

**Question:** Display medicine details along with supplier details.

```sql
SELECT
    m.Medicine_ID,
    m.Medicine_Name,
    m.Category,
    m.Price,
    s.Supplier_Name,
    s.Phone
FROM Medicine m
JOIN Supplier s
    ON m.Supplier_ID = s.Supplier_ID;
```

**Concept:** INNER JOIN, aliases, related tables.

## Query 2 – Complete Sale Details Using Multiple JOINs

**Question:** Display sale ID, customer, employee, medicine and quantity.

```sql
SELECT
    s.Sale_ID,
    c.Customer_Name,
    e.Employee_Name,
    m.Medicine_Name,
    si.Quantity
FROM Sale s
JOIN Customer c
    ON s.Customer_ID = c.Customer_ID
JOIN Employee e
    ON s.Employee_ID = e.Employee_ID
JOIN Sale_Item si
    ON s.Sale_ID = si.Sale_ID
JOIN Medicine m
    ON si.Medicine_ID = m.Medicine_ID;
```

**Concept:** Multiple-table JOIN.

## Query 3 – Category-wise Statistics

**Question:** Display the number of medicines and average price in each category.

```sql
SELECT
    Category,
    COUNT(*) AS Medicine_Count,
    ROUND(AVG(Price), 2) AS Average_Price
FROM Medicine
GROUP BY Category
ORDER BY Average_Price DESC;
```

**Concept:** GROUP BY, COUNT, AVG, ROUND, ORDER BY.

## Query 4 – Medicines Above Average Price

**Question:** Display medicines whose price is greater than the overall average price.

```sql
SELECT
    Medicine_ID,
    Medicine_Name,
    Price
FROM Medicine
WHERE Price > (
    SELECT AVG(Price)
    FROM Medicine
)
ORDER BY Price DESC;
```

**Concept:** Subquery and aggregate function.

## Query 5 – Customer Sale Summary

**Question:** Display customers and their number of sales.

```sql
SELECT
    c.Customer_ID,
    c.Customer_Name,
    COUNT(s.Sale_ID) AS Total_Sales
FROM Customer c
JOIN Sale s
    ON c.Customer_ID = s.Customer_ID
GROUP BY
    c.Customer_ID,
    c.Customer_Name
HAVING COUNT(s.Sale_ID) >= 1
ORDER BY Total_Sales DESC;
```

**Concept:** JOIN, GROUP BY, COUNT, HAVING, ORDER BY.

---

# 7. Additional SQL for Demonstration

## Low Stock Medicines

```sql
SELECT Medicine_ID, Medicine_Name, Stock
FROM Medicine
WHERE Stock < 60;
```

## Customer Prescription Details

```sql
SELECT
    c.Customer_Name,
    p.Prescription_ID,
    p.Prescription_Date,
    p.Doctor_Name
FROM Customer c
JOIN Prescription p
    ON c.Customer_ID = p.Customer_ID;
```

## Total Quantity Sold per Medicine

```sql
SELECT
    m.Medicine_Name,
    SUM(si.Quantity) AS Total_Quantity_Sold
FROM Medicine m
JOIN Sale_Item si
    ON m.Medicine_ID = si.Medicine_ID
GROUP BY m.Medicine_ID, m.Medicine_Name
ORDER BY Total_Quantity_Sold DESC;
```

---

# 8. PL/SQL – 5 Programs

Enable output first:

```sql
SET SERVEROUTPUT ON;
```

## Program 1 – Anonymous Block

**Question:** Count the total number of medicines.

```sql
DECLARE
    v_count NUMBER;
BEGIN
    SELECT COUNT(*)
    INTO v_count
    FROM Medicine;

    DBMS_OUTPUT.PUT_LINE(
        'Total number of medicines: ' || v_count
    );
END;
/
```

**Concepts:** DECLARE, variable, SELECT INTO, DBMS_OUTPUT.

---

## Program 2 – Stored Procedure

**Question:** Display stock of a medicine using its ID.

```sql
CREATE OR REPLACE PROCEDURE Show_Medicine_Stock (
    p_medicine_id IN Medicine.Medicine_ID%TYPE
)
IS
    v_name  Medicine.Medicine_Name%TYPE;
    v_stock Medicine.Stock%TYPE;
BEGIN
    SELECT Medicine_Name, Stock
    INTO v_name, v_stock
    FROM Medicine
    WHERE Medicine_ID = p_medicine_id;

    DBMS_OUTPUT.PUT_LINE('Medicine: ' || v_name);
    DBMS_OUTPUT.PUT_LINE('Stock: ' || v_stock);

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Medicine not found.');
END;
/
```

Execute:

```sql
EXEC Show_Medicine_Stock(401);
```

**Concepts:** Procedure, IN parameter, `%TYPE`, SELECT INTO, exception handling.

---

## Program 3 – Stored Function

**Question:** Return the price of a medicine.

```sql
CREATE OR REPLACE FUNCTION Get_Medicine_Price (
    p_medicine_id IN Medicine.Medicine_ID%TYPE
)
RETURN NUMBER
IS
    v_price Medicine.Price%TYPE;
BEGIN
    SELECT Price
    INTO v_price
    FROM Medicine
    WHERE Medicine_ID = p_medicine_id;

    RETURN v_price;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN NULL;
END;
/
```

Execute:

```sql
SELECT
    Medicine_Name,
    Get_Medicine_Price(Medicine_ID) AS Price
FROM Medicine;
```

**Concepts:** Function, parameter, RETURN, `%TYPE`, SELECT INTO.

---

## Program 4 – Explicit Cursor

**Question:** Display medicines whose stock is below 80.

```sql
DECLARE
    CURSOR c_low_stock IS
        SELECT Medicine_ID, Medicine_Name, Stock
        FROM Medicine
        WHERE Stock < 80;

    v_id    Medicine.Medicine_ID%TYPE;
    v_name  Medicine.Medicine_Name%TYPE;
    v_stock Medicine.Stock%TYPE;
BEGIN
    OPEN c_low_stock;

    LOOP
        FETCH c_low_stock INTO v_id, v_name, v_stock;

        EXIT WHEN c_low_stock%NOTFOUND;

        DBMS_OUTPUT.PUT_LINE(
            'ID: ' || v_id ||
            ' | Medicine: ' || v_name ||
            ' | Stock: ' || v_stock
        );
    END LOOP;

    CLOSE c_low_stock;
END;
/
```

**Concepts:** Cursor, OPEN, FETCH, `%NOTFOUND`, CLOSE, LOOP.

---

## Program 5 – Trigger

**Question:** Prevent insertion or update of a sale item with quantity <= 0.

```sql
CREATE OR REPLACE TRIGGER Check_Sale_Item_Quantity
BEFORE INSERT OR UPDATE OF Quantity
ON Sale_Item
FOR EACH ROW
BEGIN
    IF :NEW.Quantity <= 0 THEN
        RAISE_APPLICATION_ERROR(
            -20001,
            'Quantity must be greater than zero.'
        );
    END IF;
END;
/
```

Test:

```sql
INSERT INTO Sale_Item
VALUES (601, 402, 0);
```

Expected result:

```text
ORA-20001: Quantity must be greater than zero.
```

**Concepts:** BEFORE trigger, row-level trigger, `:NEW`, RAISE_APPLICATION_ERROR.

> `Quantity > 0` is also enforced by the CHECK constraint. The trigger is included to demonstrate PL/SQL trigger functionality.

---

# 9. Complete Execution Order

```text
1. CREATE Supplier
2. CREATE Customer
3. CREATE Employee
4. CREATE Medicine
5. CREATE Prescription
6. CREATE Sale
7. CREATE Sale_Item

8. INSERT Supplier
9. INSERT Customer
10. INSERT Employee
11. INSERT Medicine
12. INSERT Prescription
13. INSERT Sale
14. INSERT Sale_Item
15. COMMIT

16. Verify all tables
17. Run Advanced SQL 1
18. Run Advanced SQL 2
19. Run Advanced SQL 3
20. Run Advanced SQL 4
21. Run Advanced SQL 5

22. SET SERVEROUTPUT ON
23. Run Anonymous Block
24. Create/execute Procedure
25. Create/execute Function
26. Run Explicit Cursor
27. Create/test Trigger
```

---

# 10. Constraint Summary

| Constraint | Tables | Purpose |
|---|---|---|
| PRIMARY KEY | All main tables | Unique identification |
| Composite PK | SALE_ITEM | Unique sale-medicine combination |
| FOREIGN KEY | MEDICINE | Supplier relationship |
| FOREIGN KEY | PRESCRIPTION | Customer relationship |
| FOREIGN KEY | SALE | Customer/Employee relationships |
| FOREIGN KEY | SALE_ITEM | Sale/Medicine relationships |
| NOT NULL | Required attributes | Prevents NULL |
| CHECK | Price, Stock, Quantity | Validates values |

---

# 11. Viva Preparation

### What is the primary key of SALE_ITEM?

`(Sale_ID, Medicine_ID)` — a composite primary key.

### Why is SALE_ITEM needed?

A sale can contain multiple medicines, and a medicine can appear in many sales. SALE_ITEM represents this relationship and stores the quantity.

### What is a foreign key?

A column that references a primary/candidate key in another table and helps maintain referential integrity.

### WHERE vs HAVING?

`WHERE` filters rows before grouping. `HAVING` filters groups after `GROUP BY`.

### What is a JOIN?

A JOIN combines related rows from multiple tables.

### What is a subquery?

A query nested inside another SQL query.

### What is PL/SQL?

Oracle's procedural extension of SQL, supporting variables, conditions, loops, procedures, functions, cursors and exception handling.

### Procedure vs Function?

A procedure performs an operation and does not have to return a value. A function must return a value.

### What is a cursor?

A cursor lets PL/SQL process query results row by row.

### What is a trigger?

A stored PL/SQL program that executes automatically when a specified database event occurs.

### What is `:NEW`?

It represents the new column value in an INSERT or UPDATE trigger.

### What is `%TYPE`?

It allows a PL/SQL variable or parameter to use the datatype of a database column.

### What is `SELECT INTO`?

It retrieves query output into PL/SQL variables.

### Why use COMMIT?

To permanently save transaction changes.

---

# 12. DA2 Checklist

## Table Creation
- [x] 7 tables created
- [x] Primary keys
- [x] Composite primary key
- [x] Foreign keys
- [x] NOT NULL constraints
- [x] CHECK constraints

## Data
- [x] Supplier values
- [x] Medicine values
- [x] Customer values
- [x] Employee values
- [x] Prescription values
- [x] Sale values
- [x] Sale_Item values
- [x] COMMIT

## Advanced SQL – 5
- [x] JOIN
- [x] Multiple-table JOIN
- [x] GROUP BY + aggregate functions
- [x] Subquery
- [x] GROUP BY + HAVING

## PL/SQL – 5
- [x] Anonymous block
- [x] Procedure
- [x] Function
- [x] Explicit cursor
- [x] Trigger

## Complete Demonstration
- [x] Table creation
- [x] Insertion
- [x] Constraints
- [x] Advanced SQL
- [x] PL/SQL
- [x] Trigger test
- [x] Database relationships

---

# 13. Final Review Summary

The Pharmacy Management System DA2 implementation contains:

```text
7 Tables
        ↓
Primary + Composite Keys
        ↓
Foreign Keys + NOT NULL + CHECK Constraints
        ↓
Sample Data
        ↓
5 Advanced SQL Queries
        ↓
5 PL/SQL Programs
        ↓
Complete Database Demonstration
```

The implementation remains consistent with the finalized DA1 Pharmacy Management System schema.
