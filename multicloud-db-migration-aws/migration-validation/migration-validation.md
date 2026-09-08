# Lab 4: Execute, Monitor, and Validate the Migration

## Introduction

Run the evaluated response file during the application maintenance window. Monitor ZDM phases and EFS dump files. Validate the target independently. The supplied successful run completed in about nine minutes. Use your job IDs and log paths as participant evidence. Job 4 and its artifacts are reference evidence from the validated source run.

Estimated Time: 25 minutes

### Objectives

In this lab, you will:

- Start and monitor the ZDM offline logical migration.
- Interpret a resumable target-validation failure without restarting the migration.
- Validate row counts, object status, SecureFile LOB behavior, tablespace remapping, quota usage, and Data Pump logs.

## Task 1: Start the Migration During the Maintenance Window

1. Confirm that application writes have stopped or that the workload is in read-only mode. Do not redirect traffic to the target yet.

2. Run the same response file without `-eval`.

    ```bash
    $ZDM_HOME/bin/zdmcli migrate database -rsp "$RSP"
    ```

3. If the first target-validation call stops with the transient `ORA-12530: TNS:listener: rate limit reached`, confirm that no export has started and wait for the connection window to clear. Resume the existing job.

    ```bash
    $ZDM_HOME/bin/zdmcli resume job -jobid 4
    ```

    The validated job 4 resumed successfully. Do not create a second migration job for a resumable phase after correcting the root cause.

## Task 2: Monitor ZDM and Shared EFS

1. Query the active job.

    ```bash
    $ZDM_HOME/bin/zdmcli query job -jobid 4
    ```

2. Monitor files written directly to EFS.

    ```bash
    find /data/oracle/efs -maxdepth 1 -type f -name 'ZDM_4_*' -ls
    ```

3. Confirm the expected phase progression.

    - `ZDM_VALIDATE_TGT` - `COMPLETED`.
    - `ZDM_VALIDATE_SRC` - `COMPLETED`.
    - `ZDM_SETUP_SRC` - `COMPLETED`.
    - `ZDM_PRE_MIGRATION_ADVISOR` - `COMPLETED`.
    - `ZDM_VALIDATE_DATAPUMP_SETTINGS_SRC` - `COMPLETED`.
    - `ZDM_VALIDATE_DATAPUMP_SETTINGS_TGT` - `COMPLETED`.
    - `ZDM_PREPARE_DATAPUMP_SRC` - `COMPLETED`.
    - `ZDM_DATAPUMP_ESTIMATE_SRC` - `COMPLETED`.
    - `ZDM_PREPARE_DATAPUMP_TGT` - `COMPLETED`.
    - `ZDM_DATAPUMP_EXPORT_SRC` - `COMPLETED`.
    - `ZDM_TRANSFER_DUMPS_SRC` - `COMPLETED`.
    - `ZDM_DATAPUMP_IMPORT_TGT` - `COMPLETED`.
    - `ZDM_POST_DATAPUMP_SRC` - `COMPLETED`.
    - `ZDM_POST_DATAPUMP_TGT` - `COMPLETED`.
    - `ZDM_REFRESH_MVIEW_TGT` - `COMPLETED`.
    - `ZDM_POST_ACTIONS` - `COMPLETED`.
    - `ZDM_CLEANUP_SRC` - `COMPLETED`.

## Task 3: Review the Migration and Data Pump Results

1. Query the completed job and compare the results with the validated reference.

    The reference job 4 status is `SUCCEEDED`. ZDM reported 9 minutes 3 seconds elapsed time, and the result log reported 9 minutes 1 second. Use the job result and metrics files created by your run.

2. Review the reference Data Pump outcomes.

    - Estimate: `ZDM_4_DP_ESTIMATE_62989`, 16.55 GB using the BLOCKS method.
    - Export: `ZDM_4_DP_EXPORT_535678`, 400,000 rows, 79.62 MB compressed payload, 5 minutes 22 seconds.
    - Import: `ZDM_4_DP_IMPORT_86562`, 400,000 rows, 79.62 MB payload, 1 minute 27 seconds.
    - Export method: `direct_path`.
    - Import method: `external_table`.
    - Export and import status: Successfully completed.

3. Confirm the retained EFS artifacts from the reference run.

    - `/data/oracle/efs/ZDM_4_DP_EXPORT_535678_dmp_1_01.dmp` - 12 KiB.
    - `/data/oracle/efs/ZDM_4_DP_EXPORT_535678_dmp_1_02.dmp` - 83,529,728 bytes.
    - `/data/oracle/efs/ZDM_4_DP_EXPORT_535678.log`.
    - `/data/oracle/efs/ZDM_4_DP_IMPORT_86562.log`.

## Task 4: Run Independent Validation

1. Run the source validation as local SYSDBA after switching to `SRCPDB1`.

    ```bash
    sqlplus -s / as sysdba @validate_source.sql
    ```

2. Run the target validation as `ADMIN@zdmlabpriv_high`.

    ```bash
    export TNS_ADMIN=/data/oracle/wallets/zdmlabpriv
    sqlplus -s admin@zdmlabpriv_high @validate_target.sql
    ```

3. Confirm the row counts and object properties.

    - Source actual row count: 400,000.
    - Target actual row count: 400,000.
    - Target imported statistics `NUM_ROWS`: 400,000.
    - Source and target table objects: `VALID`.
    - Source LOB: `AUDIT_PAYLOAD` in `FINANCE_RISK_LOB`, SecureFile `YES`, `FINANCE_TS`.
    - Target LOB: `AUDIT_PAYLOAD`, SecureFile `YES`, remapped to `DATA`.
    - Source complete allocated footprint: 16.551 GiB.
    - Target complete allocated footprint: 12.557 GiB.
    - Target DATA quota: 25 GiB; used: 12.557 GiB.

    The smaller target allocation reflects segment compaction during export and import. The stale preflight `DBA_TABLES.NUM_ROWS=200000` value does not override the independent `COUNT(*)` and Data Pump evidence.

4. Query ZDM and check the successful Data Pump logs for Oracle errors.

    ```bash
    $ZDM_HOME/bin/zdmcli query job -jobid 4
    grep -n 'ORA-' /data/oracle/efs/ZDM_4_DP_EXPORT_535678.log \
      /data/oracle/efs/ZDM_4_DP_IMPORT_86562.log
    ```

    The reference run contains no `ORA-` entries in either successful Data Pump log.

## Task 5: Preserve the Evidence and Lessons Learned

1. Record the ZDM and diagnostic locations.

    - ZDM service log: `/data/oracle/zdm26/private/zdmbase/crsdata/ip-10-0-0-170/rhp/zdmserver.log.0`
    - Evaluation result: `/data/oracle/zdm26/private/zdmbase/chkbase/scheduled/job-3-2026-09-04-09:01:50.log`
    - Migration result: `/data/oracle/zdm26/private/zdmbase/chkbase/scheduled/job-4-2026-09-04-09:03:50.log`
    - Migration metrics: `/data/oracle/zdm26/private/zdmbase/chkbase/scheduled/job-4-2026-09-04-09:03:50.json`
    - CPAT report: `/data/oracle/zdm26/private/zdmbase/crsdata/ip-10-0-0-170/rhp/temp/zdm/zdm_SOURCE19C_4/out/premigration_advisor_report.txt`
    - Export log: `/data/oracle/efs/ZDM_4_DP_EXPORT_535678.log`
    - Import log: `/data/oracle/efs/ZDM_4_DP_IMPORT_86562.log`

2. Keep the migration package secret-free. The source runbook intentionally omits passwords and wallet contents. Retain the response file, validation SQL, repeatable commands, and sanitized logs with the lab package, but do not add credentials.

3. Capture the two migration lessons from the validated run.

    - Select the staging method only after validating network placement. The original public or disconnected ADB-S placement could not reach Amazon EFS, so Amazon S3 was the initial staging choice. The corrected private ODB network enabled shared NFS staging.
    - Test the ZDM-managed target import before scaling the migration. The earlier S3 path reached the target but the ZDM-managed import stopped in `ZDM_DATAPUMP_IMPORT_TGT` with `PRGZ-1477`, `PRGD-1019`, `PRGD-1016`, `ORA-20000`, and `ORA-39001` during `DBMS_DATAPUMP.OPEN`. Shared EFS staging with the private ADB-S endpoint allowed the same ZDM workflow to complete end to end.

## Acknowledgements

* **Author** - Workshop team
* **Last Updated By/Date** - Workshop team / September 8, 2026
