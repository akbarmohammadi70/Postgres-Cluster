# PostgreSQL High Availability Cluster with Patroni, etcd, HAProxy, Keepalived, and CQRS-style Read/Write Routing

This project provides an Ansible-based deployment of a highly available PostgreSQL 17 cluster using **Patroni**, **etcd**, **HAProxy**, and **Keepalived**.

The main goal is to provide:

* PostgreSQL high availability
* Automatic PostgreSQL failover
* Distributed leader election
* Streaming replication
* Read/write traffic separation
* Highly available client access through a Virtual IP
* Reproducible infrastructure using Ansible and Vagrant
* Secure management of credentials using Ansible Vault

The architecture consists of:

* **3 PostgreSQL nodes** running Patroni and etcd
* **2 HAProxy nodes** running HAProxy and Keepalived
* **1 Virtual IP (VIP)** used by clients

---

# Architecture

```text
                         Applications
                              |
                              |
                         Virtual IP
                              |
                    +---------+---------+
                    |                   |
                    v                   v
              HAProxy Node 1      HAProxy Node 2
                 ACTIVE              BACKUP
                    |                   |
                    +---------+---------+
                              |
                 +------------+------------+
                 |                         |
              Port 5000                 Port 5001
              WRITE                     READ
                 |                         |
                 v                         v
        Patroni /master            Patroni /replica
                 |                         |
                 v                         v
        +----------------+       +--------------------+
        | PostgreSQL     |       | PostgreSQL         |
        | Leader         |       | Standby Replicas   |
        +----------------+       +--------------------+
              |                         ^
              | Streaming Replication   |
              +-------------------------+
                        |
                        v
                +---------------+
                |     etcd      |
                |  3-node DCS   |
                +---------------+
```

## Components

| Component     | Purpose                                                |
| ------------- | ------------------------------------------------------ |
| PostgreSQL 17 | Database engine                                        |
| Patroni       | PostgreSQL HA, leader election and failover management |
| etcd          | Distributed Configuration Store (DCS) used by Patroni  |
| HAProxy       | PostgreSQL traffic routing and health checking         |
| Keepalived    | Virtual IP and load balancer failover                  |
| Ansible       | Infrastructure automation                              |
| Vagrant       | Reproducible local environment                         |

---

# PostgreSQL Cluster

The PostgreSQL cluster consists of three nodes.

| Node  | Role    | IP            | PostgreSQL |
| ----- | ------- | ------------- | ---------- |
| node1 | Leader  | `192.168.x.x` | `5432`     |
| node2 | Replica | `192.168.x.x` | `5432`     |
| node3 | Replica | `192.168.x.x` | `5432`     |

The actual PostgreSQL service listens on port **5432**.

The PostgreSQL nodes use Patroni to manage the cluster state and failover.

Only one node is the Patroni leader at a time.

The remaining nodes operate as PostgreSQL replicas using streaming replication.

The leader can change during a failover, therefore applications should not connect directly to a specific PostgreSQL node.

---

# HAProxy and Keepalived

Two load balancer nodes are deployed:

| Node  | Role                 | IP            |
| ----- | -------------------- | ------------- |
| node4 | HAProxy + Keepalived | `192.168.x.x` |
| node5 | HAProxy + Keepalived | `192.168.x.x` |

Keepalived provides a floating Virtual IP:

```text
192.168.x.x
```

Clients connect to this VIP instead of connecting directly to a PostgreSQL node.

If the active HAProxy node fails, Keepalived moves the VIP to the backup HAProxy node.

---

# HAProxy Traffic Routing

HAProxy exposes two logical PostgreSQL endpoints.

## Write endpoint

```text
<VIP>:5000
```

This endpoint uses the Patroni health endpoint:

```text
/master
```

Only the current Patroni leader is considered healthy.

Therefore write traffic is always routed to the current PostgreSQL leader.

```text
Client
   |
   v
VIP:5000
   |
   v
HAProxy
   |
   +---- node1 /master -> 200 -> PostgreSQL Leader
   |
   +---- node2 /master -> non-200
   |
   +---- node3 /master -> non-200
```

## Read endpoint

```text
<VIP>:5001
```

This endpoint uses:

```text
/replica
```

Only healthy PostgreSQL replicas are selected.

```text
Client
   |
   v
VIP:5001
   |
   v
HAProxy
   |
   +---- node1 /replica -> not selected
   |
   +---- node2 /replica -> healthy
   |
   +---- node3 /replica -> healthy
```

This provides **CQRS-style read/write routing**.

It is important to note that this is traffic separation inspired by CQRS rather than a complete CQRS implementation. The application is responsible for deciding which operations are sent to the write or read endpoint.

---

# Why This Architecture?

## Patroni

Patroni was selected because it provides PostgreSQL high-availability functionality including:

* Leader election
* Automatic failover
* PostgreSQL lifecycle management
* Replica management
* Health checking
* Integration with distributed configuration stores
* `pg_rewind` support
* Replication slot support

## etcd

etcd is used as Patroni's Distributed Configuration Store.

A three-node etcd cluster is used so that the cluster can tolerate a single etcd node failure while maintaining quorum.

```text
             etcd cluster

          +-------+-------+-------+
          |       |       |       |
          v       v       v       v
        node1   node2   node3

              quorum = 2
```

## HAProxy

HAProxy was selected for:

* PostgreSQL traffic routing
* Patroni health checks
* Separating leader and replica traffic
* Fast detection of unhealthy PostgreSQL nodes

## Keepalived

Keepalived provides a highly available Virtual IP between the two HAProxy nodes.

This removes the load balancer itself as a single point of failure.

---

# How High Availability Is Achieved

The architecture provides HA at multiple layers.

## PostgreSQL Layer

PostgreSQL uses streaming replication:

```text
             Leader
                |
        Streaming Replication
           /          \
          v            v
       Replica       Replica
```

If the leader fails, Patroni can promote a healthy replica.

---

## Patroni Layer

Patroni continuously monitors:

* PostgreSQL health
* Cluster state
* Replication state
* Leader lock

Patroni uses etcd for distributed leader election.

A simplified failover flow is:

```text
Leader fails
     |
     v
Patroni detects failure
     |
     v
Leader lock expires
     |
     v
Patroni selects a healthy replica
     |
     v
Replica is promoted
     |
     v
New /master endpoint becomes healthy
     |
     v
HAProxy sends write traffic to new leader
```

---

## etcd Layer

Three etcd nodes provide quorum.

With three members:

```text
3 healthy nodes -> quorum
2 healthy nodes -> quorum
1 healthy node  -> no quorum
```

Therefore one etcd node can fail without losing consensus.

---

## Load Balancer Layer

Two HAProxy nodes are protected by Keepalived.

```text
             VIP
              |
       +------+------+
       |             |
       v             v
    HAProxy 1     HAProxy 2
     ACTIVE        BACKUP
```

If HAProxy 1 fails:

```text
HAProxy 1
    X
    |
    v
Keepalived
    |
    v
VIP moves to HAProxy 2
```

Applications continue connecting to the same VIP.

---

# Repository Structure

```text
.
├── HAproxy-cluster
│   ├── handlers
│   │   └── main.yml
│   ├── tasks
│   │   ├── haproxy.yml
│   │   ├── keepalived.yml
│   │   ├── main.yml
│   │   └── requirments.yml
│   ├── templates
│   │   ├── haproxy.cfg.j2
│   │   ├── keepalived.conf.node1.j2
│   │   └── keepalived.conf.node2.j2
│   └── vars
│       ├── haproxy-vault.yml
│       └── main.yml
│
├── postgres-cluster
│   ├── handlers
│   │   └── main.yml
│   ├── tasks
│   │   ├── etcd.yml
│   │   ├── main.yml
│   │   ├── patroni.yml
│   │   ├── postgresql.yml
│   │   └── requirments.yml
│   ├── templates
│   │   ├── config.yml.j2
│   │   ├── etcd.j2
│   │   └── etcd-override.conf.j2
│   └── vars
│       ├── main.yml
│       └── postgres-vault.yml
│
├── install.yml
├── inventory.ini
├── README.md
├── requirments.txt
└── Vagrantfile
```

---

# Requirements

The following software is required on the Ansible control machine:

* Python 3
* Ansible
* Vagrant
* Git
* A virtualization provider supported by Vagrant

The target servers run:

```text
Ubuntu 22.04
```

The database version used by this project is:

```text
PostgreSQL 17
```

---

# Installation

## 1. Clone the Repository

```bash
git clone <repository-url>
cd Postgres-Cluster
```

---

## 2. Create Python Virtual Environment

```bash
python3 -m venv venv
```

Activate it:

### Linux / macOS

```bash
source venv/bin/activate
```

### Windows

```powershell
venv\Scripts\activate
```

---

## 3. Install Python Dependencies

The repository contains:

```text
requirments.txt
```

Install the dependencies:

```bash
pip install -r requirments.txt
```

If the file is renamed to the conventional:

```text
requirements.txt
```

use:

```bash
pip install -r requirements.txt
```

---

# 4. Start the Vagrant Environment

For local testing:

```bash
vagrant up
```

Check the machines:

```bash
vagrant status
```

---

# 5. Configure Inventory

Edit:

```text
inventory.ini
```

and configure the IP addresses and host groups for:

```text
PostgreSQL / Patroni / etcd
HAProxy / Keepalived
```

Example:

```ini
[postgres]
node1 ansible_host=192.168.x.x
node2 ansible_host=192.168.x.x
node3 ansible_host=192.168.x.x

[lb]
node4 ansible_host=192.168.x.x
node5 ansible_host=192.168.x.x
```

The actual IP addresses should be defined according to the local environment.

---

# 6. Configure Variables

The main variables are stored in:

```text
postgres-cluster/vars/main.yml
HAproxy-cluster/vars/main.yml
```

Sensitive values should be stored in:

```text
postgres-cluster/vars/postgres-vault.yml
HAproxy-cluster/vars/haproxy-vault.yml
```

These files should be encrypted with Ansible Vault.

Do not commit plaintext credentials to Git.

---

# 7. Encrypt Secrets

For example:

```bash
ansible-vault encrypt postgres-cluster/vars/postgres-vault.yml
```

and:

```bash
ansible-vault encrypt HAproxy-cluster/vars/haproxy-vault.yml
```

The Vault password should be provided securely when running the playbook.

---

# 8. Test Ansible Connectivity

Before deployment:

```bash
ansible all -i inventory.ini -m ping
```

All target nodes should return:

```text
SUCCESS
```

---

# 9. Deploy the Cluster

Run:

```bash
ansible-playbook \
  -i inventory.ini \
  install.yml \
  --ask-vault-pass
```

The playbook deploys:

```text
PostgreSQL
Patroni
etcd
HAProxy
Keepalived
```

according to the inventory and role configuration.

---

# Ansible Roles

## PostgreSQL Cluster Role

Location:

```text
postgres-cluster/
```

This role is responsible for:

* Installing PostgreSQL
* Installing Patroni
* Installing etcd
* Configuring etcd
* Configuring Patroni
* Configuring PostgreSQL
* Configuring authentication
* Configuring replication
* Configuring PostgreSQL HA parameters

---

## HAProxy Cluster Role

Location:

```text
HAproxy-cluster/
```

This role is responsible for:

* Installing HAProxy
* Installing Keepalived
* Configuring HAProxy
* Configuring Patroni health checks
* Configuring the Virtual IP
* Configuring HAProxy failover behavior

---

# Deployment Tags

If the playbook defines tags, the PostgreSQL stack can be deployed independently:

```bash
ansible-playbook \
  -i inventory.ini \
  install.yml \
  --tags postgres \
  --ask-vault-pass
```

HAProxy and Keepalived can be deployed independently:

```bash
ansible-playbook \
  -i inventory.ini \
  install.yml \
  --tags lb \
  --ask-vault-pass
```

---

# PostgreSQL Configuration

Important PostgreSQL HA settings include:

```yaml
wal_level: hot_standby
hot_standby: "on"
max_wal_senders: 10
max_replication_slots: 10
wal_log_hints: "on"
```

Patroni is configured to use:

```yaml
use_pg_rewind: true
use_slots: true
```

Replication slots help prevent required WAL from being removed while replicas still need it.

`pg_rewind` helps rejoin a former primary after failover when the required conditions are met.

---

# WAL Archiving

WAL archiving is enabled using:

```yaml
archive_mode: "on"
```

The archive location is:

```text
/var/lib/wal_archive
```

The archive directory is owned by PostgreSQL:

```text
postgres:postgres
```

The archive command uses an absolute path to avoid dependency on PostgreSQL's current working directory.

Example:

```text
test ! -f /var/lib/wal_archive/%f &&
cp %p /var/lib/wal_archive/%f
```

The corresponding recovery command is:

```text
cp /var/lib/wal_archive/%f %p
```

For the current lab implementation this is a local archive.

A production environment should use remote durable storage.

---

# Security

The PostgreSQL cluster uses SCRAM-SHA-256 authentication.

```yaml
auth: scram-sha-256
```

and:

```yaml
password_encryption: scram-sha-256
```

Sensitive credentials are managed using Ansible Vault.

For a production deployment, network access should also be restricted.

Recommended access policy:

```text
PostgreSQL 5432
    -> application / HAProxy networks only

etcd 2379
    -> Patroni / etcd clients only

etcd 2380
    -> etcd cluster members only

Patroni 8008
    -> HAProxy / monitoring only
```

The current lab configuration is intentionally simpler and should be hardened before production use.

---

# Resource Limits and OS Tuning

The environment also considers operating-system resource limits required for high connection workloads.

Examples include:

```text
fs.nr_open
fs.file-max
net.core.somaxconn
net.core.netdev_max_backlog
net.ipv4.tcp_max_syn_backlog
net.ipv4.ip_local_port_range
```

For systemd-managed services, service-specific limits such as `LimitNOFILE` should be configured using systemd drop-ins.

This is important because PostgreSQL, Patroni and HAProxy are system services and interactive shell `ulimit` settings do not necessarily apply to them.

---

# Validation

After deployment, the environment should be validated at several levels.

## 1. Check Ansible

```bash
ansible all -i inventory.ini -m ping
```

---

## 2. Check etcd

On an etcd node:

```bash
etcdctl member list
```

Check cluster health:

```bash
etcdctl endpoint health --cluster
```

Check cluster status:

```bash
etcdctl endpoint status --cluster
```

All three etcd members should be healthy and the cluster should have a leader.

---

# 3. Check Patroni

```bash
patronictl -c /etc/patroni/config.yml list
```

Expected topology:

```text
+--------+---------+
| Member | Role    |
+--------+---------+
| node1  | Leader  |
| node2  | Replica |
| node3  | Replica |
+--------+---------+
```

The actual leader can be any of the three nodes.

---

# 4. Check PostgreSQL

Check Patroni:

```bash
systemctl status patroni
```

Connect to PostgreSQL:

```bash
psql \
  -h <postgresql-ip> \
  -p 5432 \
  -U <admin-user> \
  -d postgres
```

Check replication:

```sql
SELECT
    client_addr,
    state,
    sync_state,
    replay_lag
FROM pg_stat_replication;
```

---

# 5. Check HAProxy

Check HAProxy:

```bash
systemctl status haproxy
```

The HAProxy statistics page is available on:

```text
http://<haproxy-ip>:8080/
```

The write endpoint is:

```text
<VIP>:5000
```

The read endpoint is:

```text
<VIP>:5001
```

---

# 6. Check Keepalived

```bash
systemctl status keepalived
```

Check the VIP:

```bash
ip addr
```

The VIP should exist on exactly one HAProxy node at a time.

---

# Failover Testing

High availability should not only be configured; it should also be tested.

## PostgreSQL Failover

First identify the leader:

```bash
patronictl -c /etc/patroni/config.yml list
```

Stop Patroni on the leader:

```bash
systemctl stop patroni
```

After the Patroni TTL expires, check:

```bash
patronictl -c /etc/patroni/config.yml list
```

One of the healthy replicas should become the new leader.

Then verify the HAProxy write endpoint.

The `/master` health check should now identify the new leader.

---

# HAProxy Failover

Identify the active HAProxy node and stop HAProxy:

```bash
systemctl stop haproxy
```

Check the VIP:

```bash
ip addr
```

The VIP should move to the backup Keepalived node.

Applications should continue using the same VIP.

---

# Read / Write Validation

## Write traffic

Connect through:

```text
<VIP>:5000
```

This should always reach the current Patroni leader.

## Read traffic

Connect through:

```text
<VIP>:5001
```

This should reach healthy PostgreSQL replicas.

This separation allows applications to direct:

```text
Commands / Writes -> :5000
Queries / Reads    -> :5001
```

---

# Assumptions

The implementation makes the following assumptions:

1. The environment is an internal lab/test environment.
2. Three PostgreSQL nodes are available.
3. Three etcd members are available.
4. Two HAProxy/Keepalived nodes are available.
5. Applications can separate read and write database traffic.
6. PostgreSQL replicas use streaming replication.
7. Ansible is the source of truth for infrastructure configuration.
8. Secrets are managed using Ansible Vault.
9. Vagrant is used to reproduce the test environment.
10. The local WAL archive is intended for demonstrating WAL archiving and recovery mechanisms, not as a complete production backup solution.

---

# Design Trade-offs

## Three PostgreSQL Nodes

Three nodes provide:

* One leader
* Two replicas
* Better failover options
* Ability to tolerate individual node failures

The trade-off is increased infrastructure and operational complexity compared with a single primary/replica deployment.

---

## Three etcd Nodes

Three etcd members provide quorum and allow one etcd node to fail without losing consensus.

Using only two etcd nodes would not provide the same failure tolerance because quorum would be lost after one node failure.

---

## HAProxy Instead of PgBouncer

HAProxy is responsible for:

* Routing
* Health checking
* Failover

It is not a PostgreSQL connection pooler.

For environments with a very high number of application connections, PgBouncer could be introduced.

Example:

```text
Application
     |
     v
 HAProxy
     |
     v
 PgBouncer
     |
     v
PostgreSQL
```

---

## Local WAL Archive

The current implementation uses:

```text
/var/lib/wal_archive
```

This is simple and useful for the lab environment.

However, it is not a sufficient disaster-recovery strategy because the archive is located on the same infrastructure.

A production implementation should use remote durable storage.

---

# Production Improvements

Before using this architecture in a production environment, I would make the following improvements.

## Security Hardening

* Restrict PostgreSQL firewall rules
* Restrict etcd ports to cluster members
* Restrict Patroni REST API access
* Enable TLS where appropriate
* Enable TLS for etcd peer/client communication
* Avoid `0.0.0.0/0` PostgreSQL access
* Rotate database credentials
* Use a dedicated secrets manager where appropriate
* Disable unnecessary services and ports

---

## PostgreSQL Tuning

Production tuning should be based on workload and available resources.

Potential parameters to tune include:

```text
shared_buffers
effective_cache_size
work_mem
maintenance_work_mem
checkpoint_timeout
max_wal_size
min_wal_size
random_page_cost
effective_io_concurrency
```

Connection limits should also be determined based on application workload rather than using unnecessarily large values.

---

## Backup and Recovery

A production deployment should implement a dedicated physical backup solution such as:

```text
pgBackRest
```

or:

```text
WAL-G
```

The backup strategy should include:

* Full physical backups
* Continuous WAL archiving
* Point-in-Time Recovery (PITR)
* Backup encryption
* Backup retention
* Remote/off-site storage
* Regular restore testing

The existence of a backup should not be considered sufficient until restoration has been tested.

---

# Monitoring and Observability

A production deployment should include PostgreSQL monitoring.

A recommended architecture is:

```text
             PostgreSQL
                  |
                  v
         postgres_exporter
                  |
                  v
             Prometheus
                  |
                  v
              Grafana
```

Important metrics include:

* PostgreSQL availability
* Replication lag
* WAL generation
* WAL retention
* Active connections
* Connection utilization
* Transaction rate
* Cache hit ratio
* Deadlocks
* Long-running queries
* Database size
* Replication slot status
* Checkpoints
* PostgreSQL errors
* Node CPU
* Node memory
* Disk utilization
* Disk latency

Patroni and etcd should also be monitored.

---

# Centralized Logging

For production, logs from the following components should be centralized:

```text
PostgreSQL
Patroni
etcd
HAProxy
Keepalived
```

A centralized logging platform such as ELK/EFK or Loki can be used.

---

# Automated Validation and Testing

The repository can be extended with automated infrastructure tests.

Recommended checks include:

```text
Ansible syntax validation
        |
        v
Ansible Lint
        |
        v
Idempotency test
        |
        v
Cluster deployment
        |
        v
Replication validation
        |
        v
Failover validation
        |
        v
HAProxy validation
        |
        v
Keepalived VIP validation
```

Useful commands include:

```bash
ansible-playbook \
  -i inventory.ini \
  install.yml \
  --syntax-check
```

and:

```bash
ansible-lint .
```

Molecule or another integration-testing framework could be introduced to automatically create and validate the complete environment.

---

# CI/CD

For a production repository, infrastructure changes should be validated through CI/CD.

A recommended pipeline would be:

```text
Git Push
   |
   v
Syntax Check
   |
   v
Ansible Lint
   |
   v
Security / Secret Scan
   |
   v
Integration Tests
   |
   v
Deployment Test
   |
   v
HA / Failover Tests
```

This prevents invalid Ansible changes from reaching the deployment environment.

---

# Operational Commands

## Patroni

```bash
patronictl -c /etc/patroni/config.yml list
```

```bash
patronictl -c /etc/patroni/config.yml topology
```

## Patroni Logs

```bash
journalctl -u patroni -f
```

## etcd

```bash
etcdctl member list
```

```bash
etcdctl endpoint health --cluster
```

```bash
etcdctl endpoint status --cluster
```

## HAProxy

```bash
systemctl status haproxy
```

## Keepalived

```bash
systemctl status keepalived
```

## VIP

```bash
ip addr
```

---

# Future Improvements

The following improvements are intentionally left as future production enhancements:

* PostgreSQL monitoring with Prometheus and Grafana
* Automated backup using pgBackRest or WAL-G
* Point-in-Time Recovery testing
* TLS for PostgreSQL and etcd
* Centralized logging
* PgBouncer for connection pooling
* Automated failover tests
* Molecule-based Ansible testing
* CI/CD pipeline
* Automated security scanning
* Remote WAL/object-storage backend
* Disaster Recovery environment
* Infrastructure metrics and alerting
* Capacity planning and workload-based PostgreSQL tuning

---

# License

This project is provided under the MIT License.

See the `LICENSE` file for details.
