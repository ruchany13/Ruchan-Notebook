---
title: "Performing Point-in-Time Recovery (PITR) with Barman"
date: 2025-04-28
categories:
  - PostgreSQL
  - Backup
  - Recovery
tags:
  - barman
  - postgresql
  - PITR
  - disaster-recovery
  - database-restore
authors:
  - ruchany13
summary: "Learn how to perform Point-in-Time Recovery (PITR) with Barman for PostgreSQL databases. A critical technique for precise database recovery."
---

# Performing Point-in-Time Recovery (PITR) with Barman

---

## Introduction

!!! info "Important Notice"
    This article focuses on **performing Point-in-Time Recovery (PITR)** operations using **Barman** for PostgreSQL databases.  
    PostgreSQL and Barman setup procedures are assumed to be already completed.

**Point-in-Time Recovery (PITR)** allows you to restore your PostgreSQL database to a specific moment in the past — for example, just before a mistaken operation or accidental deletion occurred.  
This technique is critical for minimizing data loss and restoring system integrity.

---

## What is Point-in-Time Recovery (PITR)?

Point-in-Time Recovery enables database administrators to **restore the database to any desired timestamp** using the available base backup and Write Ahead Log (WAL) archives.

It is extremely useful for:

- Recovery from human errors
- Undoing accidental schema or data changes
- Restoring pre-attack system state in case of security incidents

---

## Requirements

Before starting:

- You must have a **valid base backup** in Barman.
- Continuous WAL archiving must be **enabled**.
- The approximate **timestamp** you want to recover to must be known.

---

## Step 1: Stop PostgreSQL Service

Before performing recovery, stop the PostgreSQL service.

```bash title="Stop PostgreSQL"
sudo systemctl stop postgresql
```

---

## Step 2: Identify the Target Time

Decide the **exact timestamp** (UTC format recommended) you wish to restore to.  
Example: `2025-04-25 12:30:00`

---

## Step 3: Initiate Recovery with Barman

Use the `barman recover` command with the `--target-time` option:

```bash title="Recover to a Specific Time"
barman recover \
  --remote-ssh-command 'ssh postgres@localhost' \
  --target-time '2025-04-25 12:30:00' \
  pg latest /database/PostgreSQL/13/main/
```

- `--target-time`: The point in time to stop WAL recovery.
- `pg`: Name of your Barman server configuration.
- `/database/PostgreSQL/13/main/`: PostgreSQL data directory.

!!! warning "Critical Note"
    Ensure the target-time is correctly specified. Recovery will apply WAL files up to, but not beyond, this point.

---

## Step 4: Set the Restore Command

After restoring the backup, configure PostgreSQL to fetch missing WAL files if necessary.

Edit or create `postgresql.auto.conf`:

```bash title="Set Restore Command"
restore_command = 'barman-wal-restore -P -U barman localhost pg %f %p'
```

This command ensures that any required WAL files missing from the local archive will be pulled from Barman.

---

## Step 5: Start PostgreSQL in Recovery Mode

Start the PostgreSQL service:

```bash title="Start PostgreSQL"
sudo systemctl start postgresql
```

PostgreSQL will automatically recover to the specified point and switch to normal operation after reaching the target time.

!!! note "Recovery Behavior"
    After successful PITR, PostgreSQL will **remove the recovery configuration** and start serving normally.

---

## Conclusion

Performing **Point-in-Time Recovery (PITR)** using Barman is an essential capability for PostgreSQL administrators aiming to protect critical data and minimize downtime.

Thanks to Barman's automated backup and WAL management, recovering to a specific time becomes a reliable and repeatable process!

Stay tuned for the next guide on **restoring backups to a different server with Barman**!

> **Written by:** [Ruchan Yalçın](https://www.ruchan.dev) | [GitHub](https://github.com/ruchany13)

---
