---
title: "Setting Up Barman for Local PostgreSQL Backups"
date: 2025-04-28
categories:
  - PostgreSQL
  - Backup
  - Demo
tags:
  - barman
  - postgresql
  - backup
  - devops
  - demo
summary: "A step-by-step guide to install and configure Barman for PostgreSQL backups on the same server. Useful for demo or testing environments."
---


# Setting Up Barman for Local PostgreSQL Backups


---

## Introduction

!!! info "About This Guide"
    This guide explains how to install and configure Barman for backups on the **same server** as PostgreSQL. It is intended for demo or testing environments, not for production use.

In this article, I will explain how to install and configure **Barman** to perform backups of a PostgreSQL database on the **same server**. This method is not a best practice for production environments, but it can be very useful for demo environments, testing purposes, or emergency scenarios where simplicity is more important than scalability.

---

## What is Barman?

**Barman** (Backup and Recovery Manager) is an open-source tool designed to manage backup and recovery operations for PostgreSQL databases.

With Barman, you can:

- Take backups on the same server or a different backup server.

- Manage backups from multiple PostgreSQL clusters.

- Support various PostgreSQL versions.

- Choose between **streaming** and **rsync/ssh** backup methods.

!!! note "Why WAL Streaming Matters"
    WAL streaming ensures near-zero data loss (RPO=0), making it ideal for critical systems where minimal data loss is acceptable.

---
<!-- more -->
## Installing Barman

### System Requirements

- PostgreSQL Version >= 10

- Python >= 3.6

### Server Setup Used

- OS Version: Ubuntu 22.04.3

- PostgreSQL Version: 13

- PostgreSQL Data Directory: `/database/PostgreSQL/13/main/`

- Barman Backup Directory: `/pg_backup/`

> In this demo, **Barman is installed on the same server as PostgreSQL**.

### Installation on Ubuntu/Debian

```bash title="Ubuntu/Debian Install"
sudo apt-get -y install barman
sudo apt-get -y install barman-cli
```

### Installation on RHEL/Rocky Linux

```bash title="RHEL/Rocky Install"
sudo yum install -y barman
sudo yum install -y barman-cli
```

This installation will create a dedicated **barman** user that must be used to run backup and recovery operations.

---

## PostgreSQL Configuration

You need two PostgreSQL users:

- **barman**: for management and backup initiation.

- **streaming_barman**: for WAL file streaming.

Create the users with:

```bash title="Create PostgreSQL Users"
createuser --superuser --replication -P barman
createuser --replication -P streaming_barman
```

Grant the necessary permissions for **barman**:

```sql title="Grant Required Permissions"
GRANT EXECUTE ON FUNCTION pg_start_backup(text, boolean, boolean) TO barman;
GRANT EXECUTE ON FUNCTION pg_stop_backup() TO barman;
GRANT EXECUTE ON FUNCTION pg_switch_wal() TO barman;
GRANT EXECUTE ON FUNCTION pg_create_restore_point(text) TO barman;
GRANT pg_read_all_settings TO barman;
GRANT pg_read_all_stats TO barman;
```

Update `pg_hba.conf`:

```text title="Update pg_hba.conf"
host replication streaming_barman all md5
```

Check and update `postgresql.conf`:

```text title="postgresql.conf Settings"
max_wal_senders = 10
max_replication_slots = 10
```

Reload PostgreSQL settings:

```sql title="Reload PostgreSQL Config"
SELECT pg_reload_conf();
```

Set up password authentication by creating a `.pgpass` file:

```bash title="Create .pgpass File"
su - barman
vim ~/.pgpass
```

Add credentials:

```text title=".pgpass Contents"
localhost:*:*:barman:<password>
localhost:*:*:streaming_barman:<password>
```

Secure the `.pgpass` file:

```bash title="Secure .pgpass"
chmod 0600 ~/.pgpass
```

Finally, set up SSH key-based access between the `barman` and `postgres` users for rsync operations.

---

## Barman Configuration

Create the backup directory and assign permissions:

```bash title="Create Backup Directory"
mkdir /pg_backup
chown barman:barman /pg_backup
```

Edit the main configuration file `/etc/barman.conf`:

```ini title="/etc/barman.conf"
barman_home = /pg_backup
recovery_options = get-wal
reuse_backup = link
compression = gzip
retention_policy_mode = auto
retention_policy = RECOVERY WINDOW OF 1 days
wal_retention_policy = main
```

Configure your server under `/etc/barman.d/pg.conf`:

```ini title="/etc/barman.d/pg.conf"
[pg]
description = "Example of PostgreSQL Database (via SSH) and WAL Streaming"
ssh_command = ssh postgres@localhost
conninfo = host=localhost user=barman dbname=postgres
backup_method = rsync
backup_options = concurrent_backup

streaming_conninfo = host=localhost user=streaming_barman
streaming_archiver = on
create_slot = auto
slot_name = barman
streaming_archiver_name = barman_receive_wal
streaming_archiver_batch_size = 50
```

Test the configuration:

```bash title="Test Barman Config"
barman cron
barman check pg
```

Ensure all status outputs are `OK`.

---

## Taking Backups

Manually trigger a backup:

```bash title="Manual Backup"
barman backup pg
```

List existing backups:

```bash title="List Backups"
barman list-backup pg
```

Backup directory structure:

- `base/` : Full base backups

- `wals/` : Archived WAL files

- `stream/` : Streaming WAL files

---

## Recovery from Backup

In the event of a failure, you can recover the PostgreSQL server from the backup.

Stop the PostgreSQL service:

```bash title="Stop PostgreSQL"
sudo systemctl stop postgresql
```

Recover the latest backup:

```bash title="Recover Latest Backup"
barman recover --remote-ssh-command 'ssh postgres@localhost' pg latest /database/PostgreSQL/13/main/
```

Or recover a specific backup:

```bash title="Recover Specific Backup"
barman recover --remote-ssh-command 'ssh postgres@localhost' pg 20231211T110303 /database/PostgreSQL/13/main/
```

After recovery, edit the `postgresql.auto.conf` file and set the restore command:

```bash title="Set Restore Command"
restore_command = 'barman-wal-restore -P -U barman localhost pg %f %p'
```

Then start the PostgreSQL service:

```bash title="Start PostgreSQL"
sudo systemctl start postgresql
```

!!! warning "Important Note"
    Always verify the restored database carefully before putting it back into production.

---

## Automating Backups with Cron

To schedule automatic daily backups, edit `/etc/cron.d/barman` and add:

```cron title="Cron Backup Job"
0 23 * * * barman [ -x /usr/bin/barman ] && /usr/bin/barman backup pg
```

This example runs backups every day at 23:00.

---

## Notes

!!! warning "Configuration Files Not Backed Up"
    Barman does **not** back up the following files:

    - `postgresql.conf`

    - `pg_hba.conf`

    - `pg_ident.conf`

    Always manually back up your configuration files if needed.

- For detailed Barman usage, visit [Barman Documentation](https://docs.pgbarman.org/release/3.9.0/index.html).

---

## Conclusion

This guide demonstrates how Barman can be configured on a **single server** to perform PostgreSQL backups easily.

While not recommended for production environments, this approach is very handy for demos, training labs, or testing backup strategies.

In future posts, we will explore **advanced recovery options** and **cross-server backup configurations**!

> **Written by:** [Ruchan Yalçın](https://www.ruchan.dev) | [GitHub](https://github.com/ruchany13)

---
