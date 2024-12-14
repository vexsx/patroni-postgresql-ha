Okay, let's outline how to configure Patroni and etcd on 3 nodes with a separate OpenStack load balancer. This will be a detailed guide, covering the key steps and configuration parameters.

**Assumptions:**

*   **Operating System:** You're using a Linux distribution supported by Patroni and etcd (e.g., CentOS, Ubuntu, Debian).
*   **Network:** All nodes can communicate with each other (including the load balancer) over the network. Ensure necessary ports are open (etcd: 2379, 2380; Patroni REST API: 8008; PostgreSQL: 5432).
*   **OpenStack Load Balancer:** You have a working OpenStack environment and can create/configure a load balancer instance.
*   **PostgreSQL:** You have PostgreSQL installed on the 3 nodes, or you will install it as part of the process.
*   **User:** You have a user with `sudo` privileges on each node.
*   **Python:** Python 3 is installed on the nodes where you'll run Patroni.

**Steps**

**Phase 1: Install and Configure etcd (on all 3 nodes)**

1. **Install etcd:**

    ```bash
    # Example for CentOS/RHEL:
    sudo yum install etcd

    # Example for Ubuntu/Debian:
    sudo apt-get update
    sudo apt-get install etcd
    ```

2. **Configure etcd:**
    *   Edit the etcd configuration file (usually `/etc/etcd/etcd.conf` or `/etc/default/etcd`).
    *   **Important settings:**
        *   `ETCD_NAME`: A unique name for each node (e.g., `etcd-node1`, `etcd-node2`, `etcd-node3`).
        *   `ETCD_DATA_DIR`: The directory where etcd will store its data (e.g., `/var/lib/etcd`).
        *   `ETCD_LISTEN_PEER_URLS`: The URLs that etcd will listen on for peer communication (between etcd members). Use the node's private IP address (e.g., `http://<node1_private_ip>:2380`).
        *   `ETCD_LISTEN_CLIENT_URLS`: The URLs that etcd will listen on for client communication (from Patroni, etc.). Use the node's private IP and optionally a public-facing IP if needed (e.g., `http://<node1_private_ip>:2379,http://0.0.0.0:2379`).
        *   `ETCD_INITIAL_ADVERTISE_PEER_URLS`: The URLs that this etcd instance will advertise to other members (e.g., `http://<node1_private_ip>:2380`).
        *   `ETCD_ADVERTISE_CLIENT_URLS`: The URLs that this etcd instance will advertise to clients (e.g., `http://<node1_private_ip>:2379,http://<node1_public_ip>:2379`).
        *   `ETCD_INITIAL_CLUSTER`: A comma-separated list of all initial cluster members in the format `name=peer_url` (e.g., `etcd-node1=http://<node1_private_ip>:2380,etcd-node2=http://<node2_private_ip>:2380,etcd-node3=http://<node3_private_ip>:2380`).
        *   `ETCD_INITIAL_CLUSTER_STATE`: Set this to `new` for the initial cluster setup.
        *   `ETCD_INITIAL_CLUSTER_TOKEN`: A unique token for the cluster (e.g., `my-etcd-cluster`).

    *   **Example `/etc/etcd/etcd.conf` (for etcd-node1):**

    ```
    ETCD_NAME="etcd-node1"
    ETCD_DATA_DIR="/var/lib/etcd/data"
    ETCD_LISTEN_PEER_URLS="http://<node1_private_ip>:2380"
    ETCD_LISTEN_CLIENT_URLS="http://<node1_private_ip>:2379,http://0.0.0.0:2379"
    ETCD_INITIAL_ADVERTISE_PEER_URLS="http://<node1_private_ip>:2380"
    ETCD_ADVERTISE_CLIENT_URLS="http://<node1_private_ip>:2379,http://<node1_public_ip>:2379"
    ETCD_INITIAL_CLUSTER="etcd-node1=http://<node1_private_ip>:2380,etcd-node2=http://<node2_private_ip>:2380,etcd-node3=http://<node3_private_ip>:2380"
    ETCD_INITIAL_CLUSTER_STATE="new"
    ETCD_INITIAL_CLUSTER_TOKEN="my-etcd-cluster"
    ```

3. **Enable and start etcd:**

    ```bash
    sudo systemctl enable etcd
    sudo systemctl start etcd
    ```

4. **Verify etcd cluster health:**

    ```bash
    etcdctl --endpoints=<node1_private_ip>:2379,<node2_private_ip>:2379,<node3_private_ip>:2379 cluster-health
    ```

    You should see all three members reporting as healthy.

**Phase 2: Install and Configure Patroni (on all 3 nodes)**

1. **Install Patroni:**

    ```bash
    # Using pip (recommended):
    sudo pip3 install patroni[etcd]

    # Or from your distribution's package manager if available.
    ```

2. **Create a Patroni configuration file (e.g., `patroni.yml`):**

    ```yaml
    scope: my-cluster # A unique name for your Patroni cluster
    namespace: /db/postgres # the path your cluster information is stored under in your DCS
    name: node1 # Unique name for each node (node1, node2, node3)

    restapi:
      listen: 0.0.0.0:8008 # Patroni REST API listen address
      connect_address: <node1_private_ip>:8008 # How Patroni advertises itself to others

    etcd:
      hosts:  # List of etcd endpoints
        - <node1_private_ip>:2379
        - <node2_private_ip>:2379
        - <node3_private_ip>:2379
      # If using older etcd API, specify protocol version:
      # protocol_version: 2

    postgresql:
      use_pg_rewind: true  # Highly recommended for faster failover
      use_slots: true      # Recommended for replication
      data_dir: /var/lib/postgresql/13/main/ # Path to PostgreSQL data directory
      bin_dir:  /usr/lib/postgresql/13/bin/  # Path to PostgreSQL binaries (if needed)
      config_dir: /etc/postgresql/13/main/ #Path to the PostgreSQL config directory
      connect_address: <node1_private_ip>:5432 # How PostgreSQL advertises itself to others
      listen: 0.0.0.0:5432  # PostgreSQL listen address
      authentication:
          replication:
              username: repluser  # Replication user (create this in PostgreSQL)
              password: replpassword
          rewind:
              username: rewinduser
              password: rewindpassword
      parameters:
        max_connections: 200
        shared_buffers: 512MB
        wal_level: replica       # Required for replication
        max_wal_senders: 10     # At least the number of standbys
        wal_keep_size: 1GB    # Adjust based on WAL generation rate
        hot_standby: on         # Enable read-only queries on standbys

      # Optional but recommended:
      callbacks:  # Customize Patroni's behavior (e.g., run scripts on role change)
        on_start:
          - name: script1
            command: /path/to/your/script.sh
            role: any  # Run on any role (primary, standby)
            on_success:
            on_failure:
            timeout: 30 # Timeout in seconds
      create_replica_methods:
        - pg_basebackup
      pg_basebackup:
        - wal=stream # Stream WAL during basebackup
        #- max_wal_send_size: 10M # set a limit on max_wal_send_size
      pg_rewind:
        # Authentication details if different from replication
        # username: rewind_user
        # password: rewind_password
      recovery_conf:   # Custom settings for recovery.conf (if needed)
        restore_command: cp /path/to/archived/wals/%f %p

      # Optional: Configure archive_command to copy WAL files to an archive
      # archive_mode: on
      # archive_command: 'cp %p /path/to/archive/%f'

      # Set up a replication user (if it doesn't exist)
      users:
        repluser:
          password: "replpassword"
          options:
            - "replication"
    ```

    *   **Important:**
        *   Adjust the `data_dir`, `bin_dir`, `config_dir` to match your PostgreSQL installation.
        *   Replace placeholders like `<node1_private_ip>` with the actual IP addresses of your nodes.
        *   Create the `repluser` and `rewinduser` with appropriate passwords in PostgreSQL.
        *   Configure `recovery_conf` (especially `restore_command`) if you are using an external archive for WAL files.
        *   Customize `callbacks` as needed for your environment.

3. **Start Patroni:**

    ```bash
    sudo patroni patroni.yml
    ```

    *   It's highly recommended to run Patroni under a service manager (like systemd) for automatic startup and restart.

**Phase 3: Create a PostgreSQL User for Replication**

1. Connect to one of your PostgreSQL instances (which will become the primary initially):

    ```bash
    sudo -u postgres psql
    ```

2. Create the replication user:

    ```sql
    CREATE USER repluser WITH REPLICATION ENCRYPTED PASSWORD 'replpassword';
    CREATE USER rewinduser WITH ENCRYPTED PASSWORD 'rewindpassword';
    ```
    
    *   Adjust the username and password if you specified different ones in `patroni.yml`.

**Phase 4: Configure the OpenStack Load Balancer**

1. **Create a Load Balancer:** Use the OpenStack dashboard or CLI to create a new load balancer instance.
2. **Create a Listener:** Create a listener for the load balancer. Typically, you'll need two listeners:
    *   **Listener 1 (Read-Write):**
        *   Protocol: TCP
        *   Port: 5432 (or your desired PostgreSQL port)
    *   **Listener 2 (Read-Only):**
        *   Protocol: TCP
        *   Port: 5433 (or another port for read-only traffic - this is optional if you want to use a single port)
3. **Create a Pool:** Create two pools, one for each listener:
    *   **Pool 1 (Read-Write):**
        *   Load Balancing Algorithm: LEAST\_CONNECTIONS (or another suitable algorithm)
        *   Protocol: TCP
    *   **Pool 2 (Read-Only):**
        *   Load Balancing Algorithm: ROUND\_ROBIN (or another suitable algorithm for distributing reads)
        *   Protocol: TCP
4. **Create Health Monitors:** Create health monitors for each pool to check the health of the PostgreSQL instances:
    *   **Health Monitor 1 (Primary):**
        *   Type: TCP
        *   Port: 5432
        *   **Important:** You will need a custom script or external tool to regularly update the load balancer pool to only include the current primary.
    *   **Health Monitor 2 (Standby):**
        *   Type: TCP
        *   Port: 5432
        *   **Important:** Similar to the primary health monitor, you'll need a way to update the pool with the current standbys.
5. **Add Members (Nodes) to Pools:** This is the dynamic part. You will need to add members (your PostgreSQL nodes) to the pools using a script that queries the Patroni REST API.

**Phase 5: Service Discovery and Load Balancer Updates**

1. **Write a Script:** Create a script (e.g., in Python) that does the following:
    *   Queries the Patroni REST API on each node (port 8008) to get the cluster status (use `/leader`, `/replica`, and `/members` endpoints).
    *   Parses the JSON response to identify the current primary and standbys.
    *   Uses the OpenStack API (or CLI) to:
        *   Remove all current members from both pools.
        *   Add only the current primary to the Read-Write pool.
        *   Add the current primary and standbys to the Read-Only pool.

2. **Example Python Script (using OpenStack SDK):**
    ```python
    import openstack
    import requests

    # OpenStack Connection Details
    conn = openstack.connect(cloud='your_openstack_cloud')  # Replace with your OpenStack cloud name
    lb_id = 'your_loadbalancer_id'  # Replace with your load balancer ID
    rw_pool_id = 'your_read_write_pool_id'  # Replace with your read-write pool ID
    ro_pool_id = 'your_read_only_pool_id'  # Replace with your read-only pool ID

    # Patroni Nodes
    patroni_nodes = [
        {'name': 'node1', 'ip': '<node1_private_ip>'},
        {'name': 'node2', 'ip': '<node2_private_ip>'},
        {'name': 'node3', 'ip': '<node3_private_ip>'},
    ]

    def get_patroni_status(node_ip):
        try:
            response = requests.get(f'http://{node_ip}:8008/cluster')
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            print(f"Error querying Patroni on {node_ip}: {e}")
            return None

    def update_load_balancer():
        primary = None
        replicas = []

        for node in patroni_nodes:
            status = get_patroni_status(node['ip'])
            if status:
                for member in status['members']:
                    if member['name'] == node['name']:
                        if member['role'] == 'leader':
                            primary = node
                        elif member['role'] == 'replica':
                            replicas.append(node)

        # Clear existing pool members
        for member in conn.load_balancer.members(rw_pool_id):
            conn.load_balancer.delete_member(member, rw_pool_id)
        for member in conn.load_balancer.members(ro_pool_id):
            conn.load_balancer.delete_member(member, ro_pool_id)

        # Add primary to read-write pool
        if primary:
            conn.load_balancer.create_member(
                rw_pool_id,
                address=primary['ip'],
                protocol_port=5432,
                subnet_id='your_subnet_id'  # Replace with your subnet ID
            )
            print(f"Added {primary['name']} (primary) to read-write pool")

        # Add primary and replicas to read-only pool
        if primary or replicas:
            all_nodes = [primary] + replicas if primary else replicas
            for node in all_nodes:
                conn.load_balancer.create_member(
                    ro_pool_id,
                    address=node['ip'],
                    protocol_port=5432,
                    subnet_id='your_subnet_id'  # Replace with your subnet ID
                )
                print(f"Added {node['name']} to read-only pool")

    if __name__ == '__main__':
        update_load_balancer()
    ```

3. **Schedule the Script:** Use `cron` or another scheduler to run the script periodically (e.g., every 30 seconds or 1 minute). This will keep the load balancer configuration in sync with the Patroni cluster's state.

**Important Considerations**

*   **Error Handling:** Enhance the script with robust error handling to deal with network issues, Patroni API errors, and OpenStack API errors.
*   **Security:** Secure communication between the script, Patroni REST API, and OpenStack API (e.g., using HTTPS, API keys).
*   **Testing:** Thoroughly test the entire setup, including failover scenarios, to ensure that the load balancer correctly updates and routes traffic.
*   **Monitoring:** Monitor the health of etcd, Patroni, PostgreSQL, and the load balancer. Set up alerts for any issues.
*   **Idempotency:** Ensure that the script is idempotent, meaning that running it multiple times has the same effect as running it once. This prevents unintended changes to the load balancer configuration.
*   **Alternative Service Discovery:** Consider using tools like Consul, which can integrate with Patroni and provide service discovery, potentially simplifying the load balancer configuration update process.
*   **Read-Only Pool Port:** If you are using a separate read-only pool, make sure your application is configured to connect to the correct port (e.g., 5433) for read operations.

This detailed guide should help you set up Patroni with etcd and an OpenStack load balancer. Remember to adapt the configurations and script to your specific environment and requirements. Good luck!
