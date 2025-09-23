# Ansible Role: The Bastion

Installs and configures [The Bastion](https://github.com/ovh/the-bastion) SSH jump host on Debian systems. Supports new installations, upgrades, and high availability setups.

## Features

- **Automatic mode detection**: New installation or upgrade based on version comparison
- **High Availability**: Master-slave clustering with automatic synchronization  
- **Home encryption**: Optional LUKS encryption for home partition
- **GPG key management**: Automatic setup of encryption and signature keys for ttyrec files and backups

## Requirements

- Debian 12 or 13

## Quick Start

```yaml
- hosts: bastion
  become: true
  vars:
    bastion_first_admin:
      username: "admin"
      ssh_public_key: "ssh-ed25519 AAAA..."
  roles:
    - adfinis.the_bastion
```

## Configuration

### Basic Settings

```yaml
# Version and installation
bastion_version: "v3.22.00"
bastion_install_dir: /opt/bastion
bastion_install_method: tarball  # or 'git'

# First admin (required for new installations)
bastion_first_admin:
  username: "admin"
  ssh_public_key: "ssh-ed25519 AAAA..."

# Optional features
bastion_encrypt_home: false
bastion_install_syslog_ng: true
```

### High Availability

Configure master-slave clustering for redundancy:

```yaml
# HA configuration
bastion_ha_enabled: true
bastion_ha_role: master  # or 'slave'

# Master settings
bastion_ha_master_ip: "192.168.1.10"
bastion_ha_slave_ips:
  - "192.168.1.11"
  - "192.168.1.12"

# SSH key paths for synchronization
bastion_ha_ssh_key_path: "/root/.ssh/id_master2slave"
bastion_ha_sync_user: "bastionsync"
```

### Home Encryption

>[!WARNING]
> Make sure your /home is on a separate partition!

Encrypt user directories with LUKS:

```yaml
bastion_encrypt_home: true
bastion_encryption_passphrase: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  66386439653866323331316663346232633933663839316264663165653239363138373862373765
```

### GPG Keys for Encryption and Signature

The Bastion uses two types of GPG keys:

1. **Bastion GPG Key**: Used by the bastion to sign ttyrec files (proves authenticity)
2. **Admin GPG Keys**: Used to encrypt backups and ttyrec files (for admin decryption)

```yaml
# Enable GPG functionality
bastion_gpg_enabled: true

# Bastion GPG key (automatic generation recommended)
bastion_gpg_key_generate: true

# Admin GPG keys (REQUIRED - one or more public keys)
bastion_admin_gpg_keys:
  - |
    -----BEGIN PGP PUBLIC KEY BLOCK-----
    mQENBF...  # First admin public key
    -----END PGP PUBLIC KEY BLOCK-----
  - |
    -----BEGIN PGP PUBLIC KEY BLOCK-----
    mQGNBF...  # Second admin public key (optional)
    -----END PGP PUBLIC KEY BLOCK-----
```

**Requirements:**
- At least one admin public key must be provided when GPG is enabled
- Admin keys must be generated on secure workstations, NOT on the bastion
- Multiple admin keys are supported

#### Generating Admin GPG Keys on Your Local Machine

Generate an ed25519 GPG key on your secure workstation:

```bash
# Set your details
export GPG_NAME="Bastion Admin"
export GPG_EMAIL="admin@example.com"
export GPG_COMMENT="The Bastion Admin Key"

# Generate a secure passphrase
export GPG_PASSPHRASE=$(pwgen -sy 16 1)
echo "Generated passphrase: $GPG_PASSPHRASE"

# Generate ed25519 key (recommended)
gpg --batch --pinentry-mode loopback --passphrase-fd 0 \
    --quick-generate-key "$GPG_NAME ($GPG_COMMENT) <$GPG_EMAIL>" ed25519 sign 0 <<< "$GPG_PASSPHRASE"

# Get the key fingerprint
GPG_FPR=$(gpg --list-keys "$GPG_EMAIL" | grep -Eo '[A-F0-9]{40}')

# Add encryption subkey
gpg --batch --pinentry-mode loopback --passphrase-fd 0 \
    --quick-add-key "$GPG_FPR" cv25519 encr 0 <<< "$GPG_PASSPHRASE"

# Export public key for the bastion
echo "=== Copy this public key to your Ansible configuration ==="
gpg -a --export "$GPG_EMAIL"

# Export private key for secure backup
echo "=== Save this private key securely (NOT on the bastion) ==="
gpg --export-secret-keys --armor "$GPG_EMAIL"
```

For RSA 4096 compatibility (older GnuPG versions):

```bash
# Alternative: RSA 4096 key generation
cat << EOF | gpg --batch --gen-key
Key-Type: RSA
Key-Length: 4096
Subkey-Type: RSA
Subkey-Length: 4096
Name-Real: $GPG_NAME
Name-Comment: $GPG_COMMENT
Name-Email: $GPG_EMAIL
Expire-Date: 0
Passphrase: $GPG_PASSPHRASE
%echo Generating GPG key
%commit
%echo done
EOF
```

**Security Notes:**
- Store private keys and passphrases securely (password manager/vault)
- Share private keys with all administrators who need to decrypt files
- Test configuration: `/opt/bastion/bin/cron/osh-encrypt-rsync.pl --config-test`

### External Account Validation

The Bastion supports external account validation through custom scripts that can verify account status against external systems (LDAP, Active Directory, APIs, etc.):

```yaml
# Enable external account validation
bastion_external_validation_enabled: true

# Option 1: Deploy script content directly
bastion_external_validation_program_content: |
  #!/usr/bin/env bash
  ACCOUNT="$1"
  # Check if account exists in allowed accounts file
  if grep -q "^${ACCOUNT}$" /etc/bastion/allowed_accounts.txt; then
    exit 0  # Account is active
  else
    exit 1  # Account is inactive
  fi

# Option 2: Use a template (alternative to content)
bastion_external_validation_program_template: "external-validation-custom.sh.j2"

# Configuration options
bastion_external_validation_program_path: "/opt/bastion/bin/other/check-active-account-custom.sh"
bastion_external_validation_program_mode: "0755"
bastion_external_validation_deny_on_failure: true
```

**Important Notes:**
- Either `bastion_external_validation_program_content` OR `bastion_external_validation_program_template` must be provided
- The script receives the account name as the first argument
- Exit codes: 0 (active), 1 (inactive), 2-4 (various failure modes)

#### LDAP Validation Example

This role includes an example LDAP validation template (`templates/check-active-account-ldap.sh.j2`) that demonstrates integration with LDAP servers for account validation. This template provides:

- LDAP server connectivity with optional authentication
- Account validation based on non-expiring accounts
- Optional group membership requirements
- Caching mechanism to reduce LDAP queries

```yaml
# Use the provided LDAP validation template
bastion_external_validation_enabled: true
bastion_external_validation_program_template: "check-active-account-ldap.sh.j2"

# LDAP server configuration
bastion_external_validation_ldap_server: "ldap.example.com"
bastion_external_validation_ldap_base_dn: "ou=users,dc=example,dc=com"
bastion_external_validation_ldap_bind_dn: "cn=readonly,dc=example,dc=com"
bastion_external_validation_ldap_bind_password: "{{ vault_ldap_password }}"

# Optional: require group membership
bastion_external_validation_ldap_required_group: "cn=bastion-users,ou=groups,dc=example,dc=com"

# Caching configuration
bastion_external_validation_ldap_cache_file: "/var/cache/bastion/active_accounts.cache"
bastion_external_validation_ldap_cache_ttl: 300  # 5 minutes

# TLS configuration
bastion_external_validation_ldap_ignore_tls: false  # Set to true for testing only
```

**Important:** This is only an example template that demonstrates LDAP integration concepts. You will likely need to adjust the LDAP queries, filters, and logic to match your specific LDAP schema, security requirements, and organizational policies. The template should be reviewed and customized before production use.

### Backup Configuration

The Bastion role supports two types of backups:

1. **Ansible Role Backups**: Created before upgrades to preserve system state
2. **Bastion ACL Backups**: The Bastion's own backup system for keys and configuration

#### Backup Directory Separation

The role ensures these backup systems use separate directories to avoid conflicts:

```yaml
# Ansible role backups (created before upgrades)
bastion_backup_dir: "/root/ansible-backups"

# Bastion's own backup system
bastion_acl_backup_destdir: "/var/backups/bastion"
```

#### ACL Backup Configuration

Configure The Bastion's built-in backup system:

```yaml
# Enable/disable bastion backup system
bastion_acl_backup_enabled: "1"

# Backup retention
bastion_acl_backup_days_to_keep: "90"

# GPG encryption for backups
bastion_acl_backup_gpg_keys: "41FDB9C7 DA97EFD1"  # Space-separated GPG key IDs
bastion_acl_backup_signing_key: "41FDB9C7"  # Key for signing backups
bastion_acl_backup_signing_key_passphrase: "secure_passphrase"

# Remote backup push
bastion_acl_backup_push_remote: "backup@backup-server:/var/backups/bastion/"
bastion_acl_backup_push_options: "-i /root/.ssh/id_backup"

# Logging
bastion_acl_backup_log_facility: "local6"
bastion_acl_backup_logfile: ""  # Empty means use syslog only
```

#### Encrypt-Rsync Configuration

Configure The Bastion's TTYrec encryption and remote sync system:

```yaml
# Enable/disable encrypt-rsync system
bastion_encrypt_rsync_enabled: true

# Logging configuration
bastion_encrypt_rsync_logfile: ""  # Empty means use syslog only
bastion_encrypt_rsync_syslog_facility: "local6"
bastion_encrypt_rsync_verbose: "0"  # 0=normal, 1=verbose, 2=debug

# GPG signing and encryption (mandatory)
bastion_encrypt_rsync_signing_key: "41FDB9C7"  # GPG key for signing
bastion_encrypt_rsync_signing_key_passphrase: "secure_passphrase"

# Multi-layer encryption recipients
bastion_encrypt_rsync_recipients:
  - ["AAAAAAAA", "BBBBBBBB"]  # First layer (auditors)
  - ["CCCCCCCC", "DDDDDDDD"]  # Second layer (sysadmins)

# File handling
bastion_encrypt_rsync_encrypt_dir: "/home/.encrypt"
bastion_encrypt_rsync_ttyrec_delay_days: "14"     # Days before encrypting ttyrecs
bastion_encrypt_rsync_user_logs_delay_days: "31"  # Days before encrypting logs
bastion_encrypt_rsync_user_sqlites_delay_days: "31"  # Days before encrypting sqlite files

# Remote sync (optional)
bastion_encrypt_rsync_destination: "backup@remote:/backups/bastion/"
bastion_encrypt_rsync_rsh: "ssh -p 222 -i /root/.ssh/id_backup"
bastion_encrypt_rsync_delay_before_remove_days: "7"  # Days before removing after sync
```

### Custom Configuration

```yaml
bastion_config:
  bastionName: "my-friendly-bastion"
  adminAccounts: "admin,backup-admin"
  enabledGlobalCodes: "ssh,sftp"
  defaultAccountTTL: "90"
```

## Examples

### Single Bastion

```yaml
- hosts: bastion
  become: true
  vars:
    bastion_first_admin:
      username: "admin"
      ssh_public_key: "ssh-rsa AAAAB3... admin@company.com"
    bastion_encrypt_home: true
    bastion_encryption_passphrase: "{{ vault_bastion_passphrase }}"
  roles:
    - adfinis.the_bastion
```

### HA Master-Slave Setup

```yaml
# Master bastion
- hosts: bastion_master
  become: true
  vars:
    bastion_ha_enabled: true
    bastion_ha_role: master
    bastion_ha_master_ip: "{{ ansible_default_ipv4.address }}"
    bastion_ha_slave_ips:
      - "192.168.1.11"
      - "192.168.1.12"
    bastion_first_admin:
      username: "admin"
      ssh_public_key: "ssh-rsa AAAAB3... admin@company.com"
  roles:
    - adfinis.the_bastion

# Slave bastions
- hosts: bastion_slaves
  become: true
  vars:
    bastion_ha_enabled: true
    bastion_ha_role: slave
    bastion_ha_master_ip: "192.168.1.10"
    bastion_config:
      readOnlySlaveMode: true
  roles:
    - adfinis.the_bastion
```

### Upgrade Example

```yaml
- hosts: bastion
  become: true
  vars:
    bastion_version: "v3.22.00"  # Higher version triggers upgrade
    bastion_upgrade_backup_before: true
  roles:
    - adfinis.the_bastion
```

### GPG-Enabled Bastion

```yaml
- hosts: bastion
  become: true
  vars:
    bastion_first_admin:
      username: "admin"
      ssh_public_key: "ssh-ed25519 AAAA... admin@company.com"
    
    # Enable GPG encryption and signing
    bastion_gpg_enabled: true
    bastion_gpg_key_generate: true
    
    # Import admin public keys for encryption (required)
    bastion_admin_gpg_keys:
      - |
        -----BEGIN PGP PUBLIC KEY BLOCK-----
        mQENBGH8...
        -----END PGP PUBLIC KEY BLOCK-----
      - |
        -----BEGIN PGP PUBLIC KEY BLOCK-----
        mQGNBGJ9...
        -----END PGP PUBLIC KEY BLOCK-----
  roles:
    - adfinis.the_bastion
```

### Bastion with Backup Configuration

```yaml
- hosts: bastion
  become: true
  vars:
    bastion_first_admin:
      username: "admin"
      ssh_public_key: "ssh-ed25519 AAAA... admin@company.com"
    
    # Configure Bastion's backup system
    bastion_acl_backup_enabled: "1"
    bastion_acl_backup_destdir: "/var/backups/bastion"
    bastion_acl_backup_days_to_keep: "30"
    
    # GPG encryption for backups
    bastion_acl_backup_gpg_keys: "41FDB9C7 DA97EFD1"
    
    # Remote backup push (optional)
    bastion_acl_backup_push_remote: "backup@backup-server:/backups/bastion/"
    bastion_acl_backup_push_options: "-i /root/.ssh/id_backup"
  roles:
    - adfinis.the_bastion
```

### Bastion with TTYrec Encryption and Remote Sync

```yaml
- hosts: bastion
  become: true
  vars:
    bastion_first_admin:
      username: "admin"
      ssh_public_key: "ssh-ed25519 AAAA... admin@company.com"
    
    # Enable GPG for encryption/signing
    bastion_gpg_enabled: true
    bastion_gpg_key_generate: true
    
    # Configure TTYrec encryption and remote sync
    bastion_encrypt_rsync_enabled: true
    
    # Multi-layer encryption for compliance
    bastion_encrypt_rsync_recipients:
      - ["AUDITOR1", "AUDITOR2"]    # Auditors can decrypt
      - ["SYSADMIN1", "SYSADMIN2"]  # Sysadmins handle encrypted files
    
    # Retention and sync settings
    bastion_encrypt_rsync_ttyrec_delay_days: "7"   # Quick encryption
    bastion_encrypt_rsync_destination: "backup@archive.company.com:/secure/bastion/"
    bastion_encrypt_rsync_rsh: "ssh -p 2222 -i /root/.ssh/id_backup"
    bastion_encrypt_rsync_delay_before_remove_days: "30"  # Keep local copies for 30 days
  roles:
    - adfinis.he_bastion
```

## How It Works

### Automatic Mode Detection

The role determines the action based on version comparison:

- **New Install**: No existing installation found
- **Upgrade**: Target version > installed version  
- **Skip**: Target version <= installed version

### High Availability

HA setup creates a master-slave cluster:

1. **Master**: Handles all configuration changes
2. **Slaves**: Read-only replicas synchronized via rsync
3. **Sync Daemon**: Uses inotify to detect changes and push to slaves
4. **SSH Keys**: Secure authentication between master and slaves

### Installation Process

1. System preparation and package updates
2. Download The Bastion source code
3. Install dependencies and optional packages
4. Configure home encryption (if enabled)
5. Run installation script
6. Create admin account and configure SSH
7. Set up HA synchronization (if enabled)

## Variables Reference

### Core Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `bastion_version` | `"v3.22.00"` | Version to install/upgrade to |
| `bastion_install_dir` | `/opt/bastion` | Installation directory |
| `bastion_install_method` | `tarball` | Source method: `tarball` or `git` |
| `bastion_first_admin` | `{}` | First admin account config |

### HA Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `bastion_ha_enabled` | `false` | Enable HA clustering |
| `bastion_ha_role` | `master` | Role: `master` or `slave` |
| `bastion_ha_master_ip` | `""` | Master bastion IP address |
| `bastion_ha_slave_ips` | `[]` | List of slave IP addresses |
| `bastion_ha_ssh_key_path` | `/root/.ssh/id_master2slave` | SSH key for sync |
| `bastion_ha_sync_user` | `bastionsync` | User for synchronization |

### GPG Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `bastion_gpg_enabled` | `false` | Enable GPG encryption and signing |
| `bastion_gpg_key_generate` | `true` | Generate bastion GPG key automatically |
| `bastion_admin_gpg_keys` | `[]` | List of admin GPG public keys (required when GPG enabled) |

### Backup Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `bastion_backup_dir` | `"/root/ansible-backups"` | Directory for Ansible role backups |
| `bastion_acl_backup_enabled` | `"1"` | Enable Bastion's ACL backup system |
| `bastion_acl_backup_destdir` | `"/var/backups/bastion"` | Directory for Bastion's own backups |
| `bastion_acl_backup_days_to_keep` | `"90"` | Days to keep old backups |
| `bastion_acl_backup_logfile` | `""` | Log file path (empty = syslog only) |
| `bastion_acl_backup_log_facility` | `"local6"` | Syslog facility for logging |
| `bastion_acl_backup_gpg_keys` | `""` | Space-separated GPG key IDs for encryption (default: bastion_admin_gpg_keys) |
| `bastion_acl_backup_signing_key` | `""` | GPG key ID for signing backups (default: autogenerated bastion key) |
| `bastion_acl_backup_signing_key_passphrase` | `""` | Passphrase for signing key (default: autogenerated bastion key) |
| `bastion_acl_backup_push_remote` | `""` | Remote host for backup push (scp format) |
| `bastion_acl_backup_push_options` | `""` | Additional options for scp |

### Encrypt-Rsync Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `bastion_encrypt_rsync_enabled` | `true` | Enable encrypt-rsync system |
| `bastion_encrypt_rsync_logfile` | `""` | Log file path (empty = syslog only) |
| `bastion_encrypt_rsync_syslog_facility` | `"local6"` | Syslog facility for logging |
| `bastion_encrypt_rsync_verbose` | `"0"` | Verbosity level (0=normal, 1=verbose, 2=debug) |
| `bastion_encrypt_rsync_signing_key` | `""` | GPG key ID for signing ttyrec files (will default to autogenerated bastion key) |
| `bastion_encrypt_rsync_signing_key_passphrase` | `""` | Passphrase for signing key (will default to autogenerated bastion key) |
| `bastion_encrypt_rsync_recipients` | `[]` | Multi-layer encryption recipients (array of arrays, default: bastion_admin_gpg_keys) |
| `bastion_encrypt_rsync_encrypt_dir` | `"/home/.encrypt"` | Directory for encrypted files |
| `bastion_encrypt_rsync_ttyrec_delay_days` | `"14"` | Days before encrypting ttyrec files |
| `bastion_encrypt_rsync_user_logs_delay_days` | `"31"` | Days before encrypting user logs (min 31) |
| `bastion_encrypt_rsync_user_sqlites_delay_days` | `"31"` | Days before encrypting user sqlite files (min 31) |
| `bastion_encrypt_rsync_destination` | `""` | Rsync destination (empty = disable rsync) |
| `bastion_encrypt_rsync_rsh` | `""` | SSH command for rsync |
| `bastion_encrypt_rsync_delay_before_remove_days` | `"0"` | Days before removing local files after rsync |

### External Validation Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `bastion_external_validation_enabled` | `false` | Enable external account validation |
| `bastion_external_validation_program_path` | `/opt/bastion/bin/other/check-active-account-custom.sh` | Path where validation program will be deployed |
| `bastion_external_validation_program_content` | `""` | Content of validation script (alternative to template) |
| `bastion_external_validation_program_template` | `""` | Template file for validation script (alternative to content) |
| `bastion_external_validation_program_mode` | `"0755"` | File permissions for validation program |
| `bastion_external_validation_deny_on_failure` | `true` | Deny access when validation program fails |

#### LDAP Validation Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `bastion_external_validation_ldap_server` | `""` | LDAP server hostname |
| `bastion_external_validation_ldap_base_dn` | `""` | LDAP base DN for user searches |
| `bastion_external_validation_ldap_bind_dn` | `""` | LDAP bind DN for authentication (optional) |
| `bastion_external_validation_ldap_bind_password` | `""` | LDAP bind password (optional) |
| `bastion_external_validation_ldap_ignore_tls` | `false` | Ignore TLS certificate validation (testing only) |
| `bastion_external_validation_ldap_required_group` | `""` | Required LDAP group membership (optional) |
| `bastion_external_validation_ldap_cache_file` | `/var/cache/bastion/active_accounts.cache` | Cache file for LDAP results |
| `bastion_external_validation_ldap_cache_ttl` | `300` | Cache TTL in seconds |

### Optional Features

| Variable | Default | Description |
|----------|---------|-------------|
| `bastion_encrypt_home` | `false` | Encrypt /home with LUKS |
| `bastion_encryption_passphrase` | `""` | LUKS encryption passphrase |
| `bastion_install_syslog_ng` | `true` | Install syslog-ng |
| `bastion_install_optional_packages` | `true` | Install various optional packages (like `mosh` and `libpam-google-authenticator`) |
| `bastion_config` | `{}` | Custom bastion.conf options |

## Testing

Run Molecule tests:

> [!NOTE]
> The Bastion creates users with high UIDs which most likely exceed your `MAX_UID`. I couldn't get podman to work with those UIDs in userspace, so sudo it is (unfortunately).

```bash
# Test default scenario
sudo molecule test

# Test HA scenario  
sudo molecule test -s ha
```

## License

GPL-3.0-or-later

## Author

Created by [Adfinis AG](https://adfinis.com/) | [GitHub](https://github.com/adfinis)
