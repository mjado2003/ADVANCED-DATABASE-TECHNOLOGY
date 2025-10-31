Distributed and Parallel Database Lab 
This document explains the implementation of 10 practical tasks for a distributed and parallel database system using Oracle. Each task is aligned with the assessment guide and includes SQL scripts, configuration steps, and explanations.

 Setup: Create Users and Grant Privileges
sql
CREATE USER BranchDB_A IDENTIFIED BY "12345";
CREATE USER BranchDB_B IDENTIFIED BY "12345";

GRANT CONNECT, RESOURCE TO BranchDB_A;
GRANT CONNECT, RESOURCE TO BranchDB_B;

 Schema Creation
sql
-- Common schema (Donor, Project, Expense)
CREATE TABLE Donor (
    DonorID INT PRIMARY KEY,
    FullName VARCHAR2(100),
    Organization VARCHAR2(100),
    Contact VARCHAR2(50),
    Email VARCHAR2(100)
);

CREATE TABLE Project (
    ProjectID INT PRIMARY KEY,
    DonorID INT REFERENCES Donor(DonorID),
    Title VARCHAR2(150),
    Budget NUMBER(12,2),
    StartDate DATE,
    EndDate DATE
);

CREATE TABLE Expense (
    ExpenseID INT PRIMARY KEY,
    ProjectID INT,
    Description CLOB,
    Cost NUMBER(10,2),
    DateIncurred DATE
);

 a. Vertical Fragmentation
sql
-- Fragment 1: Identity info
CREATE TABLE Staff_Identity (
  StaffID     NUMBER PRIMARY KEY,
  FullName    VARCHAR2(100),
  Role        VARCHAR2(50)
);

-- Fragment 2: Contact and assignment
CREATE TABLE Staff_Contact (
  StaffID     NUMBER PRIMARY KEY,
  Contact     VARCHAR2(100),
  ProjectID   NUMBER
);

b. Horizontal Fragmentation
sql
-- Assume Region is added to Staff table
-- West region staff (stored in Branch B)
INSERT INTO Staff (StaffID, FullName, Role, Contact, ProjectID, Region)
VALUES (1011, 'Claudine Mukamana', 'Finance Analyst', 'claudine.m@example.com', 102, 'West');

INSERT INTO Staff (StaffID, FullName, Role, Contact, ProjectID, Region)
VALUES (112, 'Eric Niyonzima', 'Monitoring Officer', 'eric.niyonzima@example.com', 102, 'West');

Materialized View on Branch A
sql
CREATE MATERIALIZED VIEW Staff_East AS
SELECT StaffID, FullName, Role, Contact, ProjectID, Region
FROM Staff@branch_b_link
WHERE Region = 'East';

 Reconstruct Full Staff Table
sql
SELECT * FROM Staff_East
UNION ALL
SELECT * FROM Staff@branch_b_link WHERE Region = 'West';

Create Links
sql
-- On Branch 1
CREATE DATABASE LINK branch_b_link
CONNECT TO BranchDB_B IDENTIFIED BY "12345"
USING 'XEPDB1';

-- On Branch 2
CREATE DATABASE LINK branch_a_link
CONNECT TO BranchDB_A IDENTIFIED BY "12345"
USING 'XEPDB1';

 Remote SELECT
sql
SELECT * FROM Report@branchdb_b_link;

 Distributed Join
sql
SELECT 
  d.FullName AS DonorName,
  p.ProjectID,
  p.Title AS ProjectTitle,
  e.ExpenseID,
  e.Cost,
  e.DateIncurred
FROM 
  Project p
JOIN 
  Donor d ON p.DonorID = d.DonorID
JOIN 
  Expense@branch_b_link e ON p.ProjectID = e.ProjectID;


3.1 Serial Execution
sql
-- Generate execution plan for serial query
EXPLAIN PLAN FOR
SELECT * FROM Expense@branch_b_link
WHERE Cost > 1000;

-- View the plan
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Measure execution time
SET TIMING ON;
SELECT * FROM Expense@branch_b_link
WHERE Cost > 1000;
SET TIMING OFF;
3.2 Parallel Execution
sql
-- Generate execution plan with parallel hint
EXPLAIN PLAN FOR
SELECT /*+ PARALLEL(Expense, 8) */ * FROM Expense@branch_b_link
WHERE Cost > 1000;

-- View the plan
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Compare with local table
EXPLAIN PLAN FOR
SELECT * FROM Expense WHERE Cost > 1000;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
Observation: Parallel execution reduces elapsed time and improves throughput, especially for large datasets.
4.
sql
-- Distributed insert with single commit
BEGIN
  INSERT INTO Donor (DonorID, FullName)
  VALUES (1001, 'Global Health Fund');

  INSERT INTO Expense@branch_b_link (ExpenseID, ProjectID, Cost, DateIncurred, Description)
  VALUES (501, 12, 3000, SYSDATE, 'Emergency supplies');

  COMMIT;
END;
/

-- Distributed update with commit
BEGIN
  UPDATE Donor
  SET FullName = 'Global Health Fund International'
  WHERE DonorID = 1001;

  UPDATE Expense@branch_b_link
  SET Cost = 3500,
      Description = 'Emergency medical supplies'
  WHERE ExpenseID = 501;

  COMMIT;
END;
/

-- Check pending distributed transactions
SELECT LOCAL_TRAN_ID, GLOBAL_TRAN_ID, STATE, MIXED, ADVICE
FROM DBA_2PC_PENDING;
 Note: Ensures atomicity across nodes using Oracle’s two-phase commit protocol.

5️. Distributed Rollback and Recovery 
sql
BEGIN
  -- Insert into local and remote tables
  INSERT INTO Donor VALUES (1004, 'Global Health Fund', 'GHF Org', '123456789', 'ghf@example.com');
  INSERT INTO Expense@branch_b_link VALUES (505, 111, 'Emergency supplies', 4000, SYSDATE);

  -- Simulate failure before commit
  DBMS_LOCK.SLEEP(60);  -- Delay to allow manual session kill
  COMMIT;
END;
/

-- Kill session to simulate network failure
ALTER SYSTEM KILL SESSION '278,39323' IMMEDIATE;
Recovery: Use DBA_2PC_PENDING to identify unresolved transactions and ROLLBACK FORCE to resolve them.

6️. Distributed Concurrency Control 
Lock Conflict Analysis
Session 395 (BranchDB_B):

LMODE = 6: Holds exclusive lock.

BLOCK = 1: Blocking other sessions.

Session 28 (BranchDB_A):

REQUEST = 6: Waiting for exclusive lock.

BLOCK = 0: Not blocking others.

 Conclusion: Demonstrates row-level locking and conflict resolution across distributed sessions.

7️. Parallel Data Loading / ETL Simulation 
sql
-- Enable parallel DML
ALTER SESSION ENABLE PARALLEL DML;
ALTER TABLE Expense PARALLEL (DEGREE 4);

-- Create staging table
CREATE TABLE Expense_Staging AS
SELECT * FROM Expense;

-- Parallel insert with transformation
INSERT /*+ PARALLEL(Expense, 4) */ INTO Expense
SELECT /*+ PARALLEL(Expense_Staging, 4) */
  ExpenseID + 1000,
  ProjectID,
  Description || ' [ETL]',
  Cost * 1.05,
  DateIncurred
FROM Expense_Staging;

-- Parallel update
SET TIMING ON;
SET AUTOTRACE ON EXPLAIN;

UPDATE /*+ PARALLEL(Expense, 4) */ Expense
SET
  Cost = ROUND(Cost * 1.08, 2),
  Description = REPLACE(Description, '[ETL]', '[ETL v2]')
WHERE ExpenseID BETWEEN 1000 AND 1999;

COMMIT;

SET AUTOTRACE OFF;
SET TIMING OFF;

ETL Logic:

Adds 1000 to ExpenseID

Appends [ETL] to Description

Increases Cost by 5%, then 8%

8️. Three-Tier Client–Server Architecture (2 Marks)
 Architecture Layers
Presentation Layer: Web/mobile interface

Application Layer: Middleware (e.g., REST API)

Database Layer: BranchDB_A and BranchDB_B with DB links

 Data Flow:

Client sends request to app server.

App server queries distributed databases via DB links.

Results are merged and returned to client.

Include a diagram showing:

Layered architecture

DB link interactions

Data flow arrows

9️. Distributed Query Optimization 
A. Basic Join
sql
EXPLAIN PLAN FOR
SELECT p.Title, SUM(e.Cost) AS TotalCost
FROM Project p
JOIN Expense@branch_b_link e ON p.ProjectID = e.ProjectID
GROUP BY p.Title;
B. Optimized Join
sql
-- Push execution to remote site
SELECT /*+ DRIVING_SITE(e) */ p.Title, SUM(e.Cost)
FROM Project p
JOIN Expense@branch_b_link e ON p.ProjectID = e.ProjectID
GROUP BY p.Title;

-- Apply filter early
SELECT /*+ DRIVING_SITE(e) */ p.Title, SUM(e.Cost) AS TotalCost
FROM Project p
JOIN Expense@branch_b_link e ON p.ProjectID = e.ProjectID
WHERE e.DateIncurred >= DATE '2025-10-01'
GROUP BY p.Title;

-- Indexing for performance
CREATE INDEX idx_project_id ON Project(ProjectID);
CREATE INDEX idx_expense_project_id ON Expense(ProjectID);

-- Materialized view for frequent queries
CREATE MATERIALIZED VIEW mv_expense AS
SELECT ProjectID, Cost, DateIncurred
FROM Expense@branch_b_link
WHERE DateIncurred >= DATE '2025-10-01';

Hints Used:

DRIVING_SITE(e): Executes join remotely

USE_NL(e): Nested loop join for indexed tables

10.Performance Benchmark and Report 
A. Centralized (Simulated via Materialized Views)
sql
CREATE MATERIALIZED VIEW mvb_expense AS
SELECT * FROM Expense@branch_b_link;

CREATE MATERIALIZED VIEW mvb_staff AS
SELECT * FROM Staff@branch_b_link;

-- Centralized query
SET AUTOTRACE ON
SELECT p.Title, SUM(e.Cost), COUNT(s.StaffID)
FROM Project p
JOIN mvb_expense e ON p.ProjectID = e.ProjectID
JOIN mvb_staff s ON p.ProjectID = s.ProjectID
GROUP BY p.Title;
B. Parallel Execution
sql
ALTER TABLE mvb_expense PARALLEL 4;
ALTER TABLE mvb_staff PARALLEL 4;

-- Parallel query
SET AUTOTRACE ON
SELECT /*+ PARALLEL(e) PARALLEL(s) */
       p.Title, SUM(e.Cost), COUNT(s.StaffID)
FROM Project p
JOIN mvb_expense e ON p.ProjectID = e.ProjectID
JOIN mvb_staff s ON p.ProjectID = s.ProjectID
GROUP BY p.Title;
C. Distributed Execution
sql
-- Direct access via DB links
SET AUTOTRACE ON
SELECT /*+ DRIVING_SITE(e) USE_NL(e s) */
       p.Title, SUM(e.Cost), COUNT(s.StaffID)
FROM Project p
JOIN Expense@branch_b_link e ON p.ProjectID = e.ProjectID
JOIN Staff@branch_b_link s ON p.ProjectID = s.ProjectID
GROUP BY p.Title;

Conclusion:

Centralized and parallel modes offer better performance.

Distributed mode incurs higher I/O and latency.

Optimization with hints and materialized views is essential for scalability.