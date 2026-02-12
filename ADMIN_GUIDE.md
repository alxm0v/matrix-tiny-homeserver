# Administrator Guide

This guide describes how to manage your Matrix (Tuwunel) and LiveKit server.

## Prerequisites
- **Admin Token**: You need the `matrix_registration_token` from your `inventory/group_vars/all.yml` (locally) or the server configuration.
- **Note on E2EE**: The "Server Admin" room (where you chat with the bot) is **Not Encrypted** by design. This is normal. All other private chats between users *will* be End-to-End Encrypted.
- **Server Access**: Commands below are best run from your local machine (targeting `https://example.com`) or from the VPS itself.

## User Management
Tuwunel implements the Synapse Admin API. You can use raw `curl` commands to manage users.

### 1. Create a User
Use this command to create a new user manually (even if registration is disabled).

```bash
# Replace <admin_token>, <username>, <password>, and <display_name>
curl -X POST \
  -H "Authorization: Bearer <YOUR_ADMIN_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
        "password": "<USER_PASSWORD>",
        "displayname": "<DISPLAY_NAME>",
        "admin": false
      }' \
  https://example.com/_synapse/admin/v2/users/@<username>:example.com
```
*Note: Users can subsequently change their own password via their client settings (e.g., Element -> Settings -> General -> Password).*

### 2. Reset User Password
```bash
curl -X PUT \
  -H "Authorization: Bearer <YOUR_ADMIN_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
        "password": "<NEW_PASSWORD>",
        "logout_devices": true
      }' \
  https://example.com/_synapse/admin/v2/users/@<username>:example.com/password
```

### 3. Deactivate (Ban) User
```bash
curl -X POST \
  -H "Authorization: Bearer <YOUR_ADMIN_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"erase": true}' \
  https://example.com/_synapse/admin/v2/users/@<username>:example.com/deactivate
```

### Alternative: Synapse Admin UI
For a graphical interface, you can call use a web-based tool like [Synapse Admin](https://github.com/Awesome-Technologies/synapse-admin).
1.  Open the Synapse Admin page (often hosted publicly or run via Docker locally).
2.  **Homeserver URL**: `https://example.com`
3.  **Username**: `@admin:example.com` (your admin user)
4.  **Password**: Your admin password.

## Server Management via Ansible/Docker

### Check Logs
SSH into your server and run:
```bash
cd /opt/matrix-stack
docker compose logs -f tuwunel
# OR for livekit
docker compose logs -f livekit
```

### Restart Services
```bash
cd /opt/matrix-stack
docker compose restart
```

### Update Server
1. Change `matrix_version` or `livekit_version` in `roles/matrix_stack/defaults/main.yml`.
2. Run the playbook again:
   ```bash
   ansible-playbook -i inventory/hosts.yml deploy.yml
   ```

### Quick Re-Hardening
If you have `perform_hardening: false` in your config but want to force a run (e.g., to install Fail2Ban):
```bash
ansible-playbook -i inventory/hosts.yml deploy.yml -e "perform_hardening=true"
```

## Troubleshooting

### Emergency Memory Cleanup (OOM)
If the server becomes unresponsive or updates fail due to "Out of Memory" errors:

1.  **Stop all services**:
    ```bash
    cd /opt/matrix-stack
    ```bash
    cd /opt/matrix-stack
    docker compose down
    # NOTE: This command preserves your data (named volumes).
    # WARNING: 'docker compose down -v' WILL DELETE YOUR DATABASE PERMANENTLY.
    ```
2.  **Kill stray processes**:
    ```bash
    docker rm -f $(docker ps -aq)
    ```
3.  **Drop Caches**:
    ```bash
    sync; echo 3 > /proc/sys/vm/drop_caches
    ```
4.  **Reboot (Last Resort)**:
    if commands hang, force a reboot via your provider's control panel.

## Backup & Restore (RocksDB)

Since Tuwunel uses an embedded RocksDB database, backing it up is simple. You just need to back up the data directory.

### Backup
1.  **Stop Tuwunel** (Critical to ensure data consistency):
    ```bash
    cd /opt/matrix-stack
    docker compose stop tuwunel
    ```
2.  **Archive the Data Volume**:
    Run a temporary container to zip the volume content to your current directory:
    ```bash
    docker run --rm --volumes-from matrix-stack-tuwunel-1 -v $(pwd):/backup ubuntu tar cvzf /backup/tuwunel_backup_$(date +%F).tar.gz /var/lib/tuwunel
    ```
3.  **Start Tuwunel**:
    ```bash
    docker compose start tuwunel
    ```
4.  **Download**: Copy the `.tar.gz` file to your local machine using SCP.

### Restore
1.  **Stop Tuwunel**: `docker compose stop tuwunel`
2.  **Restore Data**:
    ```bash
    # Extract backup back into the volume
    docker run --rm --volumes-from matrix-stack-tuwunel-1 -v $(pwd):/backup ubuntu bash -c "cd / && tar xvzf /backup/tuwunel_backup_YYYY-MM-DD.tar.gz"
    ```
3.  **Start Tuwunel**: `docker compose start tuwunel`
