# 📗 Database Backup Guide (PostgreSQL)

All instructions on this page are concluded to support execution on the **`playbook/Backup-Postgresql.yml`** file. This playbook ensures the relational extraction process of the *PostgreSQL* database to the *DRC Server* takes place simultaneously without per-database delays.

## ⚡ Workflow
In its new schema, the playbook operates using a high-iteration intelligence system:
1. **PostgreSQL Native Dump**: The playbook utilizes `pg_dump` with custom format (`-Fc`) to produce highly compressed, restorable database dumps for each database individually.
2. **Parallel pg_dump**: Selectively queries `pg_database` system catalog to fetch all non-template databases, eliminates system databases (`postgres`, `template0`, `template1`), and then drops the *pg_dump & Gzip Compress* executions asynchronously (parallelly) into the *background* system. Checks are lightened to once per minute to prevent Database CPU and Ansible Semaphore log *overhead*.
3. **Discrete Size Calculation**: Logs the *size* payload produced by local dump files per-database (`.dump.gz`) into a partial local virtual variable for analytical metric mapping in the *report*.
4. **Bulk Rsync Transmission**: Rather than sending multiple files piecemeal per database entity name, Ansible executes a single bulk *Rsync* queue line to wrap the main `~/sql/` collection directory in a single send. This is done to preserve *TCP Overhead* and accelerate *I/O throughput* over SSH (*secure shell*) connections.
5. **Periodic Lifecycle (Cleanup)**: Like the applications, daily and monthly database *backups* on the DRC side will be incinerated and rotated automatically. Dump collection folders on the local PG machines are also completely purged to save main server storage disk utilization.
6. **Specific Fault Reporting**: Graphically broadcast to *multi-channel alerts* detailing the *gzip output* sizes per database, and discriminates the detailed status of failed databases without sounding redundant warnings.

---

## ⚙️ Ansible Server / Semaphore Server

In the ansible/semaphore server, we will need ssh-key to access the PostgreSQL server. The private key will be stored in the ansible/semaphore server and the public key will be stored in the PostgreSQL server.

## ⚙️ Base Server Preparation (PostgreSQL Server)

If your PostgreSQL server is not within the same OS environment scope as the *App Server*, perform additional OS environment configuration as in the following example:

### 🗄️ 1. PostgreSQL Access Profile Authorization
Enter the PostgreSQL console on the server (e.g., `sudo -u postgres psql`) and grant execution mandates so it can perform _Dumps_:
```sql
-- Create a dedicated backup user with minimal required privileges
CREATE ROLE backup_user WITH LOGIN PASSWORD 'YOURPASSWORD';

-- Grant CONNECT privilege on all databases
GRANT CONNECT ON DATABASE your_database TO backup_user;

-- Grant SELECT on all tables (run this inside each database)
\c your_database
GRANT USAGE ON SCHEMA public TO backup_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO backup_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO backup_user;
```
> [!NOTE]
> For the backup user to be able to dump **all** databases, you need to run the `GRANT` commands inside **each** database you want to back up. Alternatively, you can grant the `pg_read_all_data` role (PostgreSQL 14+):
> ```sql
> GRANT pg_read_all_data TO backup_user;
> ```

### 🛡️ 2. Client Credential & System Setup (`PGPASSWORD`)
The playbook authenticates `pg_dump` and `psql` commands using the `PGPASSWORD` environment variable, which is injected automatically at runtime from the Ansible inventory variable `pg_password`. This eliminates the need to manually create and maintain `.pgpass` files on every server node — especially beneficial for **Patroni/HA clusters** where multiple nodes exist.
```bash
# 1. Create Unix User (for Ansible SSH access)
sudo useradd --system --create-home --user-group --shell /bin/bash backup-user
sudo su - backup-user
```
> [!TIP]
> No `.pgpass` file is needed on the server. The password is stored centrally in the Ansible inventory (or Semaphore credentials) as `pg_password`, and passed to `pg_dump`/`psql` via the `PGPASSWORD` environment variable at execution time.

> [!IMPORTANT]
> For production environments, it is **strongly recommended** to encrypt the `pg_password` value using **Ansible Vault** or store it in the **Semaphore Keystore** to prevent plaintext password exposure in inventory files.

### 🗝️ 3. Push-Pull Authentication Gate Installation (DRC & Ansible)
Just as on the application server, establish the ed25519 _passwordless_ barricade:
```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
# Send the public key to DRC (Make sure SSH port is adjusted)
ssh-copy-id -i ~/.ssh/id_ed25519.pub <USER_DRC>@<IP_TARGET_DRC> -p <PORT_SSH>

# Unlock this server so Semaphore can navigate the Linux system and piggyback its commands going forward
cat << 'EOF' >> ~/.ssh/authorized_keys
# Place Ansible Semaphore Pubkey here:
ssh-rsa AAAAB3NzaC1yc...
EOF

# Set secure SSH permissions
chmod 600 ~/.ssh/authorized_keys
```

## ⚙️ Inventory and Variable Preparation (Ansible)
### Inventory
To execute this playbook efficiently, construct your PostgreSQL Ansible Inventory (e.g., `Inventory-pg.yml`) using the following mapping structure:
```yaml
all:
  children:
    pg_servers:
      vars:
        backup_ip: "x.x.x.x" # IP Server DRC
        backup_user: "USER_DRC" # User DRC
      hosts:
        <IP_ADDRESS_PG>:
          # [Target Identifiers & Variables]
          server_alias: "PostgreSQL Cluster Main"
          backup_path: "/data/backup" # Root DRC Target Directory
          
          # [PostgreSQL Connection Settings]
          pg_host: "localhost"       # PostgreSQL host address
          pg_port: "5432"            # PostgreSQL port (default: 5432)
          pg_user: "backup_user"     # PostgreSQL user for pg_dump
          pg_password: "YOURPASSWORD" # PostgreSQL password (use ansible-vault to encrypt!)
          
          # [Optional] Additional pg_dump flags
          pg_dump_extra_opts: "--no-owner --no-acl"

          # [Exclusions] Databases to skip explicitly
          exclude_dbs: 
            - "postgres"
```
### Variable
Variable for notification postgresql
```json
{
  // Enable your preferred channels (discord, telegram, whatsapp) you can chose more than 1
  "active_notifications": [
    "discord"
  ],

  // PUT YOUR API HERE
  "discord_webhook_url": [],
  "telegram_bot_token": [],
  "telegram_chat_id": [],
  "whatsapp_api_url": [],
  "whatsapp_api_key": [],
  "whatsapp_target_number": []
}
```
### Keystore
If using semaphore, you can use keystore to store your credentials. otherwise you can put it in the variable.
```json
{
  // Port SSH Server Apps
  "ansible_user": "backup-user",
  // Port SSH
  "ansible_port": 22 
  // We will use SSH Key for authentication
  // If you want to use password authentication, you can use the following:
  // "ansible_ssh_pass": "your_password",
  // "ansible_become_pass": "your_password"
}
```

⬅️ *[Back to Main Page](README.md)*
