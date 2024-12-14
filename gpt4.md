Setting up a highly available PostgreSQL 14 cluster using Patroni, etcd, and HAProxy on Ubuntu 22.04 involves several steps. This guide will walk you through the entire process, ensuring you have a robust active/passive PostgreSQL setup across three nodes with HAProxy managing the load.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Network Configuration](#network-configuration)
3. [Installing Dependencies](#installing-dependencies)
4. [Installing and Configuring etcd](#installing-and-configuring-etcd)
5. [Installing PostgreSQL 14](#installing-postgresql-14)
6. [Installing and Configuring Patroni](#installing-and-configuring-patroni)
7. [Setting Up HAProxy](#setting-up-haproxy)
8. [Initializing the Cluster](#initializing-the-cluster)
9. [Verifying the Setup](#verifying-the-setup)
10. [Maintenance and Monitoring](#maintenance-and-monitoring)

---

## Prerequisites

- **Operating System:** Ubuntu 22.04 LTS on all nodes.
- **Nodes:**
  - **PostgreSQL Nodes:** 3 servers (e.g., `pg1`, `pg2`, `pg3`)
  - **HAProxy Node:** 1 server (e.g., `haproxy`)
- **Network:** Ensure all nodes can communicate with each other over the network.
- **Users:** Root or a user with `sudo` privileges on all nodes.
- **Firewall:** Open necessary ports between nodes.

---

## Network Configuration

Ensure that all nodes can communicate with each other over the network. It's recommended to use private networking for inter-node communication to enhance security and performance.

- **PostgreSQL Nodes:**
  - Hostnames/IPs: `pg1`, `pg2`, `pg3`
- **HAProxy Node:**
  - Hostname/IP: `haproxy`

### Update `/etc/hosts` (Optional)

For easier management, you can map hostnames to IP addresses in the `/etc/hosts` file on each node:

```bash
sudo nano /etc/hosts
```

Add entries like:

```
192.168.1.101 pg1
192.168.1.102 pg2
192.168.1.103 pg3
192.168.1.104 haproxy
```

Save and exit.

---

## Installing Dependencies

### Update and Upgrade

On all nodes, update the package lists and upgrade existing packages:

```bash
sudo apt update && sudo apt upgrade -y
```

### Install Common Dependencies

On **all nodes** (PostgreSQL nodes and HAProxy node), install necessary packages:

```bash
sudo apt install -y curl wget vim git python3 python3-pip python3-venv
```

### Install PostgreSQL Packages

On **PostgreSQL nodes** (`pg1`, `pg2`, `pg3`):

1. **Add PostgreSQL APT Repository:**

   ```bash
   wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
   sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ jammy-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
   sudo apt update
   ```

2. **Install PostgreSQL 14:**

   ```bash
   sudo apt install -y postgresql-14 postgresql-client-14
   ```

---

## Installing and Configuring etcd

etcd serves as the distributed key-value store for Patroni to store the cluster state.

### On All PostgreSQL Nodes (`pg1`, `pg2`, `pg3`):

1. **Download etcd Binary:**

   ```bash
   ETCD_VERSION=v3.5.12
   wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz
   tar xzvf etcd-${ETCD_VERSION}-linux-amd64.tar.gz
   sudo mv etcd-${ETCD_VERSION}-linux-amd64/etcd* /usr/local/bin/
   rm -rf etcd-${ETCD_VERSION}-linux-amd64 etcd-${ETCD_VERSION}-linux-amd64.tar.gz
   ```

2. **Create etcd User and Directories:**

   ```bash
   sudo useradd -r -m -U -d /var/lib/etcd -s /bin/false etcd
   sudo mkdir -p /etc/etcd /var/lib/etcd
   sudo chown etcd:etcd /var/lib/etcd
   ```

3. **Configure etcd:**

   Create the etcd systemd service file:

   ```bash
   sudo nano /etc/systemd/system/etcd.service
   ```

   Add the following content, replacing `pg1`, `pg2`, `pg3` with actual hostnames or IPs:

   ```ini
   [Unit]
   Description=etcd key-value store
   Documentation=https://github.com/etcd-io
   After=network.target

   [Service]
   Type=notify
   User=etcd
   ExecStart=/usr/local/bin/etcd \
     --name ${HOSTNAME} \
     --data-dir /var/lib/etcd \
     --initial-advertise-peer-urls http://$(hostname -I | awk '{print $1}'):2380 \
     --listen-peer-urls http://0.0.0.0:2380 \
     --advertise-client-urls http://$(hostname -I | awk '{print $1}'):2379 \
     --listen-client-urls http://0.0.0.0:2379 \
     --initial-cluster pg1=http://pg1:2380,pg2=http://pg2:2380,pg3=http://pg3:2380 \
     --initial-cluster-state new
   Restart=always
   RestartSec=10s

   [Install]
   WantedBy=multi-user.target
   ```

   **Note:** Ensure that all PostgreSQL nodes (`pg1`, `pg2`, `pg3`) have consistent `--initial-cluster` settings.

4. **Reload Systemd and Start etcd:**

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable etcd
   sudo systemctl start etcd
   sudo systemctl status etcd
   ```

5. **Verify etcd Cluster:**

   On any PostgreSQL node, run:

   ```bash
   etcdctl member list
   ```

   Ensure all members (`pg1`, `pg2`, `pg3`) are listed and healthy.

   **Note:** You may need to set `ETCDCTL_API=3` and provide endpoints if necessary.

---

## Installing PostgreSQL 14

Assuming PostgreSQL 14 is already installed on all PostgreSQL nodes from the previous steps.

### Configure PostgreSQL for Replication

1. **Create `replicator` User:**

   On the **primary node** (will be managed by Patroni):

   ```bash
   sudo -i -u postgres
   psql
   ```

   In the PostgreSQL shell:

   ```sql
   CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replicator_password';
   \q
   ```

   Exit the PostgreSQL user shell:

   ```bash
   exit
   ```

2. **Configure `pg_hba.conf`:**

   On **all PostgreSQL nodes**, edit `pg_hba.conf`:

   ```bash
   sudo nano /etc/postgresql/14/main/pg_hba.conf
   ```

   Add the following lines to allow replication:

   ```
   host    replication     replicator      0.0.0.0/0       md5
   host    all             all             0.0.0.0/0       md5
   ```

   **Note:** For better security, replace `0.0.0.0/0` with specific IP ranges.

3. **Configure `postgresql.conf`:**

   On **all PostgreSQL nodes**, edit `postgresql.conf`:

   ```bash
   sudo nano /etc/postgresql/14/main/postgresql.conf
   ```

   Ensure the following settings:

   ```conf
   listen_addresses = '*'
   wal_level = replica
   max_wal_senders = 10
   hot_standby = on
   ```

4. **Restart PostgreSQL:**

   On **all PostgreSQL nodes**:

   ```bash
   sudo systemctl restart postgresql
   ```

---

## Installing and Configuring Patroni

Patroni will manage PostgreSQL high availability using etcd as the DCS (Distributed Configuration Store).

### On All PostgreSQL Nodes (`pg1`, `pg2`, `pg3`):

1. **Install Patroni:**

   It's recommended to use a Python virtual environment.

   ```bash
   sudo useradd -r -m -U -d /opt/patroni -s /bin/bash patroni
   sudo mkdir -p /opt/patroni
   sudo chown patroni:patroni /opt/patroni

   sudo -i -u patroni
   cd /opt/patroni
   python3 -m venv venv
   source venv/bin/activate
   pip install --upgrade pip
   pip install patroni[etcd]
   deactivate
   exit
   ```

2. **Create Patroni Configuration File:**

   ```bash
   sudo nano /etc/patroni.yml
   ```

   Sample configuration:

   ```yaml
   scope: postgres
   namespace: /db/
   name: pg1  # Change to pg2, pg3 on respective nodes

   etcd:
     host: localhost:2379

   bootstrap:
     dcs:
       ttl: 30
       loop_wait: 10
       retry_timeout: 10
       maximum_lag_on_failover: 1048576 # 1 MB
       postgresql:
         use_pg_rewind: true
         use_slots: true
         parameters:
           max_connections: 100
           shared_buffers: '256MB'
           effective_cache_size: '768MB'
           wal_level: replica
           wal_keep_size: '1GB'
           max_wal_senders: 10
           hot_standby: 'on'
     initdb:
       - encoding: UTF8
       - data-checksums
     pg_hba:
       - host replication replicator 0.0.0.0/0 md5
       - host all all 0.0.0.0/0 md5
     users:
       admin:
         password: admin_password
         options:
           - createrole
           - createdb
   postgresql:
     listen: 0.0.0.0:5432
     connect_address: pg1:5432  # Change to respective hostname/IP
     data_dir: /var/lib/postgresql/14/main
     bin_dir: /usr/lib/postgresql/14/bin
     authentication:
       replication:
         username: replicator
         password: replicator_password
       superuser:
         username: postgres
         password: your_postgres_password
     parameters:
       unix_socket_directories: '/var/run/postgresql'
   tags:
     nofailover: false
     noloadbalance: false
     clonefrom: false
     nosync: false
   ```

   **Important:**
   - Change `name` to the node's hostname (`pg1`, `pg2`, `pg3`).
   - Adjust `connect_address` to the node's hostname or IP.
   - Set `postgres` superuser password appropriately.
   - Ensure `data_dir` points to the PostgreSQL data directory.

3. **Secure the Configuration File:**

   ```bash
   sudo chown patroni:patroni /etc/patroni.yml
   sudo chmod 600 /etc/patroni.yml
   ```

4. **Create Patroni Systemd Service:**

   ```bash
   sudo nano /etc/systemd/system/patroni.service
   ```

   Add the following content:

   ```ini
   [Unit]
   Description=Patroni PostgreSQL Cluster Manager
   After=network.target

   [Service]
   Type=simple
   User=patroni
   Group=patroni
   ExecStart=/opt/patroni/venv/bin/patroni /etc/patroni.yml
   Restart=on-failure
   RestartSec=10s

   [Install]
   WantedBy=multi-user.target
   ```

5. **Reload Systemd and Start Patroni:**

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable patroni
   sudo systemctl start patroni
   sudo systemctl status patroni
   ```

6. **Repeat on All PostgreSQL Nodes:**

   Ensure that the configuration files (`patroni.yml`) are correctly set for each node (`pg1`, `pg2`, `pg3`) and that Patroni is running.

---

## Setting Up HAProxy

HAProxy will act as a load balancer and provide a single entry point to your PostgreSQL cluster.

### On HAProxy Node (`haproxy`):

1. **Install HAProxy:**

   ```bash
   sudo apt update
   sudo apt install -y haproxy
   ```

2. **Configure HAProxy:**

   ```bash
   sudo nano /etc/haproxy/haproxy.cfg
   ```

   Replace the existing content with the following configuration:

   ```ini
   global
       log /dev/log    local0
       log /dev/log    local1 notice
       daemon
       maxconn 4096

   defaults
       log     global
       mode    tcp
       option  tcplog
       option  dontlognull
       retries 3
       timeout connect 5s
       timeout client  50s
       timeout server  50s

   frontend postgres_front
       bind *:5432
       default_backend postgres_back

   backend postgres_back
       mode tcp
       option tcp-check
       default-server send-proxy
       server pg1 pg1:5432 check port 5432 inter 2000 rise 2 fall 3
       server pg2 pg2:5432 check port 5432 inter 2000 rise 2 fall 3
       server pg3 pg3:5432 check port 5432 inter 2000 rise 2 fall 3
   ```

   **Explanation:**
   - **Frontend:** Listens on port `5432` for PostgreSQL connections.
   - **Backend:** Defines the PostgreSQL servers (`pg1`, `pg2`, `pg3`) with health checks.

3. **Enable HAProxy to Start on Boot:**

   ```bash
   sudo systemctl enable haproxy
   sudo systemctl restart haproxy
   sudo systemctl status haproxy
   ```

4. **Firewall Configuration (If Applicable):**

   Ensure that port `5432` is open on the HAProxy node:

   ```bash
   sudo ufw allow 5432/tcp
   ```

---

## Initializing the Cluster

With etcd, Patroni, and PostgreSQL configured, you can now initialize the cluster.

### Steps:

1. **Ensure Patroni is Running on All PostgreSQL Nodes:**

   ```bash
   sudo systemctl status patroni
   ```

   Ensure that Patroni is active and running on all nodes.

2. **Patroni Will Automatically Initialize the Cluster:**

   - The first Patroni node (`pg1`) will become the **primary**.
   - Other nodes (`pg2`, `pg3`) will be **replicas**.

3. **Check Cluster Status:**

   You can check the cluster status using `patronictl`:

   ```bash
   sudo -i -u patroni
   source /opt/patroni/venv/bin/activate
   patronictl -c /etc/patroni.yml list
   deactivate
   exit
   ```

   This should display the current state of each node (Primary or Replica).

---

## Verifying the Setup

### 1. Connect to PostgreSQL via HAProxy

From a client machine, connect to the PostgreSQL cluster using the HAProxy IP and port:

```bash
psql -h haproxy -p 5432 -U postgres -d your_database
```

Ensure that you can connect successfully.

### 2. Simulate Failover

To test high availability, you can simulate a failure:

1. **Stop Patroni on the Primary Node:**

   ```bash
   sudo systemctl stop patroni
   ```

2. **Check if a Replica Promotes to Primary:**

   Use `patronictl` or check the cluster status:

   ```bash
   sudo -i -u patroni
   source /opt/patroni/venv/bin/activate
   patronictl -c /etc/patroni.yml list
   deactivate
   exit
   ```

   A replica should have been promoted to primary.

3. **Restart Patroni on the Original Primary:**

   ```bash
   sudo systemctl start patroni
   ```

   It should now act as a replica.

4. **Verify HAProxy Points to the New Primary:**

   Connect via HAProxy and perform operations to ensure consistency.

### 3. Check HAProxy Health Status

You can enable HAProxy's stats page for better monitoring:

1. **Edit HAProxy Configuration:**

   ```bash
   sudo nano /etc/haproxy/haproxy.cfg
   ```

   Add the following at the end:

   ```ini
   listen stats
       bind *:8404
       stats enable
       stats uri /stats
       stats refresh 10s
       stats auth admin:admin_password
   ```

2. **Restart HAProxy:**

   ```bash
   sudo systemctl restart haproxy
   ```

3. **Access the Stats Page:**

   Open `http://haproxy:8404/stats` in your browser and log in with `admin/admin_password`.

---

## Maintenance and Monitoring

### 1. Backups

Implement regular backups using tools like `pgBackRest` or `barman` to ensure data safety.

### 2. Monitoring

Use monitoring tools like **Prometheus** and **Grafana** to monitor PostgreSQL performance and cluster health.

### 3. Security

- **SSL/TLS:** Configure SSL for PostgreSQL connections to secure data in transit.
- **Firewall:** Restrict access to necessary ports and use security groups if applicable.
- **Updates:** Regularly update PostgreSQL, Patroni, etcd, and HAProxy to the latest security patches.

### 4. Logs

Monitor logs for etcd, Patroni, PostgreSQL, and HAProxy to proactively identify and resolve issues.

- **etcd Logs:**

  ```bash
  sudo journalctl -u etcd -f
  ```

- **Patroni Logs:**

  ```bash
  sudo journalctl -u patroni -f
  ```

- **PostgreSQL Logs:**

  ```bash
  sudo tail -f /var/log/postgresql/postgresql-14-main.log
  ```

- **HAProxy Logs:**

  ```bash
  sudo journalctl -u haproxy -f
  ```

---

## Conclusion

You have successfully set up a highly available PostgreSQL 14 cluster using Patroni, etcd, and HAProxy on Ubuntu 22.04. This setup ensures that your database remains available and resilient against node failures, providing a robust solution for production environments.

**Recommendations:**

- **Testing:** Regularly test failover scenarios to ensure the cluster behaves as expected.
- **Scaling:** For larger deployments, consider adding more PostgreSQL replicas and adjusting HAProxy configurations accordingly.
- **Documentation:** Keep detailed documentation of your setup for maintenance and troubleshooting.

Feel free to reach out if you encounter any issues or need further assistance!