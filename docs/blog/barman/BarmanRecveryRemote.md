---
title: "Restoring PostgreSQL Backups to a Different Server with Barman"
date: 2025-04-28
categories:
  - PostgreSQL
  - Backup
  - Recovery
tags:
  - barman
  - postgresql
  - database-recovery
  - disaster-recovery
  - cross-server-recovery
authors:
  - ruchany13
summary: "Learn how to restore PostgreSQL backups to a different server using Barman. A critical technique for disaster recovery and server migrations."
---

# Restoring PostgreSQL Backups to a Different Server with Barman
---

## Introduction

!!! info "Important Notice"
    This guide covers **restoring PostgreSQL backups to a different server** using **Barman**.  
    PostgreSQL and Barman setups are assumed to be already completed.

Restoring a PostgreSQL backup to a **different server** is crucial for:

- Disaster recovery in case of hardware failure
- Migration to a new environment
- Creating a test environment based on production data

With **Barman**, this operation becomes straightforward and reliable.

---

## Requirements

Before starting:

- A valid base backup must exist in Barman.
- Continuous WAL archiving must be configured.
- A secondary server with PostgreSQL installed and ready.

---

## Step 1: Prepare the Target Server

Ensure that PostgreSQL is installed on the new (target) server.

Install PostgreSQL:

```bash title="Install PostgreSQL (Ubuntu/Debian)"
sudo apt-get update
sudo apt-get install postgresql
```

or

```bash title="Install PostgreSQL (RHEL/Rocky)"
sudo yum install postgresql-server
```

Create an empty data directory:

```bash title="Create Data Directory"
sudo mkdir -p /database/PostgreSQL/13/main/
sudo chown postgres:postgres /database/PostgreSQL/13/main/
```

!!! warning "Data Directory Must Be Empty"
    Ensure the target data directory is **completely empty** before restoring the backup.

---

## Step 2: Enable SSH Access for Recovery

Ensure the Barman server can SSH into the **target server's** `postgres` user without a password.

On the Barman server:

```bash title="Setup SSH Access"
su - barman
ssh-copy-id postgres@<target-server-ip>
```

Test SSH connectivity:

```bash title="Test SSH"
ssh postgres@<target-server-ip>
```

You should be logged in without needing a password.

---

## Step 3: Initiate Recovery from Barman

Trigger the recovery process using the `barman recover` command.

```bash title="Recover Backup to a Different Server"
barman recover \
  --remote-ssh-command "ssh postgres@<target-server-ip>" \
  pg latest /database/PostgreSQL/13/main/
```

Replace `<target-server-ip>` with the IP address or hostname of your target server.

- `pg`: Name of your Barman server configuration.
- `/database/PostgreSQL/13/main/`: Data directory on the target server.

!!! note "Custom Paths"
    You can adjust the target data directory path according to your server's PostgreSQL setup.

---

## Step 4: Set the Restore Command on the Target Server

Edit or create the `postgresql.auto.conf` on the target server:

```bash title="Set Restore Command on Target"
restore_command = 'barman-wal-restore -P -U barman <barman-server-ip> pg %f %p'
```

Replace `<barman-server-ip>` with the IP address of your Barman server.

This allows PostgreSQL to pull any missing WAL segments from the Barman server during recovery.

---

## Step 5: Start PostgreSQL Service on the Target Server

Finally, start PostgreSQL on the new server:

```bash title="Start PostgreSQL on Target"
sudo systemctl start postgresql
```

PostgreSQL will apply all WALs and complete the recovery process.

!!! warning "Check Server Version Compatibility"
    Ensure the PostgreSQL version on the source and target servers is **identical** or **compatible** to avoid recovery issues.

---

## Conclusion

Restoring a PostgreSQL backup to a **different server** using Barman is a powerful technique for disaster recovery, server migrations, and cloning production environments for testing.

By following these steps, you can minimize downtime and ensure data consistency across servers!

> **Written by:** [Ruchan Yalçın](https://www.ruchan.dev) | [GitHub](https://github.com/ruchany13)

---
