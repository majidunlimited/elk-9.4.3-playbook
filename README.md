# ELK Stack 9.4.3 – Ansible Installation Playbook

Automated deployment of **Elasticsearch, Kibana, and Logstash 9.4.3** on a fresh Ubuntu Server 24.04 LTS host using Ansible.

The playbook configures a single-node Elastic Stack installation with TLS certificates, built-in user credentials, Elasticsearch monitoring, Kibana HTTPS, Logstash monitoring, and Nexus Repository Manager integration for APT packages.

> **Scope:** Fresh single-server installation only. This is not an in-place upgrade, cluster deployment, backup, or migration playbook.

## Features

- Installs Elasticsearch, Kibana, and Logstash with pinned package versions.
- Uses Nexus Repository Manager as an APT proxy and hosted repository.
- Generates a local Elastic Stack Certificate Authority and service certificates.
- Enables Elasticsearch authentication and TLS for HTTP and transport connections.
- Stores sensitive deployment inputs in Ansible Vault.
- Configures Kibana HTTPS and encrypted saved objects.
- Configures Logstash monitoring and its HTTP API.
- Supports a private Fleet integration registry for air-gapped environments.
- Checks the installed Elasticsearch version and HTTPS readiness.
- Uses initialization markers to avoid repeating initial password and certificate setup.

## Components

| Component | Version |
|---|---:|
| Ubuntu Server | 24.04 LTS |
| Elasticsearch | 9.4.3 |
| Kibana | 9.4.3 |
| Logstash | 9.4.3 |
| Repository manager | Nexus Repository Manager |
| Automation | Ansible |

## Repository layout

```text
elk-ansible/
├── elk-play-9.4.3.yml
├── inventory.example.ini
├── vault.example.yml
├── ansible.cfg
├── .gitignore
└── README.md
```

Copy the example files to local working files before deployment. Do not commit real inventory data or Vault secrets.

## Prerequisites

- A fresh Ubuntu Server 24.04 LTS host.
- Ansible installed on the control node.
- SSH connectivity and privilege escalation to the target host.
- Nexus repositories containing the required Ubuntu and Elastic packages.
- The Nexus APT signing key exposed through the configured RAW repository path.
- A private Fleet package registry if using the air-gapped Fleet configuration.
- Sufficient disk, memory, and CPU resources for Elasticsearch.

The playbook currently connects as `root` by default. You can override the SSH user with `ansible_ssh_user` in inventory. The connecting account must have the required privilege escalation permissions.

## Configuration

### 1. Inventory

Copy the sample inventory:

```bash
cp inventory.example.ini inventory.ini
```

Edit `inventory.ini` and set the real target address. `elk_server_ip` must be an IP address because it is included in the generated certificate Subject Alternative Names (SANs).

### 2. Repository settings

The playbook defines example defaults for the Nexus and Fleet endpoints. Override them in your inventory or group variables:

```ini
[ELKServers:vars]
nexus_repository_url=https://nexus.example.com
nexus_apt_proxy=apt-proxy-noble
nexus_apt_hosted=apt-hosted
nexus_hosted_suite=noble
nexus_hosted_components=main
fleet_registry_url=https://fleet-registry.example.com:8081
elk_timezone=Asia/Tehran
```

Adjust repository names and distribution suites to match your Nexus configuration. The hosted repository suite is configurable because repository layouts differ between environments.

The Ubuntu proxy source uses the Ubuntu archive keyring. The Nexus hosted source uses the downloaded Nexus APT signing key. Ensure your repositories are signed accordingly; do not use `trusted=yes` as a workaround for repository-signature errors.

### 3. Ansible Vault

Create a local Vault file:

```bash
cp vault.example.yml vault.yml
ansible-vault encrypt vault.yml
```

Alternatively, create it interactively:

```bash
ansible-vault create vault.yml
```

Set strong, unique values for all six variables:

```yaml
vault_elastic_password: "REPLACE_WITH_A_STRONG_PASSWORD"
vault_kibana_system_password: "REPLACE_WITH_A_STRONG_PASSWORD"
vault_logstash_system_password: "REPLACE_WITH_A_STRONG_PASSWORD"
vault_kibana_encryption_key: "REPLACE_WITH_AT_LEAST_32_RANDOM_CHARACTERS"
vault_kibana_key_pass: "REPLACE_WITH_A_CERTIFICATE_PASSPHRASE"
vault_es_p12_pass: "REPLACE_WITH_A_CERTIFICATE_PASSPHRASE"
```

The Kibana encryption key must contain at least 32 characters. Use random values rather than the example placeholders.

**Never commit `vault.yml`, its password file, or unencrypted secrets.**

## Run the playbook

Copy and customize the example inventory, create your Vault file, then run:

```bash
ansible-playbook \
  -i inventory.ini \
  elk-play-9.4.3.yml \
  -e @vault.yml \
  --ask-vault-pass
```

You can validate YAML and Ansible task syntax before deployment:

```bash
ansible-playbook \
  -i inventory.ini \
  elk-play-9.4.3.yml \
  --syntax-check
```

A syntax check does not validate repository availability, credentials, certificate generation, or runtime service health.

## Deployment workflow

The playbook performs these main operations:

1. Validates that the target is Ubuntu 24.04.
2. Checks that required Vault variables are present.
3. Configures Nexus APT repositories.
4. Installs base dependencies and the requested Elastic packages.
5. Configures `vm.max_map_count`.
6. Creates a local CA and Elasticsearch, Kibana, and Logstash certificates.
7. Configures Elasticsearch TLS, authentication, and its secure keystore.
8. Initializes the built-in `elastic`, `kibana_system`, and `logstash_system` passwords.
9. Verifies Elasticsearch authentication and version.
10. Configures Kibana HTTPS, Fleet registry settings, and saved-object encryption.
11. Configures Logstash monitoring and its API.
12. Starts and enables the services.
13. Waits for the Kibana HTTPS status endpoint.

## Service endpoints

| Service | Endpoint | Port |
|---|---|---:|
| Elasticsearch | `https://ELK_SERVER_IP:9200` | 9200 |
| Kibana | `https://ELK_SERVER_IP:5601` | 5601 |
| Logstash monitoring API | `http://ELK_SERVER_IP:9600` | 9600 |

The default Kibana administrative login username is `elastic`. Its password is the value of `vault_elastic_password`.

The generated CA certificate is located at:

```text
/etc/elastic-stack/certs/ca/elastic-stack-ca.crt
```

Clients must trust this CA when connecting to the HTTPS endpoints.

## Important operational and security notes

### Network exposure

The default configuration binds Elasticsearch, Kibana, and the Logstash API to `0.0.0.0`. Restrict access using host and network firewalls. In particular, do not expose Elasticsearch port 9200 or Logstash API port 9600 to untrusted networks.

The Logstash API is configured over HTTP in this playbook. Keep it on a trusted management network or change the configuration to use an appropriately secured endpoint.

### Secret files on the target

Ansible Vault protects secret values at rest on the control node, but the playbook writes service credentials and the Kibana key passphrase into service configuration files on the target. These files are assigned restrictive permissions, but administrators with sufficient OS privileges can still access them. Protect the host and its backups accordingly.

### CA private key

The generated CA PKCS#12 file has no passphrase in this implementation and is protected with root-only file permissions. Treat it as a highly sensitive private key container. For higher-assurance environments, consider a separately managed CA and a dedicated certificate lifecycle process.

### Certificate and password reuse

The playbook uses file-existence checks and marker files to avoid regenerating existing certificates or resetting initialized built-in passwords.

Changing Vault values does **not** rotate existing passwords or certificates. Rotation requires a separate, coordinated procedure.

If you intentionally need to rebuild credentials or certificates, plan the operation carefully. Do not simply delete marker files on a production server.

### APT repositories

The Ubuntu proxy sources use the Ubuntu archive keyring; the hosted repository uses the configured Nexus signing key. Repository names, suites, and signing arrangements must match your Nexus setup. The playbook checks that the requested package versions are visible through APT before installation.

### Logstash pipelines

This playbook installs and configures Logstash but does not deploy ingestion pipelines. Add pipeline configuration separately under:

```text
/etc/logstash/conf.d/
```

After adding pipelines, validate the configuration and check Logstash service and pipeline health.

### Air-gapped Fleet

Kibana Fleet is configured with `xpack.fleet.isAirGapped: true` and a configurable private registry URL. The private registry must contain integrations compatible with Kibana 9.4.3. This playbook does not populate or synchronize that registry.

## Limitations

- Single-node deployment only.
- Ubuntu Server 24.04 LTS only.
- No in-place upgrades or data migration.
- No cluster or high-availability configuration.
- No backup or restore automation.
- No automatic certificate or password rotation.
- No firewall policy deployment.
- No Logstash ingestion pipeline deployment.
- No automatic population of the private Fleet registry.

## Maintainer

**Majid Mortazavi**

DevOps Engineer | Linux | Automation | CI/CD | Security

- GitHub: [majidunlimited](https://github.com/majidunlimited)
- LinkedIn: [Majid Mortazavi](https://www.linkedin.com/in/majid-mortazavi/)

---

This project is based on a previously tested ELK 9.1.1 installation playbook and updated for Elastic Stack 9.4.3, including monitoring and secret-handling improvements.
