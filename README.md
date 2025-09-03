# Ansible Role: The Bastion

Installs and configures [The Bastion](https://github.com/ovh/the-bastion) SSH jump host on Debian systems. Supports new installations, upgrades, and high availability setups.

## Features

- **Automatic mode detection**: New installation, upgrade, or skip based on version comparison
- **High Availability**: Master-slave clustering with automatic synchronization  
- **Home encryption**: Optional LUKS encryption for user directories
- **GPG key management**: Automatic setup of encryption and signature keys for ttyrec files and backups
- **SSH hardening**: Automatic security configuration
- **Version management**: Safe upgrades with backup support

## Requirements

- Debian 12 or 13
- `community.general` Ansible collection

## Quick Start

```yaml
- hosts: bastion
  become: true
  vars:
    bastion_first_admin:
      username: "admin"
      ssh_public_key: "ssh-ed25519 AAAA..."
  roles:
    - ansible-role-the_bastion
```

## Configuration

### Basic Settings

```yaml
# Version and installation
bastion_version: "v3.21.00"
bastion_install_dir: /opt/bastion
bastion_install_method: tarball  # or 'git'

# First admin (required for new installations)
bastion_first_admin:
  username: "admin"
  ssh_public_key: "ssh-ed25519 AAAA..."

# Optional features
bastion_encrypt_home: false
bastion_install_syslog_ng: true
bastion_harden_ssh: true
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
- Multiple admin keys are supported for redundancy

#### Generating Admin GPG Keys on Your Local Machine

Generate an Ed25519 GPG key on your secure workstation:

```bash
# Set your details
export GPG_NAME="Bastion Admin"
export GPG_EMAIL="admin@example.com"
export GPG_COMMENT="The Bastion Admin Key"

# Generate a secure passphrase
export GPG_PASSPHRASE=$(pwgen -sy 16 1)
echo "Generated passphrase: $GPG_PASSPHRASE"

# Generate Ed25519 key (recommended)
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

### Custom Configuration

```yaml
bastion_config:
  adminAccounts: "admin,backup-admin"
  enabledGlobalCodes: "ssh,sftp"
  defaultAccountTTL: "90"
  readOnlySlaveMode: false  # Set to true on slaves
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
    - ansible-role-the_bastion
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
    - ansible-role-the_bastion

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
    - ansible-role-the_bastion
```

### Upgrade Example

```yaml
- hosts: bastion
  become: true
  vars:
    bastion_version: "v3.22.00"  # Higher version triggers upgrade
    bastion_upgrade_backup_before: true
  roles:
    - ansible-role-the_bastion
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
    - ansible-role-the_bastion
```

## How It Works

### Automatic Mode Detection

The role determines the action based on version comparison:

- **New Install**: No existing installation found
- **Upgrade**: Target version > installed version  
- **Skip**: Target version = installed version
- **Error**: Target version < installed version (downgrade not supported)

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
| `bastion_version` | `"v3.21.00"` | Version to install/upgrade to |
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
| `bastion_gpg_key_import` | `""` | Import existing bastion GPG key |
| `bastion_admin_gpg_keys` | `[]` | List of admin GPG public keys (required when GPG enabled) |

### Optional Features

| Variable | Default | Description |
|----------|---------|-------------|
| `bastion_encrypt_home` | `false` | Encrypt /home with LUKS |
| `bastion_encryption_passphrase` | `""` | LUKS encryption passphrase |
| `bastion_install_syslog_ng` | `true` | Install syslog-ng |
| `bastion_harden_ssh` | `true` | Apply SSH hardening |
| `bastion_config` | `{}` | Custom bastion.conf options |

## Testing

Run Molecule tests:

```bash
# Test default scenario
molecule test

# Test HA scenario  
molecule test -s ha
```

## License

GPL-3.0-or-later

## Author

Created by [Adfinis](https://adfinis.com/)
