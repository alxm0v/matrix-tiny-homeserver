# Matrix & LiveKit Infrastructure

Deployment of a private Matrix homeserver (Tuwunel) and LiveKit server using Ansible, Docker, and Traefik.

## Prerequisites
- **Control Node**: WSL (Windows Subsystem for Linux), Ubuntu/Debian preferred.
- **Managed Node**: VPS with Ubuntu 24.04 LTS.
- **Domain**: `example.com`
  - **DNS Records**:
    - `A` record for `example.com` -> VPS IP (Proxy status: DNS Only)
    - `A` record for `livekit.example.com` -> VPS IP (Proxy status: DNS Only)

## Setup
1. **Inventory**: 
   - Copy the example inventory:
     ```bash
     cp inventory/hosts.yml.example inventory/hosts.yml
     ```
   - Edit `inventory/hosts.yml` to set your SSH key path for user `ansible-admin`.
2. **Secrets**: 
   - Copy the example secrets file:
     ```bash
     cp inventory/group_vars/all.yml.example inventory/group_vars/all.yml
     ```
   - Edit `inventory/group_vars/all.yml` and fill in:
     - `livekit_api_key` / `livekit_api_secret`: Random keys.
     - `matrix_registration_token`: Token for creating users.
   
   **Generating Secrets:**
   You can generate secure random strings using `openssl` (available on most Linux/WSL systems):
   
   ```bash
   # Generate a 32-character random string (hex)
   openssl rand -hex 16
   
   # Or using base64 for slightly shorter keys
   openssl rand -base64 24
   ```
3. **Check Variables**: 
   - `roles/matrix_stack/defaults/main.yml` contains domain configurations.

## Deployment
Run the playbook from your WSL terminal:

```bash
# Verify connectivity
ansible all -i inventory/hosts.yml -m ping

# Check mode (Dry run)
ansible-playbook -i inventory/hosts.yml deploy.yml --check

# Real deployment
ansible-playbook -i inventory/hosts.yml deploy.yml
```

## Post-Deployment
- **Matrix**: `https://example.com`
  - **Element X**: Supported via native Sliding Sync (enabled in config).
- **LiveKit**: `https://livekit.example.com`
- **Administration**: See [ADMIN_GUIDE.md](ADMIN_GUIDE.md) for user management commands (create user, reset password, etc.).
- **Traefik Dashboard**: Not exposed securely by default.

## Development
To enable the pre-commit checks (syntax & secrets) in your local repository:
```bash
git config core.hooksPath githooks
```
This ensures that `ansible-playbook --syntax-check` runs before every commit and scans for potential secrets.

