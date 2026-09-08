# Lab 2: Prepare Shared Amazon EFS NFS Staging

## Introduction

Prepare the shared staging path used by ZDM. The source EC2 host mounts Amazon EFS through NFSv4. It exposes that mount as `DATA_PUMP_DIR_NFS`. The private Autonomous Database Serverless target attaches the same file system as `ZDM_EFS`. It exposes the file system through `FSS_DIR`. A bidirectional file test confirms the shared location before ZDM starts.

Estimated Time: 25 minutes

### Objectives

In this lab, you will:

- Create or reuse the EFS file system and mount target.
- Mount and test EFS on the source/ZDM EC2 host.
- Attach and test the same file system on private Autonomous Database Serverless.

## Task 1: Create or Reuse the EFS File System

1. In AWS CloudShell or the supplied provisioning environment, set the validated region and network values.

    ```bash
    export AWS_REGION=us-east-1
    export VPC_ID=vpc-0356ab7623c1fd719
    export SUBNET_ID=subnet-035590ab09b91d1c3
    ```

2. If the instructor has not already created EFS, create the encrypted, elastic-throughput file system and its mount target.

    ```bash
    EFS_ID=$(aws efs create-file-system \
      --region "$AWS_REGION" --creation-token zdm-lab-shared-nfs-us-east-1 \
      --encrypted --performance-mode generalPurpose --throughput-mode elastic \
      --tags Key=Name,Value=zdm-lab-shared-nfs Key=Project,Value=ZDM-HOL \
      --query FileSystemId --output text)
    aws efs create-mount-target --region "$AWS_REGION" \
      --file-system-id "$EFS_ID" --subnet-id "$SUBNET_ID" \
      --security-groups "$EFS_SG_ID"
    ```

    Amazon EFS expands as data is written, so this service does not require a fixed 500 GiB allocation. Use the complete idempotent `commands.sh` sequence from the lab package when it is available.

## Task 2: Mount and Test EFS on the Source EC2 Host

1. Set the validated EFS values and install the NFS client utilities.

    ```bash
    export EFS_ID=fs-0d73948458dc51c34
    export EFS_MOUNT=/data/oracle/efs
    export EFS_HOST=${EFS_ID}.efs.us-east-1.amazonaws.com
    sudo dnf install -y nfs-utils
    sudo install -d -o oracle -g oracle -m 0777 "$EFS_MOUNT"
    ```

2. Mount the file system as NFSv4 and make the mount persistent.

    ```bash
    sudo mount -t nfs4 -o nfsvers=4.1,_netdev "${EFS_HOST}:/" "$EFS_MOUNT"
    printf '%s\n' "${EFS_HOST}:/ $EFS_MOUNT nfs4 defaults,_netdev,nofail,nfsvers=4.1 0 0" \
      | sudo tee -a /etc/fstab
    printf 'source test %s\n' "$(date -u +%FT%TZ)" > "$EFS_MOUNT/source_efs_test.txt"
    findmnt -T "$EFS_MOUNT"
    ls -l "$EFS_MOUNT/source_efs_test.txt"
    ```

3. Create and grant the source directory object.

    Connect as local SYSDBA, switch to `SRCPDB1`, and run:

    ```sql
    ALTER SESSION SET CONTAINER=SRCPDB1;
    CREATE OR REPLACE DIRECTORY DATA_PUMP_DIR_NFS AS '/data/oracle/efs';
    GRANT READ, WRITE ON DIRECTORY DATA_PUMP_DIR_NFS TO SYSTEM;
    SELECT directory_name, directory_path
    FROM dba_directories
    WHERE directory_name = 'DATA_PUMP_DIR_NFS';
    ```

    Confirm that `DATA_PUMP_DIR_NFS` resolves to `/data/oracle/efs`.

## Task 3: Attach and Test EFS on Private Autonomous Database Serverless

1. Set the target wallet path and connect as `ADMIN` through the validated service alias.

    ```bash
    export TNS_ADMIN=/data/oracle/wallets/zdmlabpriv
    sqlplus admin@zdmlabpriv_high
    ```

2. Grant the target host access needed for DNS resolution and connection.

    ```sql
    BEGIN
      DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
        host => 'efs.zdm.internal',
        ace  => XS$ACE_TYPE(
          privilege_list => XS$NAME_LIST('connect','resolve'),
          principal_name => 'ADMIN',
          principal_type => XS_ACL.PTYPE_DB,
          granted        => TRUE));
    END;
    /
    COMMIT;
    SELECT UTL_INADDR.GET_HOST_ADDRESS('efs.zdm.internal') AS efs_ip FROM dual;
    ```

    Confirm that the resolved address is the EFS mount-target address `10.0.0.169`.

3. Create the target directory and attach the file system with NFSv4.

    ```sql
    CREATE DIRECTORY FSS_DIR AS 'fss';
    BEGIN
      DBMS_CLOUD_ADMIN.ATTACH_FILE_SYSTEM(
        file_system_name     => 'ZDM_EFS',
        file_system_location => 'efs.zdm.internal:/',
        directory_name       => 'FSS_DIR',
        description          => 'ZDM shared Amazon EFS staging over NFSv4',
        params               => JSON_OBJECT('nfs_version' VALUE 4));
    END;
    /
    ```

4. Confirm the file-system metadata and list the shared files.

    ```sql
    SELECT file_system_name, file_system_location, directory_name
    FROM dba_cloud_file_systems WHERE file_system_name = 'ZDM_EFS';
    SELECT object_name, bytes FROM DBMS_CLOUD.LIST_FILES('FSS_DIR');
    ```

    The target must list the EFS test file created on EC2. This is the staging checkpoint for the ZDM evaluation.

## Acknowledgements

* **Author** - Workshop team
* **Last Updated By/Date** - Workshop team / September 8, 2026
