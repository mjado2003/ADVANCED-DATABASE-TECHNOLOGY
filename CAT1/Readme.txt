NGO Project Management Database
This document outlines the structure and logic of a relational database designed to 
manage donors, projects, staff, activities, expenses, and reports for an NGO. 
It includes table creation scripts, data insertion, queries, views, and triggers 
— all annotated for clarity and maintainability.

1️. Donor Table
sql
CREATE TABLE Donor (
    DonorID NUMBER PRIMARY KEY,              -- Unique identifier for each donor
    FullName VARCHAR2(100) NOT NULL,         -- Required full name
    Organization VARCHAR2(100),              -- Optional organization name
    Contact VARCHAR2(20) NOT NULL,           -- Required contact number
    Email VARCHAR2(100) NOT NULL UNIQUE      -- Required and unique email address
);
 Highlights:

Ensures donor identity is unique and complete.

Email uniqueness prevents duplicate entries.

2️.Project Table
sql
CREATE TABLE Project (
    ProjectID NUMBER PRIMARY KEY,                       -- Unique project ID
    Title VARCHAR2(150) NOT NULL,                       -- Required project title
    Description VARCHAR2(1000),                         -- Optional description
    Budget NUMBER(12,2) NOT NULL                        -- Required budget
        CONSTRAINT chk_budget CHECK (Budget >= 0),      -- Must be non-negative
    StartDate DATE NOT NULL,                            -- Required start date
    EndDate DATE,                                       -- Optional end date
    DonorID NUMBER NOT NULL,                            -- Required donor reference
    Status VARCHAR2(50) DEFAULT 'Ongoing',              -- Default status
    CONSTRAINT chk_status CHECK (Status IN              -- Allowed status values
        ('Planned', 'Ongoing', 'Completed', 'Cancelled')),
    CONSTRAINT fk_donor_project FOREIGN KEY (DonorID)
        REFERENCES Donor(DonorID)
);
 Highlights:

Enforces valid budget and status values.

Links each project to a donor.

3️. Staff Table
sql
CREATE TABLE Staff (
    StaffID NUMBER PRIMARY KEY,                         -- Unique staff ID
    FullName VARCHAR2(100) NOT NULL,                    -- Required name
    Role VARCHAR2(50) NOT NULL                          -- Required role
        CHECK (Role IN ('Manager', 'Coordinator', 'Volunteer', 'Analyst', 'Support')),
    Contact VARCHAR2(20) NOT NULL,                      -- Required contact
    ProjectID NUMBER NOT NULL,                          -- Required project reference
    CONSTRAINT fk_project_staff FOREIGN KEY (ProjectID)
        REFERENCES Project(ProjectID)
);
 Highlights:

Restricts roles to predefined values.

Ensures each staff member is linked to a project.

4️. Activity Table
sql
CREATE TABLE Activity (
    ActivityID NUMBER PRIMARY KEY,                         -- Unique activity ID
    ProjectID NUMBER NOT NULL,                             -- Required project reference
    Title VARCHAR2(150) NOT NULL,                          -- Required title
    ScheduleDate DATE NOT NULL,                            -- Required date
    Status VARCHAR2(50) NOT NULL                           -- Required status
        CHECK (Status IN ('Planned', 'Ongoing', 'Completed', 'Cancelled')),
    CONSTRAINT fk_project_activity FOREIGN KEY (ProjectID)
        REFERENCES Project(ProjectID)
);
Highlights:

Tracks project activities with status and schedule.

5️. Expense Table
sql
CREATE TABLE Expense (
    ExpenseID NUMBER PRIMARY KEY,                         -- Unique expense ID
    ProjectID NUMBER NOT NULL,                            -- Required project reference
    Description VARCHAR2(1000) NOT NULL,                  -- Required description
    Cost NUMBER(10,2) NOT NULL                            -- Required cost
        CHECK (Cost >= 0),                                -- Must be non-negative
    DateIncurred DATE NOT NULL                            -- Required date
        CHECK (DateIncurred >= TO_DATE('2000-01-01', 'YYYY-MM-DD')),
    CONSTRAINT fk_project_expense FOREIGN KEY (ProjectID)
        REFERENCES Project(ProjectID) ON DELETE CASCADE
);
Highlights:

Enforces valid cost and date.

Cascades deletion when a project is removed.

6️. Report Table
sql
CREATE TABLE Report (
    ReportID NUMBER PRIMARY KEY,                         -- Unique report ID
    ProjectID NUMBER NOT NULL,                           -- Required project reference
    SubmittedBy VARCHAR2(100) NOT NULL,                  -- Required submitter name
    DateSubmitted DATE NOT NULL                          -- Required date
        CHECK (DateSubmitted >= TO_DATE('2000-01-01', 'YYYY-MM-DD')),
    Summary VARCHAR2(4000) NOT NULL,                     -- Required summary
    CONSTRAINT fk_project_report FOREIGN KEY (ProjectID)
        REFERENCES Project(ProjectID) ON DELETE CASCADE
);
Highlights:

Tracks project reports with submission details.

Cascades deletion with project removal.

7️. Sample Data Insertion
 Donors
sql
INSERT INTO Donor VALUES (1, 'Alice Mugenzi', 'Mugenzi Foundation', '0788123456', 'alice@mugenzi.org');
INSERT INTO Donor VALUES (2, 'Jean Bosco', 'Bosco Group', '0788345678', 'jean.bosco@bosco.com');
INSERT INTO Donor VALUES (3, 'Clara Uwase', NULL, '0788567890', 'clara.uwase@gmail.com');

Projects
sql
INSERT INTO Project VALUES (101, 'Clean Water Initiative', 'Providing clean water to rural communities.', 50000.00, TO_DATE('2025-01-15','YYYY-MM-DD'), TO_DATE('2025-06-15','YYYY-MM-DD'), 1, 'Ongoing');
INSERT INTO Project VALUES (102, 'Youth Empowerment Program', 'Training youth in entrepreneurship and leadership.', 75000.00, TO_DATE('2025-02-01','YYYY-MM-DD'), NULL, 2, 'Planned');
INSERT INTO Project VALUES (103, 'Health Outreach Campaign', 'Mobile clinics and health education.', 60000.00, TO_DATE('2025-03-10','YYYY-MM-DD'), TO_DATE('2025-08-10','YYYY-MM-DD'), 1, 'Ongoing');
INSERT INTO Project VALUES (104, 'Digital Literacy for Women', 'Teaching basic computer skills to women.', 40000.00, TO_DATE('2025-04-05','YYYY-MM-DD'), NULL, 3, 'Planned');
INSERT INTO Project VALUES (105, 'Environmental Awareness Drive', 'Workshops and campaigns on climate change.', 30000.00, TO_DATE('2025-05-20','YYYY-MM-DD'), TO_DATE('2025-09-20','YYYY-MM-DD'), 2, 'Ongoing');
8️. Queries and Logic
 Total Expenses per Project
sql
SELECT 
    p.ProjectID,
    p.Title,
    SUM(e.Cost) AS TotalExpense
FROM 
    Project p
LEFT JOIN 
    Expense e ON p.ProjectID = e.ProjectID
GROUP BY 
    p.ProjectID, p.Title;

	Update Project Status Based on Final Report
sql
UPDATE Project
SET Status = 'Completed'
WHERE ProjectID IN (
    SELECT ProjectID
    FROM Report
    WHERE Summary LIKE '%final report%'
);
 Donors Funding Multiple Projects
sql
SELECT DonorID, COUNT(ProjectID) AS ProjectCount
FROM Project
GROUP BY DonorID
HAVING COUNT(ProjectID) > 1;
9️.View: Project Spending Summary
sql
CREATE VIEW ProjectSpending AS
SELECT 
    p.ProjectID,
    p.Title,
    COALESCE(SUM(e.Cost), 0) AS TotalSpending
FROM 
    Project p
LEFT JOIN 
    Expense e ON p.ProjectID = e.ProjectID
GROUP BY 
    p.ProjectID, p.Title;
sql
-- View usage
SELECT * FROM ProjectSpending;
10. Trigger: Prevent Over-Budget Expenses
sql
CREATE OR REPLACE TRIGGER prevent_over_budget_expense
BEFORE INSERT ON Expense
FOR EACH ROW
DECLARE
    total_spent NUMBER;
    project_budget NUMBER;
BEGIN
    SELECT COALESCE(SUM(Cost), 0)
    INTO total_spent
    FROM Expense
    WHERE ProjectID = :NEW.ProjectID;

    SELECT Budget
    INTO project_budget
    FROM Project
    WHERE ProjectID = :NEW.ProjectID;

    IF total_spent + :NEW.Cost > project_budget THEN
        RAISE_APPLICATION_ERROR(-20001, 'Expense exceeds project budget.');
    END IF;
END;

Purpose:

Prevents inserting expenses that would exceed the allocated project budget.