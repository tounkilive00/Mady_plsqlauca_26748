# PL/SQL Collections, Records, and GOTO — Demo Repository

## Overview

This repository demonstrates practical examples of PL/SQL **Collections**, **Records**, and **GOTO** statements. It's designed as a compact learning project you can run in Oracle (SQL*Plus, SQL Developer, or similar). The repository includes SQL scripts, a PL/SQL package with procedures showing each concept, test scripts, and a well-organized README so you and your classmates can be assessed next week.

---

## Learning Objectives

* Understand and use PL/SQL **associative arrays**, **nested tables**, and **VARRAYs**.
* Define and manipulate **RECORD** types and collections of records.
* See an example use-case for the **GOTO** statement (for demonstration only).
* Combine collections and records to process tabular data in PL/SQL.

---

## Repository Structure

```
plsql-demo/
├─ README.md                     <- This document (also viewable here)
├─ create_schema.sql             <- creates sample tables and seed data
├─ pkg_demo_collections.sql      <- package spec & body (main demo code)
├─ run_demo.sql                  <- script to run demo procedures and show output
└─ grading_checklist.md          <- checklist for instructors/assessors
```

---

## Prerequisites

* Oracle Database (any recent release)
* An account with privileges to create tables, packages, and run PL/SQL
* SQL*Plus, SQLcl or SQL Developer to run the scripts

---

## Problem Statement

You are building a lightweight data-processing utility to process orders in memory before writing aggregated results back to the database. The utility should:

1. Load rows from an `orders` table into a PL/SQL collection.
2. Use a `RECORD` type to represent a single order row.
3. Use collections of records (a nested table) to hold and process the loaded orders.
4. Demonstrate three collection types: associative array (index-by), nested table, and VARRAY.
5. Use a `GOTO` statement to bail out of processing early under a specific demonstration condition (e.g., a sentinel `order_amount` value of -1) — purely for demonstration.
6. Produce an aggregated result (total amount per customer) and store it in a DB table.

The repository implements this scenario with clear, commented PL/SQL code and a runnable script.

---

## Schema & Seed Script (`create_schema.sql`)

```sql
-- create_schema.sql
-- Creates tables and seed data used by the demo

DROP TABLE orders_total_agg PURGE;
DROP TABLE orders PURGE;

CREATE TABLE orders (
  order_id      NUMBER PRIMARY KEY,
  customer_id   NUMBER,
  order_date    DATE,
  order_amount  NUMBER
);

CREATE TABLE orders_total_agg (
  customer_id   NUMBER PRIMARY KEY,
  total_amount  NUMBER
);

-- seed data
INSERT INTO orders (order_id, customer_id, order_date, order_amount) VALUES (1, 100, SYSDATE-3, 250);
INSERT INTO orders (order_id, customer_id, order_date, order_amount) VALUES (2, 101, SYSDATE-2, 100);
INSERT INTO orders (order_id, customer_id, order_date, order_amount) VALUES (3, 100, SYSDATE-1, 150);
INSERT INTO orders (order_id, customer_id, order_date, order_amount) VALUES (4, 102, SYSDATE,     75);
-- sentinel example: -1 triggers GOTO demo early exit
INSERT INTO orders (order_id, customer_id, order_date, order_amount) VALUES (5, 103, SYSDATE,    -1);

COMMIT;
```

---

## Package: `pkg_demo_collections.sql`

Contains both the package specification and body. It exposes two demo procedures:

* `process_orders_with_collections` — main demo: loads, processes, aggregates using nested table of RECORDs.
* `demo_varray_and_assoc_array` — shows VARRAY and associative array behavior.

```sql
-- pkg_demo_collections.sql
CREATE OR REPLACE PACKAGE demo_collections AS
  PROCEDURE process_orders_with_collections;
  PROCEDURE demo_varray_and_assoc_array;
END demo_collections;
/

CREATE OR REPLACE PACKAGE BODY demo_collections AS

  -----------------------------------------------------------------
  -- Types used across procedures
  -----------------------------------------------------------------
  TYPE t_order_rec IS RECORD (
    order_id     orders.order_id%TYPE,
    customer_id  orders.customer_id%TYPE,
    order_date   orders.order_date%TYPE,
    order_amount orders.order_amount%TYPE
  );

  TYPE t_order_tab IS TABLE OF t_order_rec; -- nested table of records

  -- VARRAY example: fixed max size list of top N amounts (demo only)
  TYPE t_amount_varray IS VARRAY(5) OF NUMBER;

  -- Associative array (index-by) example: map customer_id -> total_amount
  TYPE t_cust_totals IS TABLE OF NUMBER INDEX BY PLS_INTEGER;

  -----------------------------------------------------------------
  -- Procedure: load orders into nested table (collection of records),
  -- aggregate totals per customer, and store into orders_total_agg.
  -----------------------------------------------------------------
  PROCEDURE process_orders_with_collections IS
    orders_tab   t_order_tab := t_order_tab();
    idx          PLS_INTEGER := 0;
    cust_totals  t_cust_totals; -- associative array
  BEGIN
    -- 1) Load orders into the nested table using BULK COLLECT
    SELECT order_id, customer_id, order_date, order_amount
    BULK COLLECT INTO
      orders_tab
    FROM (
      SELECT order_id, customer_id, order_date, order_amount FROM orders ORDER BY order_id
    );

    DBMS_OUTPUT.PUT_LINE('Loaded ' || orders_tab.COUNT || ' orders into collection.');

    -- 2) Process each item in PL/SQL (using FOR loop over collection)
    FOR i IN 1 .. orders_tab.COUNT LOOP
      DECLARE
        r t_order_rec := orders_tab(i);
      BEGIN
        -- Demonstrate GOTO usage: if sentinel order_amount = -1 then exit to label
        IF r.order_amount = -1 THEN
          DBMS_OUTPUT.PUT_LINE('Sentinel detected at order_id=' || r.order_id || '. Jumping to CLEANUP via GOTO.');
          GOTO CLEANUP;
        END IF;

        -- Aggregate into associative array keyed by customer_id
        IF cust_totals.EXISTS(r.customer_id) THEN
          cust_totals(r.customer_id) := cust_totals(r.customer_id) + NVL(r.order_amount,0);
        ELSE
          cust_totals(r.customer_id) := NVL(r.order_amount,0);
        END IF;
      END;
    END LOOP;

    -- 3) Persist aggregated results into orders_total_agg
    -- simple approach: delete and insert
    DELETE FROM orders_total_agg;

    idx := cust_totals.FIRST;
    WHILE idx IS NOT NULL LOOP
      INSERT INTO orders_total_agg (customer_id, total_amount) VALUES (idx, cust_totals(idx));
      idx := cust_totals.NEXT(idx);
    END LOOP;

    COMMIT;

    DBMS_OUTPUT.PUT_LINE('Aggregation persisted for ' || NVL(cust_totals.COUNT,0) || ' customers.');

  <<CLEANUP>>
    -- label target for GOTO; still allowed to perform cleanup work here.
    NULL; -- can add logging/other cleanup here
  END process_orders_with_collections;

  -----------------------------------------------------------------
  -- Procedure: demo of VARRAY and associative array behaviors
  -----------------------------------------------------------------
  PROCEDURE demo_varray_and_assoc_array IS
    top_amounts  t_amount_varray := t_amount_varray();
    cust_map     t_cust_totals; -- assoc array
    c_id         PLS_INTEGER;
  BEGIN
    -- VARRAY: push a few values (remember: VARRAY max is 5)
    top_amounts.EXTEND; top_amounts(top_amounts.LAST) := 500;
    top_amounts.EXTEND; top_amounts(top_amounts.LAST) := 400;
    top_amounts.EXTEND; top_amounts(top_amounts.LAST) := 300;

    DBMS_OUTPUT.PUT_LINE('VARRAY size: ' || top_amounts.COUNT);

    -- Associative array: demonstrate non-contiguous keys
    cust_map(100) := 400;
    cust_map(200) := 150;
    cust_map(999) := 0; -- existed but zero

    c_id := cust_map.FIRST;
    WHILE c_id IS NOT NULL LOOP
      DBMS_OUTPUT.PUT_LINE('Customer ' || c_id || ' => ' || cust_map(c_id));
      c_id := cust_map.NEXT(c_id);
    END LOOP;
  END demo_varray_and_assoc_array;

END demo_collections;
/
```

> **Note about GOTO**: `GOTO` is generally discouraged in production code because it can make control flow harder to follow. This example uses it only for demonstration (to show how labels and `GOTO` work inside PL/SQL). In real systems, prefer structured control (exceptions, loops, early `RETURN`, etc.).

---

## Run Script (`run_demo.sql`)

```sql
-- run_demo.sql
SET SERVEROUTPUT ON SIZE 1000000

-- Create objects and seed (run create_schema.sql first)
@create_schema.sql

-- Create package
@pkg_demo_collections.sql

-- Run demos
BEGIN
  demo_collections.demo_varray_and_assoc_array;
  demo_collections.process_orders_with_collections;
END;
/

-- Show aggregated results
SELECT * FROM orders_total_agg ORDER BY customer_id;
```

---

## What to hand in for assessment

* Create a fresh repository (e.g., on GitHub) with the file structure above.
* Ensure `create_schema.sql`, `pkg_demo_collections.sql`, and `run_demo.sql` are present and runnable.
* Add a short `grading_checklist.md` (or `grading_checklist.txt`) used by the instructor. A sample checklist is included in this repo.
* Provide instructions to run the demo in `README.md` (this file) and include expected DB output.

---

## Grading Checklist (sample contents — `grading_checklist.md`)

* [ ] Package compiles without errors.
* [ ] Collections (assoc array, nested table, VARRAY) are demonstrated.
* [ ] RECORD type is used to represent table row(s).
* [ ] GOTO usage is present and explained in README.
* [ ] Script runs end-to-end and produces aggregated results in `orders_total_agg`.
* [ ] Documentation is complete and well-organized.

---

## Helpful Hints & Troubleshooting

* If package compilation fails, run `SHOW ERRORS PACKAGE demo_collections;` in SQL*Plus/SQL Developer and fix missing types or references.
* If `BULK COLLECT` returns no rows, confirm `orders` table has data and that `SELECT` is correct.
* When testing GOTO behavior, change or remove the sentinel row (`order_amount = -1`) to see full aggregation.

---

## License

Use freely for classwork. No warranty.

---

*End of document.*
