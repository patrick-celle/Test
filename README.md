# Ansible Playbook for Patroni-Managed PostgreSQL Cluster on Debian

This playbook automates the setup of a highly available PostgreSQL cluster using Patroni and a Distributed Consensus Store (DCS) like etcd on Debian-based systems.

## Features

- Installs and configures a 3-node etcd cluster (or other DCS).
- Installs and configures a 3-node PostgreSQL cluster managed by Patroni.
- Supports a dedicated replication network for PostgreSQL.
- Basic firewall configuration for necessary ports.

## Prerequisites

1.  **Ansible:** Ensure Ansible is installed on your control machine.
2.  **SSH Access:** Passwordless SSH access (using SSH keys) from the control machine to all target servers for the `ansible_user` specified in the inventory.
3.  **Sudo Privileges:** The `ansible_user` must have passwordless sudo privileges on all target servers.
4.  **Operating System:** Target servers should be Debian-based (e.g., Debian, Ubuntu).
5.  **Network Configuration:** Ensure main and replication network interfaces are configured on the PostgreSQL nodes as per your environment.

## Setup

1.  **Clone the Repository:**
    ```bash
    git clone <repository_url>
    cd <repository_directory>
    ```

2.  **Configure Inventory (`inventory.ini`):**

    Update the `inventory.ini` file with the actual IP addresses and hostnames of your servers. Pay special attention to:
    - `ansible_host` for each DCS and PostgreSQL node (main management IP).
    - `pg_replication_ip` for each PostgreSQL node (dedicated replication IP).
    - `ansible_user`: The SSH user for Ansible to connect with.
    - `etcd_...` variables if you need to customize etcd settings.
    - `patroni_...` variables for user/password settings. **It is highly recommended to change default passwords and use Ansible Vault for sensitive data.**
    - `pg_version`: Ensure this matches the PostgreSQL version you intend to install.

    **Example Snippet for `inventory.ini`:**
    ```ini
    [dcs_nodes]
    dcs1 ansible_host=192.168.1.11
    dcs2 ansible_host=192.168.1.12
    dcs3 ansible_host=192.168.1.13

    [postgresql_nodes]
    pg1 ansible_host=192.168.1.21 pg_replication_ip=10.0.0.21
    pg2 ansible_host=192.168.1.22 pg_replication_ip=10.0.0.22
    pg3 ansible_host=192.168.1.23 pg_replication_ip=10.0.0.23

    [all:vars]
    ansible_user=your_ssh_user
    # ansible_become_pass=your_sudo_password # Uncomment if not using passwordless sudo

    pg_version=17
    # ... other variables ...
    ```

3.  **Review Role Variables:**
    Check default variables in role files (e.g., `roles/patroni/templates/patroni.yml.j2`) and inventory variables. Adjust them as per your requirements. For production, use Ansible Vault to encrypt sensitive variables like passwords.

## Running the Playbook

Execute the main playbook:

```bash
ansible-playbook -i inventory.ini deploy_patroni_cluster.yml
```

You might consider using `serial: 1` in `deploy_patroni_cluster.yml` for the `dcs` and `patroni` plays during the first run to ensure nodes come up in a controlled manner, especially for etcd cluster formation and Patroni bootstrap.

## Verifying the Cluster

1.  **Check etcd Cluster Health (on a DCS node):**
    ```bash
    etcdctl endpoint health --cluster
    # Or for etcd v2: etcdctl cluster-health
    ```

2.  **Check Patroni Cluster Status (on a PostgreSQL node):**
    ```bash
    sudo patronictl list
    # Or run as postgres user: patronictl list
    ```
    This command will show the status of all PostgreSQL nodes, their roles (Leader, Replica), and current LSN.

3.  **Connect to PostgreSQL:**
    Connect to the leader node using `psql` or your preferred PostgreSQL client. The leader's IP will be shown by `patronictl list`.

    ```bash
    psql -h <leader_ip> -U {{ patroni_admin_user }} -d postgres
    ```

## Important Notes

- **Idempotency:** The playbook is designed to be idempotent. Running it multiple times should not cause issues.
- **Firewall:** The playbook includes basic `ufw` rules. If you use a different firewall, you'll need to adjust the firewall tasks in `roles/dcs/tasks/main.yml` and `roles/patroni/tasks/main.yml`.
- **PostgreSQL Version:** The playbook is configured for PostgreSQL 17. To use a different version, update the `pg_version` variable in `inventory.ini` and ensure the PostgreSQL APT repository supports it.
- **Security:**
    - **Change default passwords immediately.**
    - Use Ansible Vault for all sensitive data.
    - Review `pg_hba.conf` settings for your specific security requirements.
    - Consider enabling SSL/TLS for Patroni API, etcd, and PostgreSQL connections.
- **Backup and Recovery:** This playbook sets up replication but does not configure backups (e.g., `pg_basebackup`, `pgBackRest`). Ensure you have a separate backup strategy.

## Troubleshooting

- Check Ansible output for errors.
- Review Patroni logs on PostgreSQL nodes (e.g., `/var/log/patroni.log` or via journalctl).
- Review etcd logs on DCS nodes.
- Review PostgreSQL logs on PostgreSQL nodes.
