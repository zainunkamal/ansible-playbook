# 📙 Database Backup Guide (MariaDB/MySQL)

All instructions on this page are concluded to support execution on the **`playbook/Backup-Mariadb.yml`** file. This playbook ensures the relational extraction process of the *SQL Engine* database to the *DRC Server* takes place simultaneously without per-database delays.

## ⚡ Workflow
In its new schema, the playbook operates using a high-iteration intelligence system and engine flexibility:
1. **Dynamic Engine Identification**: The playbook reads the configurable variable `db_type` in the inventory (choosing routes between native `mysql` or `mariadb` execution).
2. **Parallel SQL Dump**: Selectively calculates the list of all server databases, eliminates built-in static linux schemas, and then drops the *SQL Dump & Gzip Compress* executions asynchronously (parallelly) into the *background* system. Checks are lightened to once per minute to prevent Database CPU and Ansible Semaphore log *overhead*.
3. **Discrete Size Calculation**: Logs the *size* payload produced by local dump files per-database (`.sql.gz`) into a partial local virtual variable for analytical metric mapping in the *report*.
4. **Bulk Rsync Transmission**: Rather than sending multiple files piecemeal per database entity name, Ansible executes a single bulk *Rsync* queue line to wrap the main `~/sql/` collection directory in a single send. This is done to preserve *TCP Overhead* and accelerate *I/O throughput* over SSH (*secure shell*) connections.
5. **Periodic Lifecycle (Cleanup)**: Like the applications, daily and monthly database *backups* on the DRC side will be incinerated and rotated automatically. Dump collection folders on the local DB machines are also completely purged to save main server storage disk utilization.
6. **Specific Fault Reporting**: Graphically broadcast to *multi-channel alerts* detailing the *gzip output* sizes per database, and discriminates the detailed status of failed databases without sounding redundant warnings.

---

## ⚙️ Base Server Preparation (Database Server)

If your Database server is not within the same OS environment scope as the *App Server*, perform additional OS environment configuration as in the following example:

### 🗄️ 1. SQL Access Profile Authorization
Enter the MariaDB / MySQL console on the server (e.g., `mysql -u root -p`) and grant execution mandates so it can perform _Dumps_:
```sql
CREATE USER 'backup-drc'@'localhost' IDENTIFIED BY 'YOURPASSWORD';
GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER, PROCESS, RELOAD ON *.* TO 'backup-drc'@'localhost';
FLUSH PRIVILEGES;
```
> [!NOTE]
> If the *node executor* (Ansible Semaphore) runs remotely externally, and is not directly executing within the Linux OS where SQL natively runs as `localhost` (SSH Forwarding), you may need to alter `'localhost'` on the SQL *string* above to `'%'`.

### 🛡️ 2. Client Credential Shell & System Setup (config.cnf)
The creation of this CLI configuration file is absolute for security purposes, allowing the *automatic bot (Ansible)* to inject database *dumps* unobstructed by *password shell prompts* when the script runs covertly.
```bash
# 1. Create Unix User
sudo adduser --system --group --shell /bin/bash backup-drc
sudo su - backup-drc

# 2. Inject Secret Auth String
mkdir -p ~/.conf
cat << 'EOF' >> ~/.conf/config.cnf
[client]
user = "backup-drc"
password = "YOURPASSWORD"
host = "10.100.0.91"
port = "3306"
EOF

# 3. Lock Permissions from other Unix Users (Security)
chmod 700 ~/.conf
chmod 600 ~/.conf/config.cnf
```

### 🗝️ 3. Push-Pull Authentication Gate Installation (DRC & Ansible)
Just as on the application server, establish the RSA *(Rivest–Shamir–Adleman)* _passwordless_ barricade:
```bash
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
# Send the public key to DRC (Make sure SSH port is adjusted)
ssh-copy-id -i ~/.ssh/id_rsa.pub <USER_DRC>@<IP_TARGET_DRC> -p <PORT_SSH>

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
To execute this playbook efficiently, construct your Database Ansible Inventory (e.g., `Inventory-db.yml`) using the following mapping structure:
```yaml
all:
  children:
    db_servers:
      vars:
        backup_ip: "x.x.x.x" # IP Server DRC
        backup_user: "USER_DRC" # User DRC
      hosts:
        <IP_ADDRESS_DB>:
          # [Target Identifiers & Variables]
          server_alias: "Database Cluster Main"
          db_type: "mariadb" # Valid Options: 'mariadb' | 'mysql'
          backup_path: "/data/backup" # Root DRC Target Directory
          
          # [Exclusions] Databases to skip explicitly
          exclude_dbs: 
            - "mysql"
            - "information_schema"
            - "performance_schema"
            - "sys"
```
### Variable
Variable for notification mariadb
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
  "ansible_user": "backup-drc",
  // Port SSH
  "ansible_port": 22 
  // We will use SSH Key for authentication
  // If you want to use password authentication, you can use the following:
  // "ansible_ssh_pass": "your_password",
  // "ansible_become_pass": "your_password"
}
```

⬅️ *[Back to Main Page](README.md)*
