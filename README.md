# Ansible role - Promtail

[![License](https://img.shields.io/github/license/voidquark/promtail)](LICENSE)

The Ansible Promtail Role allows you to effortlessly deploy and manage Promtail, agent which ships contents of local logs to private Loki.
This role is tailored for operating systems such as **RedHat**, **Rocky Linux**, **AlmaLinux**, **Ubuntu**, and **Debian**.

**🔑 Key Features**
- **🛡️ Root-less Log Collection**: Promtail operates without requiring root privileges. ACL (Access Control List) ensures that Promtail can access logs securely without needing root permissions.
- **🧹 Effortless Uninstall**: Easily remove Promtail from your system using the "promtail_uninstall" tag.

📢 **[Check the blog post](https://voidquark.com/blog/rootless-promtail-with-ansible/)** 📝 **Understand the rationale behind constructing this role in a specific manner.**

## Table of Content

- [Requirements](#requirements)
- [Role Variables](#role-variables)
- [Playbook](#playbook)

## Requirements

- Ansible 2.10+

## Role Variables

```yaml
promtail_version: "latest"
```
The version of Promtail to download and deploy. Supported standard version "3.0.0" format or "latest".

```yaml
promtail_http_listen_port: 9080
```
The TCP port on which Promtail listens. By default, it listens on port `9080`.

```yaml
promtail_http_listen_address: "0.0.0.0"
```
The address on which Promtail listens for HTTP requests. By default, it listens on all interfaces.

```yaml
promtail_expose_port: false
```
By default, this is set to `false`. It supports only simple `firewalld` configurations. If set to `true`, a firewalld rule is added to expose the TCP `promtail_http_listen_port`. If set to `false`, the system ensures that the rule is not present. If the `firewalld.service` is not active, all firewalld tasks are skipped.

```yaml
promtail_positions_path: "/var/lib/promtail"
```
Promtail path for position file. File indicating how far it has read into a file. It is needed for when Promtail is restarted to allow it to continue from where it left off.

```yaml
promtail_user_append_groups:
  - "systemd-journal"
```
Appends the promtail user to specific groups. By default, it appends the user to the `systemd-journal` group, granting permission to read system journal logs.

```yaml
promtail_download_url_rpm: "https://github.com/grafana/loki/releases/download/v{{ promtail_version }}/promtail-{{ promtail_version }}.{{ __promtail_arch }}.rpm"
```
The default download URL for the Promtail rpm package from GitHub.

```yaml
promtail_download_url_deb: "https://github.com/grafana/loki/releases/download/v{{ promtail_version }}/promtail_{{ promtail_version }}_{{ __promtail_arch }}.deb"
```
The default download URL for the Promtail deb package from GitHub.

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
The `clients` block configures how Promtail connects to instances of Loki. [All possible values for `clients`](https://grafana.com/docs/loki/latest/clients/promtail/configuration/#clients). ⚠️ This configuration is mandatory. By default, it's empty, and the example above serves as a simple illustration for inspiration.

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
          - localhost
        labels:
          job: messages
          instance: "{{ ansible_fqdn }}"
          __path__: /var/log/messages
```
The `scrape_configs` block configures how Promtail can scrape logs from a series of targets using a specified discovery method. [All possible values for `scrape_configs`](https://grafana.com/docs/loki/latest/clients/promtail/configuration/#scrape_configs). ⚠️ This configuration is mandatory. By default, it's empty, and the example above serves as a simple illustration for inspiration.

| Variable Name | Description
| ----------- | ----------- |
| `promtail_limits_config` | The optional limits_config block configures global limits for this instance of Promtail. 📚 [documentation](https://grafana.com/docs/loki/latest/clients/promtail/configuration/#limits_config).
| `promtail_target_config` | The target_config block controls the behavior of reading files from discovered targets. 📚 [documentation](https://grafana.com/docs/loki/latest/clients/promtail/configuration/#target_config).

## Dependencies

No Dependencies

## Playbook

```yaml
- name: Manage promtail service
  hosts: all
  become: true
  vars:
    promtail_clients:
      - url: http://localhost:3100/loki/api/v1/push
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
              - localhost
            labels:
              job: messages
              instance: "{{ ansible_fqdn }}"
              __path__: /var/log/messages
  roles:
    - role: voidquark.promtail
```

- Playbook execution example
```shell
# Deploy Promtail
ansible-playbook function_promtail_play.yml

# Uninstall Promtail
ansible-playbook function_promtail_play.yml -t promtail_uninstall
```

## License

MIT

## Contribution

Feel free to customize and enhance the role according to your needs.
Your feedback and contributions are greatly appreciated. Please open an issue or submit a pull request with any improvements.

## Author Information

Created by [VoidQuark](https://voidquark.com)
