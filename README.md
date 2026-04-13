# 🚀 DRC Backup Automation (Ansible Playbook)

This repository contains **Ansible Playbooks** optimized for executing high-performance backup processes (Application Source Code & SQL Databases) to the *Disaster Recovery Center (DRC)*.

## ✨ Key Features
- **Asynchronous Parallel Processing**: Backup processes (Database Dumps, Folder Rsync, Tar) are now executed concurrently in the background, drastically cutting down rotation time compared to sequential processing.
- **Detailed Analytics Reporting**: Automatically measures storage usage changes per-application and local dump sizes per-database.
- **Smart Notification Ecosystem**: Modular multi-channel interactive alerts (Discord, Telegram, WhatsApp) with individual partial recaps (Success/Failed for each *Application* and *Database*).
- **Isolated Error Handling**: A failure in one application/database entity will not halt the backup process for other applications within the same node.

---

## 📚 System & Execution Guide

For readability and brevity, server setup instructions and playbook workflows have been separated for respective services. Please follow the instructions based on the roles your servers perform.

Choose your guide below:

### 👉 [📘 Application Backup Management Guide (Web/Apps)](README-Apps.md)
This documentation covers the execution system via `Backup-Apps.yml`. It includes:
- How to set up the `backup-drc` user & *Ansible Semaphore* authorization.
- *Access Control List (ACL) Bypass* tricks to safely back up paths inside `/var/www/html/`.
- The *Parallel Delay Interval Workflow* scheme we developed.

### 👉 [📙 Database Backup Management Guide (MariaDB/MySQL)](README-Mariadb.md)
This documentation covers the database remote backup processing via `Backup-Mariadb.yml`. It includes:
- Configuration of the `db_type` variable (Flexible support for **Mysql** or **Mariadb**).
- Database account access assignment and the `~/.conf/config.cnf` secret credential file generation.
- The asynchronous multi-threading *SQL dump* and efficient SSH transfer schemes.

---

## 🔔 Multi-channel Notification Integration (Alerting)

This playbook ecosystem no longer throws static "All Success" alerts. It reads the outcome of each item processed in parallel and reports the specific conditions to the notifications with aggregate graphical formatting (*Size tracking & error flags*).

These webhooks can be enabled and dynamically combined at the Inventory level using the `active_notifications` list parameter.
*Example configuration:*
```yaml
vars:
  active_notifications: ['discord', 'telegram', 'whatsapp']
```
