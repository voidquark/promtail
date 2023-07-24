# Ansible role - Promtail

[![License](https://img.shields.io/github/license/voidquark/promtail)](LICENSE)

The Ansible Promtail Role allows you to effortlessly deploy and manage Promtail, agent which ships contents of local logs to private Loki. Role is tailored for systems from the Red Hat family (e.g., RHEL, RockyLinux, AlmaLinux, etc.).

**🔑 Key Features**
- **🛡️ Root-less Log Collection**: Promtail run without requiring root privileges, allowing seamless collection of system logs from locations like `/var/log/secure`, `/var/log/messages`, etc.
- **📦 Out-of-the-box Deployment**: Get Promtail up and running quickly with default configurations that work seamlessly with Red Hat family systems. See [Quick Start](#quick-start) for easy setup.
- **🧩 Flexible Configuration**: Easily customize Promtail configuration to match your specific requirements.
- **🧹 Effortless Uninstall**: Completely remove Promtail from your system with a single command, ensuring a clean uninstallation.

📢 **[Check the blog post](https://voidquark.com/rootless-promtail-with-ansible/)** 📝 **Understand the rationale behind constructing this role in a specific manner.**

## Table of Content

- [Requirements](#requirements)
- [Role Variables](#role-variables)
- [Playbook](#playbook)
- [Quick Start](#quick-start)

## Requirements

- Ansible 2.9+
- `RHEL`/`RockyLinux`/`AlmaLinux` 8+ or compatible distributions

## Role Variables

```yaml
promtail_version: "latest"
```
The version of Promtail to download and deploy. Supported standard version "2.8.3" format or "latest".

```yaml
promtail_arch: "x86_64"
```
The architecture for `RPM` package for which Promtail is being deployed. Possible values `x86_64`, `aarch64`, `arm`.

```yaml
promtail_http_listen_port: 9080
```
The TCP port on which Promtail listens. By default, it listens on port `9080`.

```yaml
promtail_http_listen_address: "0.0.0.0"
```
The address on which Promtail listens for HTTP requests. By default, it listens on all interfaces.

```yaml
promtail_expose_port: true
```
Set to `true` by default, controls whether to add a firewalld rule for exposing the Promtail port. When `true`, a firewalld rule is added to allow inbound traffic on the specified Promtail port. Set to `false` to ensure that firewalld rule is not present. Enabling this option allows you to collect Promtail `/metrics`, which are essential for Prometheus monitoring.

```yaml
promtail_positions_path: "/var/lib/promtail"
```
Promtail path for position file. File indicating how far it has read into a file. It is needed for when Promtail is restarted to allow it to continue from where it left off.

```yaml
promtail_systemd_journal_access: true
```
Enables adding the `promtail` user to the `systemd-journal` group, granting permission to read system journal logs. Required for configuring `promtail_scrape_config` for journal log scraping.

```yaml
promtail_download_url: "https://github.com/grafana/loki/releases/download/v{{ promtail_version }}/promtail-{{ promtail_version }}.{{ promtail_arch }}.rpm"
```
The default download URL for the Promtail rpm package from GitHub.

```yaml
promtail_server:
  http_listen_port: "{{ promtail_http_listen_port }}"
  http_listen_address: "{{ promtail_http_listen_address }}"
```
The `server` block configures Promtail behavior as an HTTP server. [All possible values for `server`](https://grafana.com/docs/loki/latest/clients/promtail/configuration/#server)

```yaml
promtail_positions:
  filename: "{{ promtail_positions_path }}/positions.yaml"
```
The `positions` block configures where Promtail will save a file indicating how far it has read into a file. It is needed for when Promtail is restarted to allow it to continue from where it left off. [All possible values for `positions`](https://grafana.com/docs/loki/latest/clients/promtail/configuration/#positions)

```yaml
promtail_clients:
  - url: http://localhost:3100/loki/api/v1/push
```
The `clients` block configures how Promtail connects to instances of Loki. [All possible values for `clients`](https://grafana.com/docs/loki/latest/clients/promtail/configuration/#clients). ⚠️ Please remember to update the url field to match your Loki instance's address. Failure to do so will result in Promtail being unable to send logs to the correct destination.

```yaml
promtail_scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: secure
          instance: "{{ ansible_fqdn }}"
          __path__: /var/log/secure
      - targets:
          - localhoost
        labels:
          job: messages
          instance: "{{ ansible_fqdn }}"
          __path__: /var/log/messages
      - targets:
          - localhoost
        labels:
          job: cron
          instance: "{{ ansible_fqdn }}"
          __path__: /var/log/cron
      - targets:
          - localhoost
        labels:
          job: dnf
          instance: "{{ ansible_fqdn }}"
          __path__: /var/log/dnf.log
      - targets:
          - localhoost
        labels:
          job: boot
          instance: "{{ ansible_fqdn }}"
          __path__: /var/log/boot.log
```
The `scrape_configs` block configures how Promtail can scrape logs from a series of targets using a specified discovery method. [All possible values for `scrape_configs`](https://grafana.com/docs/loki/latest/clients/promtail/configuration/#scrape_configs). By default, the configuration uses `static_configs` and includes preconfigured settings for common system logs. The `instance` label is used to identify the system based on the `ansible_fqdn` magic variable, which gathers the fully qualified domain name of the system.

```yaml
promtail_acl_log_file_permission:
  - "/var/log/secure"
  - "/var/log/messages"
  - "/var/log/cron"
  - "/var/log/dnf.log"
  - "/var/log/boot.log"
```
The `promtail_acl_log_file_permission` variable allows applying ACL read permissions for the `promtail` user to the specified log files. Additionally, it creates a logrotate configuration file (`/etc/logrotate.d/promtail_acl`) to maintain the read permission for promtail after log rotation. This approach ensures that Promtail can read system logs without requiring root privileges.
To work around logrotate limitations, a dummy empty log directory (`/var/log/dummy_promtail_acl`) is created. This avoids modifying system logrotate configurations like `/etc/logrotate.d/rsyslog`, providing a flexible way to manage ACL permissions for both system and non-system logs (e.g., nginx logs).


```yaml
promtail_acl_log_dir_permission:
  - "/var/log"
```
The `promtail_acl_log_dir_permission` variable ensures that default and recursive read and execute permissions are applied to the specified directory and all its subdirectories. This grants the `promtail` user access to the directory and its subdirectories, without requiring root privileges. The configuration is permanent and managed in `/etc/logrotate.d/promtail_acl`. Please note that this ACL permission is only applied to directories and their subdirectories, not to individual log files.

| Variable Name | Description
| ----------- | ----------- |
| `promtail_limits_config` | The optional limits_config block configures global limits for this instance of Promtail. 📚 [documentation](https://grafana.com/docs/loki/latest/clients/promtail/configuration/#limits_config).
| `promtail_target_config` | The target_config block controls the behavior of reading files from discovered targets. 📚 [documentation](https://grafana.com/docs/loki/latest/clients/promtail/configuration/#target_config).

## Dependencies

No Dependencies

## Playbook

- playbook by default requires the `ansible_fqdn` magic variable, reducing the need to collect all facts. This improves performance and ensures efficient execution of the playbook.
```yaml
- name: Manage promtail service
  hosts: promtail
  gather_subset:
    - "network"
    - "!all"
    - "!min"
  become: true
  roles:
    - role: voidquark.promtail
```

## Quick Start

To quickly deploy Promtail using this Ansible role, follow these steps:

**1. Set up your project directory structure:**
```shell
ansible_structure
├── playbook
│   └── function_promtail_play.yml # Playbook
└── inventory
    ├── group_vars
    │   └── promtail
    │       └── promtail_vars.yml # Overwrite variables in group_vars (optional)
    ├── hosts
    └── host_vars
        └── promtail-demo.voidquark.com
            └── host_vars.yml # Overwrite variables in host_vars (optional)
```
**2. Install the Ansible Promtail Role from Ansible Galaxy:**
```shell
ansible-galaxy install voidquark.promtail
```
**3. Create your inventory - `inventory/hosts`**
```shell
[promtail]
promtail-demo.voidquark.com
```
**4. Create your playbook - `playbook/function_promtail_play.yml`**
```yaml
- name: Manage promtail service
  hosts: promtail
  gather_subset:
    - "network"
    - "!all"
    - "!min"
  become: true
  roles:
    - role: voidquark.promtail
```
**5. Execute the playbook**
```shell
# Deployment
ansible-playbook -i inventory/hosts playbook/function_promtail_play.yml

# Uninstall
ansible-playbook -i inventory/hosts playbook/function_promtail_play.yml -t promtail_uninstall
```

## License

MIT

## Contribution

Feel free to customize and enhance the role according to your needs.
Your feedback and contributions are greatly appreciated. Please open an issue or submit a pull request with any improvements.

## Author Information

Created by [VoidQuark](https://voidquark.com)
