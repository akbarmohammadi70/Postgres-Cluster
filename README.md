```markdown
# PostgreSQL Cluster with Patroni, etcd, HAProxy, and Keepalived

This setup consists of a high-availability PostgreSQL cluster using Patroni for automatic failover, etcd for consensus, and HAProxy for load balancing. The cluster consists of the following nodes:

- **3 PostgreSQL nodes** with Patroni and etcd for managing the database cluster.
- **2 HAProxy nodes** with Keepalived for providing load balancing and high availability for client connections.

## Nodes and IP Addresses

### PostgreSQL Cluster Nodes

| Node Name | Role        | IP Address       |
|-----------|-------------|------------------|
| node1     | PostgreSQL  | 192.168.123.10   |
| node2     | PostgreSQL  | 192.168.123.11   |
| node3     | PostgreSQL  | 192.168.123.12   |

### HAProxy & Keepalived Nodes

| Node Name | Role         | IP Address       |
|-----------|--------------|------------------|
| node4     | HAProxy & Keepalived | 192.168.123.13   |
| node5     | HAProxy & Keepalived | 192.168.123.14   |

## Ansible Roles

Two Ansible roles have been created to deploy and configure the components of this setup:

1. **Patroni and etcd**:
   - Installs and configures Patroni and etcd on the PostgreSQL nodes (node1, node2, node3).
   - Ensures that Patroni automatically manages PostgreSQL failover using etcd for cluster state management.

2. **HAProxy and Keepalived**:
   - Installs and configures HAProxy and Keepalived on the HAProxy nodes (node4, node5).
   - Configures HAProxy to load balance PostgreSQL traffic between the three database nodes.
   - Uses Keepalived to ensure high availability for the HAProxy instances.

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

2. **HAProxy Load Balancer**:
   - Test the HAProxy setup by connecting to the provided virtual IP or DNS name and verify traffic distribution across the database nodes.

## Requirements

- Ansible 2.x or later
- Python 3.x
- Vagrant (for local development)
- PostgreSQL 13 or later
- Keepalived and HAProxy

## License

This setup is licensed under the MIT License. See the LICENSE file for more details.
```

### Key Changes:
- **Python Virtual Environment Setup**: Added instructions for creating and activating a Python virtual environment (`venv`).
- **Installation of Requirements**: Added a step to install the necessary dependencies using `pip install -r requirements.txt`.
- **Step-by-step Ansible Playbook Execution**: The playbooks are run after setting up the virtual environment and installing the dependencies.

Make sure you have a `requirements.txt` file containing the necessary Python packages like `ansible`, `paramiko`, etc., for the setup to work properly.