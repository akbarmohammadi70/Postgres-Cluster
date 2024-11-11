# PostgreSQL Cluster with Patroni, etcd, HAProxy, Keepalived, and CQRS

This setup consists of a high-availability PostgreSQL cluster using Patroni for automatic failover, etcd for consensus, and HAProxy for load balancing. The cluster is configured with **CQRS (Command Query Responsibility Segregation)**, where the **leader** database is used for write operations and the **standby** databases are used for read operations. The leader and standby databases are differentiated by their respective ports:

- **Leader Database** (Port: 5000) - Used for write (command) operations.
- **Standby Database** (Port: 5001) - Used for read (query) operations.

The cluster consists of the following nodes:

- **3 PostgreSQL nodes** with Patroni and etcd for managing the database cluster.
- **2 HAProxy nodes** with Keepalived for providing load balancing and high availability for client connections.

## Nodes and IP Addresses

### PostgreSQL Cluster Nodes

| Node Name | Role        | IP Address       | OS Version     | PostgreSQL Port |
|-----------|-------------|------------------|----------------|-----------------|
| node1     | PostgreSQL (Leader)  | 192.168.123.10   | Ubuntu 22.04   | 5000            |
| node2     | PostgreSQL (Standby) | 192.168.123.11   | Ubuntu 22.04   | 5001            |
| node3     | PostgreSQL (Standby) | 192.168.123.12   | Ubuntu 22.04   | 5001            |

### HAProxy & Keepalived Nodes

| Node Name | Role         | IP Address       | OS Version     |
|-----------|--------------|------------------|----------------|
| node4     | HAProxy & Keepalived | 192.168.123.13   | Ubuntu 22.04   |
| node5     | HAProxy & Keepalived | 192.168.123.14   | Ubuntu 22.04   |

### Virtual IP for Keepalived

- **Keepalived Virtual IP (VIP)**: `192.168.123.100`

  The **Virtual IP (VIP)** provided by Keepalived will be used for client connections to the PostgreSQL cluster, ensuring high availability and automatic failover.

## Ansible Roles

Two Ansible roles have been created to deploy and configure the components of this setup:

1. **Patroni and etcd**:
   - Installs and configures Patroni and etcd on the PostgreSQL nodes (node1, node2, node3).
   - Ensures that Patroni automatically manages PostgreSQL failover using etcd for cluster state management.

2. **HAProxy and Keepalived**:
   - Installs and configures HAProxy and Keepalived on the HAProxy nodes (node4, node5).
   - Configures HAProxy to load balance PostgreSQL traffic between the three database nodes. The HAProxy configuration ensures that write traffic is directed to the leader (port 5000) and read traffic is directed to the standby nodes (port 5001).
   - Uses Keepalived to provide the **Virtual IP** (`192.168.123.100`), ensuring that the HAProxy nodes remain available even if one of them fails.

## Setup Steps

### Step 1: Create and Activate Python Virtual Environment

Before running the playbooks, it's recommended to set up a Python virtual environment to manage dependencies. This ensures a clean setup without interfering with your system Python packages.

1. **Create a virtual environment**:
   ```bash
   python3 -m venv venv
   ```

2. **Activate the virtual environment**:
   - On Linux/macOS:
     ```bash
     source venv/bin/activate
     ```
   - On Windows:
     ```bash
     venv\Scripts\activate
     ```

### Step 2: Install Requirements

Once the virtual environment is active, you need to install the required dependencies for Ansible.

1. **Install the requirements**:
   ```bash
   pip install -r requirements.txt
   ```

   The `requirements.txt` file should contain the necessary Python packages, such as `ansible`, `paramiko`, and any other dependencies for your setup.

### Step 3: Modify Inventory File

Ensure the IP addresses and hostnames for the PostgreSQL and HAProxy nodes match your setup. Edit the `inventory` file as needed.

### Step 4: Run the Ansible Playbooks

Now that your virtual environment is set up and dependencies are installed, you can run the Ansible playbooks to configure the cluster.

1. **Run the playbook to configure the PostgreSQL cluster**:
   ```bash
   ansible-playbook -i inventory playbook-patroni-etcd.yml
   ```

2. **Run the playbook to configure HAProxy and Keepalived**:
   ```bash
   ansible-playbook -i inventory playbook-haproxy-keepalived.yml
   ```

### Step 5: Verify the Setup

1. **PostgreSQL Cluster**:
   - Check the Patroni status to ensure all PostgreSQL nodes are synchronized and healthy.
   - Verify that the leader database is on port 5000 and the standby databases are on port 5001.

2. **HAProxy Load Balancer**:
   - Test the HAProxy setup by connecting to the provided **Virtual IP (VIP)** (`192.168.123.100`) and verify traffic distribution:
     - Write requests should be directed to port 5000 (Leader).
     - Read requests should be directed to port 5001 (Standby).

## Requirements

- Ansible 2.x or later
- Python 3.x
- Vagrant (for local development)
- PostgreSQL 13 or later
- Keepalived and HAProxy

## License

This setup is licensed under the MIT License. See the LICENSE file for more details.
