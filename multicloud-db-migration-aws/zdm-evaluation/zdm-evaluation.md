# Lab 3: Configure and Evaluate the ZDM Migration

## Introduction

Configure the secret-free response file for the offline logical migration. The validated ZDM version is `26.1.0`. The response file selects table mode and includes only `FINANCE.RISK_AUDIT_ARCHIVE`. It writes the export to `DATA_PUMP_DIR_NFS` and imports from `FSS_DIR`. It retains the dump set and remaps `FINANCE_TS` to `DATA`. Run evaluation first. ZDM and CPAT can then check the source, target, directory objects, storage, and privileges.

Estimated Time: 20 minutes

### Objectives

In this lab, you will:

- Create protected source-admin and target-admin auto-login wallet locations without placing passwords in the response file.
- Review the bounded target owner prerequisite.
- Create the successful response-file settings and run ZDM evaluation.

## Task 1: Prepare Wallet-Based Authentication

1. Create the wallet directories with the supplied permissions.

    ```bash
    mkdir -p /data/oracle/zdm26/private/wallets/{source_admin,target_admin}
    chmod 700 /data/oracle/zdm26/private/wallets/{source_admin,target_admin}
    ```

2. Create the source-admin and target-admin auto-login wallets.

    ```bash
    orapki wallet create -wallet /data/oracle/zdm26/private/wallets/source_admin -auto_login_only
    mkstore -wrl /data/oracle/zdm26/private/wallets/source_admin -createCredential store SYSTEM
    orapki wallet create -wallet /data/oracle/zdm26/private/wallets/target_admin -auto_login_only
    mkstore -wrl /data/oracle/zdm26/private/wallets/target_admin -createCredential store ADMIN
    ```

    Enter credentials only at the protected runtime prompts. Wallet files must remain mode 600, wallet directories must remain mode 700, and passwords must not appear in the response file, command transcript, or captured logs.

## Task 2: Confirm the Target Owner Prerequisite

1. Confirm that the target has an empty `FINANCE` owner with a bounded 25 GiB DATA quota and only `CREATE SESSION` plus `CREATE TABLE`. Before the run, the target had no `FINANCE` user and no `FINANCE.RISK_AUDIT_ARCHIVE` table.

    Do not use broad `RESOURCE` privileges or an unlimited quota. Do not create the target table or data manually. ZDM must create and populate the table through the migration.

2. If evaluation stops with `PRGZ-1391`, create the empty target owner using the supplied target-preparation SQL, then rerun evaluation. The source run recorded this exact prerequisite failure during evaluation job 2.

## Task 3: Create the Successful Response File

1. Create `/data/oracle/zdm26/private/response/zdm_finance_risk_nfs.rsp` with the following secret-free settings.

    ```text
    MIGRATION_METHOD=OFFLINE_LOGICAL
    DATA_TRANSFER_MEDIUM=NFS
    RUNCPATREMOTELY=TRUE
    SOURCEDATABASE_ENVIRONMENT_NAME=ORACLE
    SOURCEDATABASE_ENVIRONMENT_DBTYPE=ORACLE
    SOURCEDATABASE_ADMINUSERNAME=SYSTEM
    SOURCEDATABASE_CONNECTIONDETAILS_HOST=127.0.0.1
    SOURCEDATABASE_CONNECTIONDETAILS_PORT=1521
    SOURCEDATABASE_CONNECTIONDETAILS_SERVICENAME=srcpdb1
    TARGETDATABASE_DBTYPE=ADBS
    TARGETDATABASE_ADMINUSERNAME=ADMIN
    TARGETDATABASE_CONNECTIONDETAILS_HOST=z0edijb4.aws-us-east-1.adb.us-ashburn-1.oraclecloud.com
    TARGETDATABASE_CONNECTIONDETAILS_PORT=1522
    TARGETDATABASE_CONNECTIONDETAILS_SERVICENAME=zdmlabpriv_high
    TARGETDATABASE_CONNECTIONDETAILS_TLSDETAILS_CREDENTIALSLOCATION=/data/oracle/wallets/zdmlabpriv
    DATAPUMPSETTINGS_JOBMODE=TABLE
    INCLUDEOBJECTS-1=owner:FINANCE,objectName:RISK_AUDIT_ARCHIVE,objectType:TABLE
    DATAPUMPSETTINGS_EXPORTDIRECTORYOBJECT_NAME=DATA_PUMP_DIR_NFS
    DATAPUMPSETTINGS_EXPORTDIRECTORYOBJECT_PATH=/data/oracle/efs
    DATAPUMPSETTINGS_IMPORTDIRECTORYOBJECT_NAME=FSS_DIR
    DATAPUMPSETTINGS_IMPORTDIRECTORYOBJECT_PATH=
    DUMPTRANSFERDETAILS_SHAREDSTORAGE_NAME=ZDM_EFS
    DUMPTRANSFERDETAILS_SHAREDSTORAGE_HOST=efs.zdm.internal
    DUMPTRANSFERDETAILS_SHAREDSTORAGE_PATH=/
    DUMPTRANSFERDETAILS_SHAREDSTORAGE_DETACHFSSPOST=FALSE
    DATAPUMPSETTINGS_METADATAREMAPS-1=type:REMAP_TABLESPACE,oldValue:FINANCE_TS,newValue:DATA
    DATAPUMPSETTINGS_DATAPUMPPARAMETERS_EXPORTPARALLELISMDEGREE=2
    DATAPUMPSETTINGS_DATAPUMPPARAMETERS_IMPORTPARALLELISMDEGREE=2
    DATAPUMPSETTINGS_DATAPUMPPARAMETERS_ENCRYPTION=NONE
    DATAPUMPSETTINGS_RETAINDUMPS=TRUE
    DATAPUMPSETTINGS_ENABLEDIAGCOLLECTION=TRUE
    WALLET_SOURCEADMIN=/data/oracle/zdm26/private/wallets/source_admin
    WALLET_TARGETADMIN=/data/oracle/zdm26/private/wallets/target_admin
    ```

    The wallet service alias `zdmlabpriv_high` is intentional. ZDM 26.1 rejected the fully qualified ADB service string in `TARGETDATABASE_CONNECTIONDETAILS_SERVICENAME` with `PRGZ-1131` during evaluation job 1. The alias matches the ADB wallet and the successful evaluation.

## Task 4: Run the ZDM Evaluation

1. Set the ZDM environment and run the evaluation.

    ```bash
    export ZDM_HOME=/data/oracle/zdm26/private/zdmhome
    export RSP=/data/oracle/zdm26/private/response/zdm_finance_risk_nfs.rsp
    $ZDM_HOME/bin/zdmservice status
    $ZDM_HOME/bin/zdmcli migrate database -rsp "$RSP" -eval
    $ZDM_HOME/bin/zdmcli query job -jobid 3
    ```

2. Review the evaluation result.

    Job 3 completed every evaluation phase, verified both NFS directory objects, recognized the already attached target file system, and estimated 16.55 GB using the BLOCKS method. CPAT performed 27 checks with zero Failed and zero Action Required results. Its one Review Required item concerned owner create privileges; the target `FINANCE` owner has the required `CREATE TABLE` privilege.

3. Record the evaluation result file.

    ```text
    /data/oracle/zdm26/private/zdmbase/chkbase/scheduled/job-3-2026-09-04-09:01:50.log
    ```

    Keep this evaluation as a pre-migration checkpoint. The participant migration must use a new job identifier.

## Acknowledgements

* **Author** - Workshop team
* **Last Updated By/Date** - Workshop team / September 8, 2026
