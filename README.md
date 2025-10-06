# Mady_plsqlauca_26748

# Oracle PDB and OEM Assignment

## Overview

This project demonstrates how to:

1. Create a new Pluggable Database (PDB).
2. Create and delete a second PDB.
3. Configure Oracle Enterprise Manager (OEM) and show the dashboard.
4. Provide screenshots of the process.

---

## Task 1: Create a New Pluggable Database (PDB)

**Steps:**

1. Connect to the container database (CDB) as SYS with admin privileges.
2. Run the `CREATE PLUGGABLE DATABASE` command using the required naming convention:
   `(FirstTwoLettersOfFirstName_pdb_StudentID)`
3. Provide a simple password for the new admin user.
4. Open the PDB and save its state so it opens automatically on restart.

**SQL Code:**

```sql
-- Connect as sysdba
sqlplus sys@localhost:1521/cdb1 as sysdba;

-- Create PDB
CREATE PLUGGABLE DATABASE er_pdb_2024101
ADMIN USER eric_plsqlauca_2024101 IDENTIFIED BY YourPassword
FILE_NAME_CONVERT=('/u01/app/oracle/oradata/CDB1/pdbseed/',
                   '/u01/app/oracle/oradata/CDB1/er_pdb_2024101/');

-- Open PDB
ALTER PLUGGABLE DATABASE er_pdb_2024101 OPEN;

-- Save state for automatic opening
ALTER PLUGGABLE DATABASE er_pdb_2024101 SAVE STATE;
```

**Screenshot:**
*Add screenshot of successful PDB creation here.*

---

## Task 2: Create and Delete a PDB

**Steps:**

1. Create another PDB with the required format:
   `(FirstTwoLettersOfName_to_delete_pdb_StudentID)`.
2. Open the PDB to confirm it was created successfully.
3. Delete the PDB with the `DROP PLUGGABLE DATABASE` command.

**SQL Code:**

```sql
-- Create PDB to delete
CREATE PLUGGABLE DATABASE er_to_delete_pdb_2024101
ADMIN USER eric_plsqlauca_2024101 IDENTIFIED BY YourPassword
FILE_NAME_CONVERT=('/u01/app/oracle/oradata/CDB1/pdbseed/',
                   '/u01/app/oracle/oradata/CDB1/er_to_delete_pdb_2024101/');

-- Open the PDB
ALTER PLUGGABLE DATABASE er_to_delete_pdb_2024101 OPEN;

-- Delete the PDB (with its datafiles)
DROP PLUGGABLE DATABASE er_to_delete_pdb_2024101 INCLUDING DATAFILES;
```

**Screenshot:**
*Add screenshot of PDB creation.*
*Add screenshot of PDB deletion.*

---

## Task 3: Oracle Enterprise Manager (OEM)

**Steps:**

1. Open Oracle Enterprise Manager (usually at `https://localhost:5500/em`).
2. Log in with your username.
3. Confirm the dashboard displays your PDBs and username clearly.

**Screenshot:**
*Add screenshot of the OEM dashboard here.*

---

## Notes & Troubleshooting

* If you encounter path errors during PDB creation, confirm the correct `FILE_NAME_CONVERT` paths match your Oracle setup.
* Ensure your Oracle listener is running before connecting.
* If OEM doesn’t open, check your database services and restart the OEM console.

---

## Submission

* Push this README and your screenshots to your GitHub repository.
* Send the public repo link via email before the deadline.

---

## Grading Breakdown

* **Task 1:** 2 points
* **Task 2:** 2 points
* **Task 3:** 1 point
  *Total: 5 points*
