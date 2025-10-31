Healthcare and Billing SQL Codebase
This document explains the purpose and logic behind each SQL module, with inline comments highlighting corrections, constraints, 
and reasoning strategies. The codebase spans five domains: patient prescriptions, billing triggers, supervision hierarchies, 
semantic triples, and spatial queries.

1️. Patient and Medication Tables
Table Definitions
sql
-- Create patient table
CREATE TABLE PATIENT (
  ID NUMBER PRIMARY KEY,
  NAME VARCHAR2(100) NOT NULL
); 

-- Corrected medication table with constraints
CREATE TABLE PATIENT_MED (
  PATIENT_MED_ID NUMBER PRIMARY KEY, -- unique ID for each medication entry
  PATIENT_ID NUMBER NOT NULL REFERENCES PATIENT(ID), -- foreign key must reference existing patient
  MED_NAME VARCHAR2(80) NOT NULL, -- corrected: added NOT NULL to ensure medication name is mandatory
  DOSE_MG NUMBER(6,2) CHECK (DOSE_MG >= 0), -- corrected: added parentheses around CHECK condition
  START_DT DATE,
  END_DT DATE,
  CONSTRAINT CK_RX_DATES CHECK (
    START_DT IS NULL OR END_DT IS NULL OR START_DT <= END_DT
  ) -- corrected: replaced invalid logic with proper NULL-safe date comparison
);
 Sample Inserts and Constraint Tests
sql
-- Attempt to insert invalid dose (violates CHECK constraint)
INSERT INTO PATIENT_MED VALUES (1, 1, 'Amoxicillin', -50, TO_DATE('2025-10-01','YYYY-MM-DD'), TO_DATE('2025-10-10','YYYY-MM-DD'));

-- Attempt to insert inverted dates (violates CK_RX_DATES constraint)
INSERT INTO PATIENT_MED VALUES (2, 1, 'Ibuprofen', 200, TO_DATE('2025-10-15','YYYY-MM-DD'), TO_DATE('2025-10-10','YYYY-MM-DD'));

-- Insert valid patient
INSERT INTO PATIENT VALUES (1, 'John Doe');

-- Valid prescription with proper dose and date range
INSERT INTO PATIENT_MED VALUES (3, 1, 'Paracetamol', 500, TO_DATE('2025-10-01','YYYY-MM-DD'), TO_DATE('2025-10-05','YYYY-MM-DD'));

-- Valid prescription with NULL dates (allowed by constraint)
INSERT INTO PATIENT_MED VALUES (4, 1, 'Cetirizine', 10, NULL, NULL);

-- View inserted medication records
SELECT * FROM PATIENT_MED;

2️. Billing System with Audit Logging
Table Definitions
sql
-- Main bill table
CREATE TABLE BILL (
  ID NUMBER PRIMARY KEY,
  TOTAL NUMBER(12,2)
);

-- Items linked to bills
CREATE TABLE BILL_ITEM (
  BILL_ID NUMBER,
  AMOUNT NUMBER(12,2),
  UPDATED_AT DATE,
  CONSTRAINT FK_BILL_ITEM_BILL FOREIGN KEY (BILL_ID) REFERENCES BILL(ID)
);

-- Audit log for changes
CREATE TABLE BILL_AUDIT (
  BILL_ID NUMBER,
  OLD_TOTAL NUMBER(12,2),
  NEW_TOTAL NUMBER(12,2),
  CHANGED_AT DATE
);

 Corrected Compound Trigger
sql
-- Corrected trigger: avoids mutating-table error by using statement-level logic
CREATE OR REPLACE TRIGGER TRG_BILL_TOTAL_STMT
AFTER INSERT OR UPDATE OR DELETE ON BILL_ITEM
DECLARE
  TYPE bill_id_table IS TABLE OF BILL_ITEM.BILL_ID%TYPE INDEX BY PLS_INTEGER;
  v_bill_ids bill_id_table;
  v_index PLS_INTEGER := 0;
BEGIN
  -- Collect affected BILL_IDs
  FOR r IN (
    SELECT DISTINCT BILL_ID FROM BILL_ITEM
    WHERE BILL_ID IS NOT NULL
  ) LOOP
    v_index := v_index + 1;
    v_bill_ids(v_index) := r.BILL_ID;
  END LOOP;

  -- Recompute totals and insert audit rows
  FOR i IN 1 .. v_index LOOP
    DECLARE
      v_old_total BILL.TOTAL%TYPE;
      v_new_total BILL.TOTAL%TYPE;
    BEGIN
      SELECT TOTAL INTO v_old_total FROM BILL WHERE ID = v_bill_ids(i);
      SELECT NVL(SUM(AMOUNT), 0) INTO v_new_total FROM BILL_ITEM WHERE BILL_ID = v_bill_ids(i);

      UPDATE BILL SET TOTAL = v_new_total WHERE ID = v_bill_ids(i);

      INSERT INTO BILL_AUDIT (BILL_ID, OLD_TOTAL, NEW_TOTAL, CHANGED_AT)
      VALUES (v_bill_ids(i), v_old_total, v_new_total, SYSDATE);
    END;
  END LOOP;
END;
/

 Sample Transactions
sql
-- Insert bill items (trigger fires)
INSERT INTO BILL_ITEM VALUES (1, 100, SYSDATE);
INSERT INTO BILL_ITEM VALUES (1, 200, SYSDATE);
INSERT INTO BILL_ITEM VALUES (2, 300, SYSDATE);

-- Update an item (trigger fires)
UPDATE BILL_ITEM SET AMOUNT = 150 WHERE BILL_ID = 1 AND AMOUNT = 100;

-- Delete an item (trigger fires)
DELETE FROM BILL_ITEM WHERE BILL_ID = 2 AND AMOUNT = 300;

-- Check updated totals
SELECT * FROM BILL;

-- Check audit trail
SELECT * FROM BILL_AUDIT ORDER BY CHANGED_AT;
3️. Supervision Hierarchy with Recursion
 Corrected Recursive Query
sql
-- Corrected recursive query to find top supervisor
WITH SUPERS (EMP, SUP, HOPS, PATH) AS (
  -- Anchor: start with direct supervision, hop count = 1 (corrected from 0)
  SELECT EMPLOYEE, SUPERVISOR, 1, EMPLOYEE || '>' || SUPERVISOR
  FROM STAFF_SUPERVISOR
  UNION ALL
  -- Recursive: climb up the supervision chain (corrected join direction)
  SELECT S.EMPLOYEE, T.SUP, T.HOPS + 1, T.PATH || '>' || T.SUP
  FROM STAFF_SUPERVISOR S
  JOIN SUPERS T ON S.SUPERVISOR = T.EMP
  WHERE INSTR(T.PATH, T.SUP) = 0 -- corrected cycle guard
)
-- Final selection: top supervisor per employee
SELECT EMP, SUP AS TOP_SUPERVISOR, HOPS
FROM (
  SELECT EMP, SUP, HOPS,
         RANK() OVER (PARTITION BY EMP ORDER BY HOPS DESC) AS RANK -- replaced scalar MAX with RANK
  FROM SUPERS
)
WHERE RANK = 1;
4️. Semantic Reasoning with Triple Store
 Table and Inserts
sql
-- Create triple store table
CREATE TABLE TRIPLE (
  S VARCHAR2(100),
  P VARCHAR2(50),
  O VARCHAR2(100)
); 

-- Patient diagnoses
INSERT INTO TRIPLE VALUES ('patient1', 'hasDiagnosis', 'Influenza');
INSERT INTO TRIPLE VALUES ('patient2', 'hasDiagnosis', 'COVID19');
INSERT INTO TRIPLE VALUES ('patient3', 'hasDiagnosis', 'Malaria');
INSERT INTO TRIPLE VALUES ('patient4', 'hasDiagnosis', 'Diabetes');

-- Taxonomy edges
INSERT INTO TRIPLE VALUES ('Influenza', 'isA', 'ViralInfection');
INSERT INTO TRIPLE VALUES ('COVID19', 'isA', 'ViralInfection');
INSERT INTO TRIPLE VALUES ('Malaria', 'isA', 'ParasiticInfection');
INSERT INTO TRIPLE VALUES ('ViralInfection', 'isA', 'InfectiousDisease');
INSERT INTO TRIPLE VALUES ('ParasiticInfection', 'isA', 'InfectiousDisease');
INSERT INTO TRIPLE VALUES ('Diabetes', 'isA', 'ChronicDisease');

 Corrected Recursive Query
sql
-- Recursive query to infer infectious patients
WITH ISA(ANCESTOR, CHILD) AS (
  -- Anchor: direct isA relationships
  SELECT O, S FROM TRIPLE WHERE P = 'isA'
  UNION ALL
  -- Recursive: climb up the taxonomy
  SELECT I.ANCESTOR, T.S
  FROM TRIPLE T
  JOIN ISA I ON T.P = 'isA' AND T.O = I.CHILD
),
INFECTIOUS_PATIENTS AS (
  SELECT DISTINCT T.S
  FROM TRIPLE T
  JOIN ISA ON T.O = ISA.CHILD
  WHERE T.P = 'hasDiagnosis'
    AND ISA.ANCESTOR = 'InfectiousDisease'
)
SELECT S AS PATIENT_ID FROM INFECTIOUS_PATIENTS;
5️. Spatial Queries for Clinics
 Table and Metadata Setup
sql
-- Create clinic table with spatial geometry
CREATE TABLE CLINIC (
  ID NUMBER PRIMARY KEY,
  NAME VARCHAR2(100),
  GEOM SDO_GEOMETRY
);

  Register Spatial Metadata
sql
-- Register spatial metadata for the CLINIC.GEOM column
-- SDO_DIM_ELEMENT defines the valid coordinate ranges and resolution (tolerance)
-- SRID 4326 corresponds to WGS 84 (standard GPS coordinate system)
INSERT INTO USER_SDO_GEOM_METADATA
  (TABLE_NAME, COLUMN_NAME, DIMINFO, SRID)
VALUES (
  'CLINIC',
  'GEOM',
  SDO_DIM_ARRAY(
    SDO_DIM_ELEMENT('Longitude', 30.0, 31.0, 0.005), -- longitude range with 0.005 tolerance
    SDO_DIM_ELEMENT('Latitude', -2.5, -1.5, 0.005)   -- latitude range with 0.005 tolerance
  ),
  4326
);

Create Spatial Index
sql
-- Create a spatial index on the GEOM column to enable efficient spatial queries
CREATE INDEX CLINIC_SPX ON CLINIC(GEOM)
INDEXTYPE IS MDSYS.SPATIAL_INDEX;
 Insert Clinic Locations
sql
-- Insert clinics with their geographic coordinates (longitude, latitude)
-- Geometry type 2001 = 2D point; SRID 4326 = WGS 84

-- Kigali Central Clinic
INSERT INTO CLINIC VALUES (
  1, 'Kigali Central Clinic',
  SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(30.0610, -1.9575, NULL), NULL, NULL)
);

-- Nyamirambo Health Center
INSERT INTO CLINIC VALUES (
  2, 'Nyamirambo Health Center',
  SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(30.0595, -1.9560, NULL), NULL, NULL)
);

-- Gikondo Medical
INSERT INTO CLINIC VALUES (
  3, 'Gikondo Medical',
  SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(30.0700, -1.9500, NULL), NULL, NULL)
);

-- Commit the inserts
COMMIT;

 Find Clinics Within 1 KM of Ambulance
sql
-- Find clinics within 1 kilometer of the ambulance location (30.0600, -1.9570)
-- SDO_WITHIN_DISTANCE returns 'TRUE' if the distance condition is met
SELECT C.ID, C.NAME
FROM CLINIC C
WHERE SDO_WITHIN_DISTANCE(
  C.GEOM,
  SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(30.0600, -1.9570, NULL), NULL, NULL),
  'distance=1 unit=KM'
) = 'TRUE';

 Rank Clinics by Distance
sql
-- Calculate and return the distance (in KM) from the ambulance to each clinic
-- Sort by distance and return the top 3 closest clinics
SELECT C.ID, C.NAME,
       SDO_GEOM.SDO_DISTANCE(
         C.GEOM,
         SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(30.0600, -1.9570, NULL), NULL, NULL),
         0.005, -- tolerance for calculation
         'unit=KM' -- distance unit
       ) AS KM
FROM CLINIC C
ORDER BY KM
FETCH FIRST 3 ROWS ONLY;