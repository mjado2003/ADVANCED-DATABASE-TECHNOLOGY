ADVANCED DB FINAL EXAM USED CODES

223028000 MUSONERA JEAN DE DIEU





A1: Fragment & Recombine Main Fact
1. Table Creation on Node A and Node B
sql
CREATE TABLE Expense_A (
  Expense_ID INT PRIMARY KEY,
  Description VARCHAR(100),
  Amount DECIMAL(10,2)
);

This creates a table Expense_A on Node A to store expense records. Each row has:

Expense_ID: unique identifier

Description: what the expense is for

Amount: cost, with 2 decimal places


CREATE TABLE Expense_B (
  Expense_ID INT PRIMARY KEY,
  Description VARCHAR(100),
  Amount DECIMAL(10,2)
);
Same structure as Expense_A, but on Node B. 
This sets up a distributed system with two fragments.

2. Inserting Data into Expense Tables
sql

INSERT INTO Expense_A VALUES (1, 'Travel', 120.50);
INSERT INTO Expense_A VALUES (2, 'Meals', 45.00);
INSERT INTO Expense_A VALUES (3, 'Supplies', 78.90);
INSERT INTO Expense_A VALUES (4, 'Taxi', 33.25);
INSERT INTO Expense_A VALUES (5, 'Hotel', 200.00);
COMMIT;
Adds five expense records to Expense_A and commits the transaction.

INSERT INTO Expense_B VALUES (6, 'Conference', 150.00);
INSERT INTO Expense_B VALUES (7, 'Flight', 300.00);
INSERT INTO Expense_B VALUES (8, 'Dinner', 60.00);
INSERT INTO Expense_B VALUES (9, 'Stationery', 25.75);
INSERT INTO Expense_B VALUES (10, 'Snacks', 15.00);
COMMIT;
Adds five records to Expense_B and commits

3. Create View on Node A
sql
CREATE VIEW Expense_ALL AS
SELECT * FROM Expense_A
UNION ALL
SELECT * FROM Expense_B@branch_b_link;
Creates a unified view Expense_ALL that combines 
both tables. UNION ALL keeps duplicates and improves performance. 
@branch_b_link accesses Node B remotely.

4. Count and Checksum Queries
sql
SELECT COUNT(*) FROM Expense_A; -- Should return 5
SELECT COUNT(*) FROM Expense_B@branch_b_link; -- Should return 5
SELECT COUNT(*) FROM Expense_ALL;
Counts rows in each fragment and the combined view.


SELECT SUM(MOD(Expense_ID, 97)) FROM Expense_A;
SELECT SUM(MOD(Expense_ID, 97)) FROM Expense_B@branch_b_link;
SELECT SUM(MOD(Expense_ID, 97)) FROM Expense_ALL;
Calculates checksums using modulo 97 to verify data integrity across 
fragments and the view.




A2: Database Link & Cross-Node Join
1. Create a Database Link
sql
CREATE DATABASE LINK proj_link
CONNECT TO branchdb_b IDENTIFIED BY "12345"
USING 'XEPDB1';
This sets up a database link named proj_link from Node A to Node B. 
It allows Node A to run SQL queries on Node B's tables.

branchdb_b: remote user

"12345": password

'XEPDB1': service name of the remote database


2. Remote SELECT via Link
sql
SELECT * FROM expense@proj_link
FETCH FIRST 5 ROWS ONLY;
This fetches up to 5 rows from the expense table located on Node B using the proj_link. 
It’s a remote query — executed from Node A but retrieving data from Node B.

3. Distributed Join
sql
SELECT e.Expense_ID, e.Description, e.Amount, d.Donor_Name, d.Donation_Amount
FROM Expense_A e
CROSS JOIN branchdb_b.Donor@proj_link d
WHERE e.Amount > 100
AND d.Donation_Amount >= 500
AND d.Donation_Date >= DATE '2025-01-01'
FETCH FIRST 10 ROWS ONLY;
This performs a cross-node join between:

Expense_A on Node A

Donor table on Node B via proj_link

It filters:

Expenses over 100

Donations over 500 made after Jan 1, 2025

Returns up to 10 matching combinations

A3: Parallel vs Serial Aggregation
1. Serial Aggregation
sql
SELECT Description, SUM(Amount) AS Total_Amount
FROM Expense_ALL
GROUP BY Description;
This query performs a serial aggregation — meaning it runs in a single-threaded mode. 
It groups expenses by their description and calculates the total amount for each category.

2. Execution Plan for Serial Aggregation
sql
EXPLAIN PLAN FOR
SELECT Description, SUM(Amount) AS Total_Amount
FROM Expense_ALL
GROUP BY Description;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
This shows how Oracle will execute the serial aggregation query. EXPLAIN PLAN generates
 the plan, and DBMS_XPLAN.DISPLAY displays it. It helps you understand performance and 
 resource usage.
Processes the query in a single thread.

Oracle reads data sequentially, using one CPU core.

Suitable for small datasets or low system load.

3. Parallel Aggregation
sql
SELECT /*+ PARALLEL(Expense_A,8) PARALLEL(Expense_B,8) */
Description, SUM(Amount) AS Total_Amount
FROM Expense_ALL
GROUP BY Description;
This query uses Oracle's parallel hint to split the workload across multiple threads 
(8 for each table). It speeds up aggregation, especially for large datasets.
Splits the query across multiple threads or CPU cores.

Each thread processes a portion of the data simultaneously.

Ideal for large datasets or performance-critical tasks.

4. Execution Plan for Parallel Aggregation
sql
EXPLAIN PLAN FOR
SELECT /*+ PARALLEL(Expense_A,8) PARALLEL(Expense_B,8) */
Description, SUM(Amount) AS Total_Amount
FROM Expense_ALL
GROUP BY Description;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
Same as before, but now shows the execution plan for the parallel version. 
You’ll see operations like PX COORDINATOR, PX SEND, and PX RECEIVE, 
which indicate parallel execution.

5. Autotrace for Serial and Parallel
sql
SET AUTOTRACE ON STATISTICS
SELECT Description, SUM(Amount) AS Total_Amount
FROM Expense_ALL
GROUP BY Description;
This runs the serial query and shows execution statistics like CPU time, I/O, and 
rows processed.

sql
SET AUTOTRACE ON STATISTICS
SELECT /*+ PARALLEL(Expense_A,8) PARALLEL(Expense_B,8) */
Description, SUM(Amount) AS Total_Amount
FROM Expense_ALL
GROUP BY Description;
Same for the parallel query — lets you compare performance between serial and 
parallel modes.

A4. Two-Phase Commit & Recovery
This section demonstrates distributed transaction control using Oracle’s two-phase commit protocol.

1. PL/SQL Block for Distributed Insert
sql
BEGIN
  INSERT INTO Expense_A VALUES (11, 'Local Insert', 250.00);
  INSERT INTO Expense_B@proj_link VALUES (12, 'Remote Insert', 300.00);
  COMMIT;
END;
This block inserts:

One row into Expense_A (local)

One row into Expense_B via proj_link (remote)

Then commits both together. Oracle uses two-phase commit to 
ensure atomicity — either both succeed or both fail.

2. Simulate Failure to Create In-Doubt Transaction
sql
BEGIN
  INSERT INTO Expense_A VALUES (13, 'Local Insert Fail', 400.00);
  -- Simulate failure: disable link or shut down remote DB
  INSERT INTO Expense_B@proj_link VALUES (14, 'Remote Insert Fail', 500.00);
  COMMIT;
END;
This block is designed to fail during the remote insert. If the remote node is
 unreachable, Oracle marks the transaction as in-doubt — pending resolution.
I was not able to simulate fail in Oracle Sql Dev because data are inserted automatically,
 to prove this concept, It is better to use PostGre sql 

3. Check Pending Distributed Transactions
sql
SELECT LOCAL_TRAN_ID, GLOBAL_TRAN_ID, STATE
FROM DBA_2PC_PENDING;
This query lists all unresolved distributed transactions. You’ll see entries like:

STATE = prepared

LOCAL_TRAN_ID = 1.23.456

In My case not Transaction was found 

4. Force Resolution
sql
COMMIT FORCE '1.23.456';  -- or ROLLBACK FORCE '1.23.456';
This manually resolves the in-doubt transaction using its ID.
 Use COMMIT FORCE to finalize or ROLLBACK FORCE to cancel.

This was suppose to rollback failed transaction but I don't have one

5. Final Consistency Check
sql
SELECT * FROM Expense_A WHERE Expense_ID IN (11, 13);
SELECT * FROM Expense_B@proj_link WHERE Expense_ID IN (12, 14);
This confirms which rows were successfully committed and which were rolled back.

 A5: Distributed Lock Conflict & Diagnosis
This section simulates a row-level lock conflict across nodes.

1. Session 1: Lock a Row on Node A
sql
-- Session 1
UPDATE Expense_A SET Amount = 999.99 WHERE Expense_ID = 11;
-- Do NOT commit yet — keep transaction open
This locks row Expense_ID = 11 on Node A.

2. Session 2: Try to Update Same Row from Node B
sql
-- Session 2 (via link)
UPDATE Expense_A@proj_link SET Amount = 888.88 WHERE Expense_ID = 11;
This session will hang — waiting for Session 1 to release the lock.

3. Diagnose Lock Conflict
sql
SELECT * FROM DBA_BLOCKERS;
SELECT * FROM DBA_WAITERS;
SELECT * FROM V$LOCK;
These views show:

Which session is blocking

Which session is waiting

Lock types and status

4. Release Lock and Confirm Completion
sql
-- Session 1
COMMIT;
Once committed, Session 2 proceeds. You can confirm with timestamps or logs

B6: Declarative Rules Hardening
This section enforces data integrity using constraints and tests them 
with passing and failing inserts.

1. Add Constraints
sql
ALTER TABLE Expense_A
ADD CONSTRAINT chk_amount_positive CHECK (Amount > 0);

ALTER TABLE Expense_A
MODIFY Description NOT NULL;
Adds a check constraint to ensure Amount is always positive.

Modifies Description to be mandatory (NOT NULL).

2. Test Inserts
sql
-- Passing inserts
INSERT INTO Expense_A VALUES (15, 'Valid Expense', 100.00);
INSERT INTO Expense_A VALUES (16, 'Another Valid', 250.00);
COMMIT;
These meet all constraints and are committed.

sql
-- Failing inserts
BEGIN
  INSERT INTO Expense_A VALUES (17, NULL, 300.00); -- NULL description
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('FAILED: ' || SQLERRM);
    ROLLBACK;
END;

BEGIN
  INSERT INTO Expense_A VALUES (18, 'Negative Expense', -50.00); -- Negative amount
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('FAILED: ' || SQLERRM);
    ROLLBACK;
END;
These violate constraints and are rolled back.

 B7: E–C–A Trigger for Denormalized Totals
This section uses a trigger to maintain totals and log changes.

1. Audit Table
sql
CREATE TABLE Expense_AUDIT (
  bef_total NUMBER,
  aft_total NUMBER,
  changed_at TIMESTAMP,
  key_col VARCHAR2(64)
);
Stores before/after totals, timestamp, and affected key.

2. Trigger
sql
CREATE OR REPLACE TRIGGER trg_expense_total
AFTER INSERT OR UPDATE OR DELETE ON Expense_A
DECLARE
  v_before NUMBER;
  v_after NUMBER;
BEGIN
  SELECT NVL(SUM(Amount), 0) INTO v_before FROM Expense_A;

  -- Simulate change
  -- (actual logic may vary depending on event type)

  SELECT NVL(SUM(Amount), 0) INTO v_after FROM Expense_A;

  INSERT INTO Expense_AUDIT VALUES (v_before, v_after, SYSTIMESTAMP, 'Expense_A');
END;
Logs total before and after any change to Expense_A.

3. Test DML
sql
INSERT INTO Expense_A VALUES (19, 'Trigger Test', 300.00);
UPDATE Expense_A SET Amount = Amount + 50 WHERE Expense_ID = 19;
DELETE FROM Expense_A WHERE Expense_ID = 19;
Each action triggers the audit log.

4. 
Check current expenses for P001: 
SELECT * FROM Expense_a WHERE ProjectCode = 'P001';

Check updated budget in Project: 
SELECT ProjectCode, Budget FROM Project WHERE ProjectCode = 'P001';

B8: Recursive Hierarchy Roll-Up
This section builds a hierarchy and rolls up values using recursion.

1. Hierarchy Table
sql
CREATE TABLE HIER (
  parent_id INT,
  child_id INT
);
Stores parent-child relationships.

2. Insert Hierarchy
sql
INSERT INTO HIER VALUES (NULL, 1);
INSERT INTO HIER VALUES (1, 2);
INSERT INTO HIER VALUES (2, 3);
-- Forms a 3-level hierarchy: 1 → 2 → 3
3. Recursive Query
sql
Join to Expense_a and Compute Rollups: 
WITH hierarchy (child_id, root_id, depth) AS ( -- Anchor: root nodes 
SELECT child_id, child_id AS root_id, 0 AS depth 
FROM HIER 
WHERE parent_id IS NULL 
UNION ALL -- Recursive step 
SELECT h.child_id, hr.root_id, hr.depth + 1 
FROM HIER h 
JOIN hierarchy hr ON h.parent_id = hr.child_id 
) 
SELECT * FROM hierarchy; 

to build a recursive hierarchy and then aggregate expenses by node
 
AFTER hierarchy (child_id, AS id, depth) AS (
  SELECT child_id, child_id AS parent_id, 0 AS depth
  FROM Expense_a
  UNION ALL
  SELECT parent_id, NULL
  UNION ALL
  SELECT child_id, AS id, depth + 1
  FROM AFTER hierarchy AS h INNER_id = b.child_id
  FROM SITE_b
  JOIN hierarchy AS hp ON b.parent_id = hp.child_id
)

SELECT id,
SUM(Amount) AS total_expense
FROM Expense_a AS e
JOIN Expense_a AS total_expense
GROUP BY id
ORDER BY id;

B9: Mini-Knowledge Base with Transitive Inference
This section models simple logic using triples and recursive inference.

1. Triple Table
sql
CREATE TABLE TRIPLE (
  s VARCHAR2(64),
  p VARCHAR2(64),
  o VARCHAR2(64)
);
Stores facts in subject–predicate–object format.

2. Insert Facts
sql
INSERT INTO TRIPLE VALUES ('A', 'isA', 'B');
INSERT INTO TRIPLE VALUES ('B', 'isA', 'C');
-- Implies: A isA C (transitive)
3. Recursive Inference
sql
WITH INFER AS (
  SELECT s, o FROM TRIPLE WHERE p = 'isA'
  UNION ALL
  SELECT i.s, t.o
  FROM INFER i
  JOIN TRIPLE t ON i.o = t.s AND t.p = 'isA'
)
SELECT * FROM INFER;
Infers transitive relationships like A isA C.

 B10: Business Limit Alert (Function + Trigger)
This section enforces a business rule using a function and trigger.

1. Rule Table
sql
CREATE TABLE BUSINESS_LIMITS (
  rule_key VARCHAR2(64),
  threshold NUMBER,
  active CHAR(1) CHECK (active IN ('Y', 'N'))
);

Stores rules like max expense per project.

2. Function
sql
CREATE OR REPLACE FUNCTION fn_should_alert(p_projectcode VARCHAR2, p_new_amount NUMBER)
RETURN NUMBER
IS
  v_threshold NUMBER;
  v_total NUMBER;
BEGIN
  SELECT MAX(threshold) INTO v_threshold
  FROM BUSINESS_LIMITS
  WHERE rule_key = 'MAX_PROJECT_EXPENSE' AND active = 'Y';

  SELECT NVL(SUM(amount), 0) INTO v_total
  FROM expense_a
  WHERE projectcode = p_projectcode;

  IF v_total + p_new_amount > v_threshold THEN
    RETURN 1;
  ELSE
    RETURN 0;
  END IF;
END;
Returns 1 if new amount would exceed threshold.

3. Trigger
sql
CREATE OR REPLACE TRIGGER trg_expense_alert
BEFORE INSERT OR UPDATE ON expense_a
FOR EACH ROW
DECLARE
  v_alert NUMBER;
BEGIN
  v_alert := fn_should_alert(:NEW.projectcode, :NEW.amount);
  IF v_alert = 1 THEN
    RAISE_APPLICATION_ERROR(-20001, 'Expense threshold exceeded for project: ' || :NEW.projectcode);
  END IF;
END;
Blocks inserts/updates that violate the rule.

d. Test Cases
sql
-- Passing
INSERT INTO expense_a VALUES (311, 'Design Services', 1200, 'P006');

-- Failing
BEGIN
  INSERT INTO expense_a VALUES (313, 'Premium Booth Upgrade', 2500, 'P006');
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('FAILED: ' || SQLERRM);
    ROLLBACK;
END;



Passing inserts are committed; failing ones are rolled back.