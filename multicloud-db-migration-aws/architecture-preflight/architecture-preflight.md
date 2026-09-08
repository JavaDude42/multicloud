# Lab 1: Confirm the Migration Architecture and Preflight Checks

## Introduction

Confirm the migration boundary. The source and ZDM service run on the same Oracle Database 19c EC2 host. Oracle ZDM 26.1 writes the Data Pump dump set to Amazon EFS. Private Autonomous Database Serverless reads the same file system through its FSS directory. The target must sit in an ODB network with private routing to the AWS VPC.

The lab uses one table: `FINANCE.RISK_AUDIT_ARCHIVE`. Keep the requested scope below 20 GiB. The source allocation is 16.551 GiB, including a 16.501 GiB LOB segment. The later independent count is 400,000 rows. The initial `DBA_TABLES.NUM_ROWS` statistic showed 200,000 rows.

Estimated Time: 15 minutes

### Objectives

In this lab, you will:

- Confirm the source, target, EFS, ZDM, and directory-object values.
- Confirm the offline downtime model and bounded migration scope.
- Verify DNS, routing, and TCP reachability before attaching shared storage.

## Task 1: Confirm the Migration Boundary

1. Confirm that the lab uses an offline logical migration.

    Stop application writes or place the workload in read-only mode before the final export. Redirect traffic only after target validation succeeds. This workflow does not use GoldenGate, change data capture, or manually executed `expdp` and `impdp` commands.

2. Confirm the migration scope.

    - Source database: Oracle Database 19c, SID `SOURCE19C`, PDB/service `srcpdb1`.
    - Target database: Autonomous Database Serverless `zdm-lab-adbs-private` on Oracle Database@AWS.
    - Target database and AWS ID: `zdm-lab-adbs-private`, `adb_q8qutj06v0`.
    - Target database version: Oracle Database 19c.
    - Target container name: `G7C2CC53B996BEB_ZDMLABPRIV`.
    - Scope: `FINANCE.RISK_AUDIT_ARCHIVE` only.
    - Maximum requested scope: 20 GiB.
    - Tablespace remap: `FINANCE_TS` to `DATA`.
    - Target owner: `FINANCE`, with a bounded 25 GiB DATA quota and only `CREATE SESSION` plus `CREATE TABLE`.

3. Note the source-size and row-count distinction for later validation.

    The initial optimizer statistic showed 200,000 rows. The later actual count was 400,000 rows. The source allocation was 16.551 GiB, made up of a 0.050 GiB table segment and a 16.501 GiB LOB segment. Treat the independent `COUNT(*)` and Data Pump evidence as the row-count result.

## Task 2: Review the Validated Environment

1. Confirm the source and ZDM host values.

    - EC2 instance: `i-0de3349a478bae10`.
    - Private IP: `10.0.0.170`.
    - Instance type: `r6i.xlarge`.
    - Operating system: Red Hat Enterprise Linux 8.10.
    - Oracle Home: `/data/oracle/app/oracle/product/19.0.0/dbhome_1`.
    - Oracle data mount: `/data/oracle`.
    - ZDM Home: `/data/oracle/zdm26/private/zdmhome`.
    - ZDM Base: `/data/oracle/zdm26/private/zdmbase`.

2. Confirm the target and shared-storage values.

    - Target service alias: `zdmlabpriv_high`.
    - Target private endpoint: `z0edijb4.aws-us-east-1.adb.us-ashburn-1.oraclecloud.com`.
    - Target private IP: `172.128.1.205`.
    - Target service: `g7c2cc53b996beb_zdmlabpriv_high.adb.oraclecloud.com`.
    - EFS file system: `fs-0d73948458dc51c34`.
    - EFS mount target: `10.0.0.169`.
    - EC2 mount: `/data/oracle/efs`.
    - Source directory object: `DATA_PUMP_DIR_NFS`.
    - Target directory object: `FSS_DIR`.
    - Target file system name: `ZDM_EFS`, location `efs.zdm.internal:/`.
    - ODB client network: `172.128.1.0/24`.

## Task 3: Run the Network Preflight Checks

1. On the source and ZDM EC2 host, verify DNS resolution and TCP reachability.

    ```bash
    getent hosts efs.zdm.internal
    nc -vz 10.0.0.169 2049
    getent hosts z0edijb4.aws-us-east-1.adb.us-ashburn-1.oraclecloud.com
    nc -vz z0edijb4.aws-us-east-1.adb.us-ashburn-1.oraclecloud.com 1522
    ```

2. Confirm the corresponding network controls.

    - The target uses a private endpoint in an ODB network connected to the EC2 VPC.
    - The VPC route table contains the active route to the ODB client CIDR.
    - The ODB network routes back to the EC2 subnet.
    - The EFS security group allows inbound TCP 2049 from the EC2 source security group and the ODB client CIDR.
    - The target resolver resolves `efs.zdm.internal` to the EFS mount-target address before attachment.
    - The EC2 host reaches the source listener on TCP 1521 and the target private endpoint on TCPS 1522.

3. If a check fails, stop before mounting or attaching EFS. The original environment first lacked a private ADB-S endpoint in a connected ODB network. That gap prevented shared NFS staging. The validated environment corrected the network placement before the migration run.

## Acknowledgements

* **Author** - Workshop team
* **Last Updated By/Date** - Workshop team / September 8, 2026
