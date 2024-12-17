```markdown
# Patroni PostgreSQL HA

![License](https://img.shields.io/github/license/vexsx/patroni-postgresql-ha)
![Stars](https://img.shields.io/github/stars/vexsx/patroni-postgresql-ha)
![Forks](https://img.shields.io/github/forks/vexsx/patroni-postgresql-ha)
![Issues](https://img.shields.io/github/issues/vexsx/patroni-postgresql-ha)
![CI](https://github.com/vexsx/patroni-postgresql-ha/actions/workflows/ci.yml/badge.svg)

## Table of Contents

- [Introduction](#introduction)
- [Features](#features)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Deployment](#deployment)
  - [Using Docker Compose](#using-docker-compose)
  - [Kubernetes Deployment](#kubernetes-deployment)
- [Monitoring](#monitoring)
- [Backup and Restore](#backup-and-restore)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)
- [Contact](#contact)

## Introduction

Patroni PostgreSQL HA is a high-availability solution for PostgreSQL databases, leveraging [Patroni](https://patroni.readthedocs.io/en/latest/) to manage failover and ensure seamless replication. This setup provides automated failover, scaling, and maintenance, making your PostgreSQL deployment resilient and highly available.

## Features

- **Automated Failover:** Automatically promotes a standby to master in case of primary failure.
- **Dynamic Configuration:** Easily adjust configurations without downtime.
- **Scalable Replication:** Supports multiple replicas for load balancing and redundancy.
- **Integration with DCS:** Uses etcd, Consul, or ZooKeeper for distributed configuration storage.
- **Monitoring and Alerts:** Integrated monitoring to keep track of cluster health.
- **Backup Solutions:** Simplified backup and restore processes.

## Architecture

![Architecture Diagram](docs/architecture.png)

The architecture consists of:

- **Patroni Cluster:** Manages PostgreSQL instances and orchestrates failover.
- **Distributed Configuration Store (DCS):** Stores cluster state using etcd, Consul, or ZooKeeper.
- **Load Balancer:** Distributes read traffic among replicas and directs write traffic to the primary.
- **Monitoring Tools:** Tracks the health and performance of the PostgreSQL cluster.

## Prerequisites

- **Operating System:** Ubuntu 20.04 or later / CentOS 7 or later
- **PostgreSQL:** Version 12 or higher
- **Python:** Version 3.6 or higher
- **DCS:** etcd, Consul, or ZooKeeper
- **Docker & Docker Compose:** For containerized deployments (optional)
- **Kubernetes Cluster:** For Kubernetes deployments (optional)

## Installation

### Using Ansible

Ansible playbooks are provided for easy deployment.

1. **Clone the Repository:**

   ```bash
   git clone https://github.com/vexsx/patroni-postgresql-ha.git
   cd patroni-postgresql-ha
   ```

2. **Install Dependencies:**

   Ensure you have Ansible installed:

   ```bash
   sudo apt-get update
   sudo apt-get install -y ansible
   ```

3. **Configure Inventory:**

   Edit the `inventory.ini` file to match your environment.

4. **Run the Playbook:**

   ```bash
   ansible-playbook -i inventory.ini setup.yml
   ```

### Manual Installation

1. **Install PostgreSQL:**

   Follow the official [PostgreSQL installation guide](https://www.postgresql.org/download/).

2. **Install Patroni:**

   ```bash
   sudo apt-get install -y python3-pip
   pip3 install patroni[etcd]
   ```

3. **Configure Patroni:**

   Create a `patroni.yml` configuration file. Refer to the [Patroni documentation](https://patroni.readthedocs.io/en/latest/configuration.html) for detailed settings.

4. **Start Patroni:**

   ```bash
   patroni patroni.yml
   ```

## Configuration

Customize the `patroni.yml` file to suit your environment. Key configuration sections include:

- **DCS Configuration:** Specify your chosen Distributed Configuration Store.
- **PostgreSQL Settings:** Define data directories, replication settings, and authentication.
- **Failover Policies:** Set parameters for automated failover behavior.
- **Logging:** Configure log levels and output destinations.

Example configuration snippet:

```yaml
scope: postgres
namespace: /service/
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.1.1:8008

etcd:
  host: 127.0.0.1:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576 # 1 MB
  initdb:
    - encoding: UTF8
    - data-checksums
  pg_hba:
    - host replication replicator 127.0.0.1/32 md5
    - host all all 0.0.0.0/0 md5
  users:
    admin:
      password: adminpassword
      options:
        - createrole
        - createdb

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.1.1:5432
  data_dir: /var/lib/postgresql/data
  bin_dir: /usr/lib/postgresql/12/bin
  authentication:
    replication:
      username: replicator
      password: replicatorpassword
    superuser:
      username: postgres
      password: postgrespassword
  parameters:
    max_connections: 100
    shared_buffers: 256MB
    wal_level: replica
    hot_standby: "on"
```

## Deployment

### Using Docker Compose

Deploy the Patroni PostgreSQL HA cluster using Docker Compose for a containerized setup.

1. **Clone the Repository:**

   ```bash
   git clone https://github.com/vexsx/patroni-postgresql-ha.git
   cd patroni-postgresql-ha/docker
   ```

2. **Configure Environment Variables:**

   Edit the `.env` file to set your desired configurations.

3. **Start the Cluster:**

   ```bash
   docker-compose up -d
   ```

4. **Verify the Deployment:**

   ```bash
   docker-compose ps
   ```

### Kubernetes Deployment

Deploying Patroni PostgreSQL HA on Kubernetes ensures scalability and resilience.

1. **Clone the Repository:**

   ```bash
   git clone https://github.com/vexsx/patroni-postgresql-ha.git
   cd patroni-postgresql-ha/kubernetes
   ```

2. **Apply Kubernetes Manifests:**

   ```bash
   kubectl apply -f namespace.yaml
   kubectl apply -f configmap.yaml
   kubectl apply -f statefulset.yaml
   kubectl apply -f service.yaml
   ```

3. **Verify the Deployment:**

   ```bash
   kubectl get pods -n patroni
   ```

## Monitoring

Integrate monitoring tools to keep track of your PostgreSQL cluster's health and performance.

- **Prometheus:** Collect metrics from Patroni and PostgreSQL.
- **Grafana:** Visualize metrics with pre-built dashboards.
- **Alertmanager:** Set up alerts for critical events.

## Backup and Restore

Ensure data safety with regular backups and a streamlined restore process.

- **pgBackRest:** Use pgBackRest for reliable backups.
- **Barman:** Alternative backup solution for PostgreSQL.
- **Restore Process:** Detailed steps to restore from backups are available in the [Backup Documentation](docs/backup.md).

## Troubleshooting

Common issues and their resolutions:

- **Failover Not Triggering:**
  - Check DCS connectivity.
  - Ensure Patroni is running on all nodes.
- **Replication Lag:**
  - Monitor network latency.
  - Adjust PostgreSQL replication settings.
- **Authentication Errors:**
  - Verify credentials in `pg_hba.conf` and Patroni configuration.

For more detailed troubleshooting steps, refer to the [Troubleshooting Guide](docs/troubleshooting.md).

## Contributing

Contributions are welcome! Please follow these steps to contribute:

1. **Fork the Repository**
2. **Create a Feature Branch:**

   ```bash
   git checkout -b feature/YourFeature
   ```

3. **Commit Your Changes:**

   ```bash
   git commit -m "Add Your Feature"
   ```

4. **Push to the Branch:**

   ```bash
   git push origin feature/YourFeature
   ```

5. **Open a Pull Request**

Please ensure your code adheres to the project's coding standards and includes relevant tests.

## License

This project is licensed under the [MIT License](LICENSE).

## Contact

For questions, issues, or support, please open an [issue](https://github.com/vexsx/patroni-postgresql-ha/issues) on GitHub or contact the maintainer at [p.gheiratian1382@yahoo.com](mailto:p.gheiratian1382@yahoo.com).

