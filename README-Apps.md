# 📘 Application Backup Guide (Web/Apps)

The primary file for this execution is **`playbook/Backup-Apps.yml`**. This playbook ensures that all Application *Source Code* on the App Server is securely distributed to the *DRC Server*, without localized failure interruptions.

## ⚡ Workflow
In its new schema, the playbook operates using a *High-Performance Async* system:
1. **Parallel Rsync**: The playbook calculates the application configurations (from `Inventory-apps.yml`) and asynchronously triggers the `rsync` execution (simultaneously parallel) into the background. Progress checks are stabilized at 1-minute intervals (to prevent `FAILED - RETRYING` spam logs).
2. **Size Checking**: The playbook queries the directory *size* on the target (DRC Server) before and after the *Rsync*.
3. **Daily & Monthly Tar Compression**: Compresses the target HTML source codes into `.tar.gz` bundles. This is executed daily, as well as an exclusive rotation every 1st of the month using the same parallel execution method.
4. **Scheduled Cleanup**: Scans and deletes expired archival histories (`find -mtime`).
5. **Partial Report & Notification**: Initializes an *individual report array*, sorts out successful applications, calculates capacity differences, and tags failed applications without halting the rotation map for other applications on the list. The report is delivered neatly to Discord/Telegram/WhatsApp.

---

## ⚙️ Base Server Preparation (App Server)

You must configure account permission settings on the initial target server so that the Ansible Semaphore bot can act securely and automatically (`passwordless`).

Run the following sequences using `root` or `sudo` access on the application server:

**1. Create a dedicated sync user**
```bash
sudo adduser --system --group --shell /bin/bash backup-drc
```

**2. Install *Access Control List (ACL)* Management**
```bash
sudo apt update
sudo apt install acl -y
```

**3. Bypass Read Limitation (*Set ACL for application directories*)**
This function allows the *backup-drc* user to read and execute the source folders without being able to write or corrupt them.
```bash
sudo setfacl -R -m u:backup-drc:rx /var/www/html/
```

**4. Generate Access Keys (SSH Passwordless to the DRC port)**
```bash
sudo su - backup-drc
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
# Send the public key to DRC (Make sure SSH ports are matched if non-standard)
ssh-copy-id -i ~/.ssh/id_rsa.pub <USER_DRC>@<IP_TARGET_DRC> -p <PORT_SSH>
```

**5. Authorize Ansible Semaphore Keystore**
So that Ansible can remote/login into this Linux account, you must place the Semaphore Server's *SSH Public Key* into the trusted list.
```bash
cat << 'EOF' >> ~/.ssh/authorized_keys
# Place your Ansible Semaphore Pubkey below this line:
ssh-rsa AAAAB3NzaC1yc...
EOF

# Lock the permissions
chmod 600 ~/.ssh/authorized_keys
```

*(Perform the above integrations across all new App Servers you register in the Inventory).*

## ⚙️ Inventory and Variable Configuration (Ansible)

To execute this playbook successfully, your Ansible Inventory (e.g., `Inventory-apps.yml`) must be structured with the app definitions intact. Here is a reference configuration:

```yaml
all:
  children:
    app_servers:
      hosts:
        server_app_1:
          ansible_host: x.x.x.x # IP ADdress
          ansible_user: "semaphore" # Ansible SSH User
          
          # [Target Identifiers]
          server_alias: "Web Application Farm"
          backup_user: "backup-drc"
          backup_ip: "x.x.x.x"     # DRC Destination IP
          backup_path: "/data/backup" # Root DRC Target Directory
          
          # [App Manifest] List apps processed on this specific node
          apps:
            - name: "NAME_APPS1"
              src: "/var/www/html/apps1"
              exclude: [".env", "storage/logs"] # Prevent junk data sync
            - name: "NAME_APPS2"
              src: "/var/www/html/apps2"
```
Variable for notification apps
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
⬅️ *[Back to Main Page](README.md)*
