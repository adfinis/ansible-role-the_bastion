# Ansible Role: The Bastion

Ansible role to install and configure [The Bastion](https://github.com/ovh/the-bastion) on Debian.

This role supports both new installations and upgrades of The Bastion, following the official installation and upgrade documentation.

## Supported Platforms

- Debian 12 and 13

### Requirements

- `python3-pexpect` 

## Role Variables

### Automatic Installation Mode

The role automatically determines the installation mode based on version comparison:

- **New Installation**: When no bastion installation is found
- **Upgrade**: When the target version is higher than the current installed version
- **Skip**: When the target version equals the current installed version  
- **Error**: When the target version is lower than current (downgrade not supported)

Version detection works by checking the `/etc/bastion/version` file created during installation, making it compatible with both git and tarball installation methods.

### Basic Configuration

```yaml
# Installation directory for The Bastion
bastion_install_dir: /opt/bastion

# The Bastion version to install
bastion_version: "v3.21.00"

# Installation method: 'tarball' or 'git'
bastion_install_method: tarball

# Whether to install syslog-ng (recommended)
bastion_install_syslog_ng: true
```

### First Admin Account (New Installations Only)

```yaml
# First admin account configuration (required for new installations)
bastion_first_admin:
  username: "admin"
  ssh_public_key: "ssh-rsa AAAAB3NzaC1yc2E... your-key-here"
```

### Home Directory Encryption (New Installations Only)

```yaml
# Whether to encrypt /home directory (new installations only)
bastion_encrypt_home: false

# Encryption passphrase (should be vaulted in production!)
bastion_encryption_passphrase: "your-secure-passphrase"
```

### Upgrade Settings

```yaml
# Create backup before upgrade
bastion_upgrade_backup_before: true

# Use --managed-upgrade instead of --upgrade (for infrastructure automation)
bastion_upgrade_use_managed: false
```

### Additional Configuration

```yaml
# Custom bastion configuration options
bastion_config:
  defaultLogin: "admin"
  accountCreateSupported: true

# Optional tools installation
bastion_install_yubico_piv_checker: false
bastion_install_mkhash_helper: false
```

### Configuration Format

The `bastion_config` variable accepts key-value pairs that will be written to `/etc/bastion/bastion.conf` in JSON format. Each configuration option will be rendered as:

```json
"configKey": value,
```

Where:
- Boolean values are rendered as `true` or `false`
- String values are rendered with quotes: `"stringValue"`
- Numeric values are rendered without quotes: `123`

## Example Playbooks

### Installation (Automatic Mode Detection)

```yaml
---
- hosts: bastion_servers
  become: true
  vars:
    bastion_version: "v3.21.00"  # Role automatically determines if new/upgrade needed
    bastion_first_admin:
      username: "bastionadmin"
      ssh_public_key: "ssh-rsa AAAAB3NzaC1yc2E... admin@example.com"
    bastion_install_syslog_ng: true
    bastion_encrypt_home: false
  roles:
    - ansible-role-the_bastion
```

### Upgrade Example

```yaml
---
- hosts: bastion_servers
  become: true
  vars:
    bastion_version: "v3.22.00"  # Higher version triggers automatic upgrade
    bastion_upgrade_backup_before: true
    bastion_upgrade_use_managed: false
  roles:
    - ansible-role-the_bastion
```

## Installation Process

The role automatically determines the appropriate installation steps based on version comparison:

### New Installation

When no bastion installation is detected, the role:

1. Validates OS compatibility (Debian 12 and 13)
2. Updates system packages
3. Downloads The Bastion source code
4. Installs required packages and dependencies
5. Optionally encrypts /home directory
6. Runs installation script with `--new-install`
7. Creates first admin account
8. Configures SSH hardening

### Upgrade Process

When a newer version is detected, the role:

1. Detects current version from `/etc/bastion/version` file
2. Compares with target version
3. Creates backup of current installation (optional)
4. Downloads new version source code
5. Updates packages as needed
6. Runs installation script with `--upgrade` or `--managed-upgrade`
7. Applies custom configuration

### Version Comparison

- **Same Version**: Installation is skipped (idempotent)
- **Newer Version**: Automatic upgrade is performed
- **Older Version**: Role fails with error (downgrades not supported)

## Security Considerations

- The role is designed for dedicated bastion hosts only
- SSH configuration is hardened during installation
- Home directory encryption is available but optional
- Admin SSH keys should be managed securely
- Consider using Ansible Vault for sensitive variables

## Testing

The role includes Molecule tests that verify:

- New installation functionality
- Upgrade from older version
- Service availability
- Configuration validation
- Admin account creation

Run tests with:

```bash
molecule test
```

## Role Variables

### Required Variables

```yaml
bastion_first_admin:
  username: "admin_username"
  ssh_public_key: "ssh-rsa AAAAB3NzaC1yc2E... user@host"
```

### Optional Variables

```yaml
# Installation settings
bastion_install_dir: /opt/bastion
bastion_version: "v3.21.00"
bastion_install_method: tarball  # or 'git'

# Feature toggles
bastion_encrypt_home: false
bastion_install_syslog_ng: true
bastion_install_dev_packages: false
bastion_new_install: true

# Security settings
bastion_harden_ssh: true
bastion_regenerate_host_keys: true

# Optional tools
bastion_install_yubico_piv_checker: false
bastion_install_mkhash_helper: false

# Custom configuration
bastion_config: {}  # Configuration options for bastion.conf in JSON format
```

### Encryption Variables

```yaml
bastion_encrypt_home: true
bastion_encryption_passphrase: "your-secure-passphrase"  # Should be vaulted!
```

## Dependencies

This role requires the `community.general` collection:

```yaml
collections:
  - community.general
```

## Example Playbook

### Basic Installation

```yaml
- hosts: bastion_servers
  become: true
  vars:
    bastion_first_admin:
      username: "bastionadmin"
      ssh_public_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQ... admin@company.com"
  roles:
    - ansible-role-the_bastion
```

### Advanced Configuration with Encryption

```yaml
- hosts: bastion_servers
  become: true
  vars:
    bastion_first_admin:
      username: "bastionadmin"
      ssh_public_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQ... admin@company.com"
    bastion_encrypt_home: true
    bastion_encryption_passphrase: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      66386439653866323331316663346232633933663839316264663165653239363138373862373765
      ...
    bastion_install_yubico_piv_checker: true
    bastion_install_mkhash_helper: true
    bastion_config:
      adminAccounts: "bastionadmin,otheradmin"
      enabledGlobalCodes: "ssh"
      defaultAccountTTL: "90"
  roles:
    - ansible-role-the_bastion
```

## Installation Process

The role follows the official installation guide and performs these steps:

1. **System Preparation**: Updates system packages and creates required directories
2. **Source Download**: Downloads The Bastion source code (tarball or git)
3. **Package Installation**: Installs required system packages and dependencies
4. **Home Encryption**: Optionally encrypts the /home directory using LUKS
5. **Bastion Configuration**: Runs the official installation script and applies configuration
6. **Admin Account**: Creates the first administrative account with SSH key

## Security Considerations

- **SSH Hardening**: The role automatically hardens SSH configuration (can be disabled)
- **Home Encryption**: Strongly recommended for production environments
- **Passphrase Security**: Always use Ansible Vault for encryption passphrases
- **System Updates**: The role updates all packages as part of the security baseline

## Testing

This role includes Molecule tests using Podman:

```bash
# Install test dependencies
pip install molecule molecule-plugins[podman] ansible

# Run tests
molecule test
```

## License

GPL-3.0-or-later

## Author Information

This role was created by [Adfinis](https://adfinis.com/).

## Contributing

1. Fork the repository
2. Create a feature branch
3. Add tests for new functionality
4. Ensure all tests pass
5. Submit a pull request
