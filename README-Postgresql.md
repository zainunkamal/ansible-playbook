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

## ⚙️ Ansible Server / Semaphore Server Preparation

Karena playbook ini dijalankan secara **lokal** pada Ansible Server (`connection: local`), pastikan Ansible/Semaphore Server Anda telah terpasang utility klien PostgreSQL agar perintah `pg_dump` dan `psql` dapat dieksekusi secara remote:

```bash
# Ubuntu / Debian
sudo apt-get install -y postgresql-client

# CentOS / RHEL / Rocky Linux
sudo dnf install -y postgresql
```

> [!NOTE]
> Pastikan versi `pg_dump` di Ansible Server setara atau lebih tinggi dibandingkan versi PostgreSQL Server target agar proses restore dump berjalan tanpa masalah kompatibilitas.

### 🗝️ SSH Access to DRC Server
Ansible Server juga harus memiliki akses SSH *passwordless* (menggunakan private key) ke target **DRC Server** agar perintah `rsync` dan `ssh` pembersihan file lama dapat berjalan otomatis.

---

## ⚙️ Base Server Preparation (PostgreSQL / Patroni Server Target)

Keuntungan utama dari arsitektur terpusat ini adalah **tidak perlu melakukan instalasi SSH keys atau konfigurasi Unix user baru di server database target**. Anda cukup memberikan izin akses PostgreSQL secara remote:

### 🗄️ 1. PostgreSQL Access Profile Authorization
Masuk ke konsol PostgreSQL di salah satu server database (atau cluster VIP) untuk membuat user backup:
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
> [!TIP]
> Agar user backup dapat men-dump **seluruh** database secara otomatis tanpa memberikan hak akses satu per satu, Anda bisa memberikan role `pg_read_all_data` (tersedia mulai dari PostgreSQL 14+):
> ```sql
> GRANT pg_read_all_data TO backup_user;
> ```

### 🌐 2. PostgreSQL Remote Network Access (pg_hba.conf)
Pastikan PostgreSQL Server target mengizinkan koneksi dari IP **Ansible Server**. Tambahkan baris berikut di file `pg_hba.conf` server PostgreSQL target Anda:

```text
# Mengizinkan koneksi dari IP Ansible Server
host    all             backup_user     <IP_ANSIBLE_SERVER>/32   md5
```
Jangan lupa untuk me-reload servis PostgreSQL / Patroni setelah mengubah konfigurasi network.

---

## ⚙️ Inventory and Variable Preparation (Ansible)
### Inventory
Buat PostgreSQL Ansible Inventory (contoh: `Inventory-pg.yml`) seperti berikut:
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
          # pg_host: "localhost"       # (Opsional) Gunakan jika database dipanggil melalui VIP/lainnya. Default: IP host (<IP_ADDRESS_PG>)
          pg_port: "5432"            # PostgreSQL port (default: 5432)
          pg_user: "backup_user"     # PostgreSQL user untuk pg_dump
          pg_password: "YOURPASSWORD" # PostgreSQL password (sangat disarankan di-encrypt pakai ansible-vault!)
          # pg_default_db: "postgres"  # (Opsional) Database default untuk query list DB. Default: postgres
          
          # [Optional] Additional pg_dump flags
          pg_dump_extra_opts: "--no-owner --no-acl"

          # [Exclusions] Database yang ingin di-skip dari backup
          exclude_dbs: 
            - "postgres"
```
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
