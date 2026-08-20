# proton — Multi-Tier Java App: Local VM Provisioning with Vagrant + Ansible

> Full-stack Java application with fully automated local environment provisioning — Vagrant VMs, Ansible configuration management, zero-manual-step setup from `vagrant up` to running application.

A complete local development and deployment reference for a multi-tier Java application. Vagrant provisions five VMs (db, cache, queue, search, app); Ansible configures each service automatically — mirroring a production topology on a local machine without Docker.

---

## What this solves

Running a multi-tier application locally typically means either managing five separate processes manually or flattening everything into Docker Compose (which hides real network topology). This repo provisions five dedicated VMs that communicate over a real private network — the same way they would in production.

**Result:** `vagrant up` → application running at `http://localhost:8080/vprofile` with all dependencies configured.

---

## Application stack

| Component | Technology |
|---|---|
| Web framework | Spring MVC · Spring Security · Spring Data JPA |
| Runtime | JDK 17 or 21 · Tomcat |
| Build | Maven 3.9 |
| Database | MySQL 8 |
| Cache | Memcached |
| Message queue | RabbitMQ |
| Search | ElasticSearch |
| Frontend | JSP |

---

## Architecture

```text
vagrant up
    │
    ├──► VM: db01      (MySQL 8)         ← Ansible: db.yml
    ├──► VM: mc01      (Memcached)       ← Ansible: memcache.yml
    ├──► VM: rmq01     (RabbitMQ)        ← Ansible: rabbitmq.yml
    ├──► VM: search01  (ElasticSearch)   ← Ansible: elasticsearch.yml
    └──► VM: web01     (Tomcat + app)    ← Ansible: tomcat.yml + deploy.yml

Each VM has a static private IP.
web01 reads db01/mc01/rmq01/search01 hostnames from application.properties.
```

Ansible playbooks are idempotent — re-running `vagrant provision` produces the same result.

---

## Repository structure

```text
proton/
├── ansible/
│   ├── playbooks/           # Service-specific configuration playbooks
│   ├── roles/               # Reusable Ansible roles
│   └── ansible.cfg          # Inventory and connection settings
├── vagrant/
│   └── Vagrantfile          # Multi-VM topology (5 VMs)
├── src/                     # Spring Boot Java source
│   └── main/resources/
│       └── db_backup.sql    # Schema + seed data (auto-imported by Ansible)
├── Jenkinsfile              # CI/CD pipeline definition
└── pom.xml                  # Maven build
```

---

## Quick start

### Prerequisites

```bash
# Install these before proceeding:
VirtualBox  >= 6.1   # https://www.virtualbox.org/
Vagrant     >= 2.3   # https://www.vagrantup.com/
Ansible     >= 2.9   # pip install ansible (macOS/Linux) or WSL on Windows
JDK 17 or 21
Maven 3.9
```

### Provision (first run ~15 min, downloads base boxes + packages)

```bash
git clone https://github.com/emman2582/proton.git
cd proton/vagrant

vagrant up
# Vagrant provisions all 5 VMs and runs Ansible automatically
```

### Access

```
http://localhost:8080/vprofile
```

### Manage VMs

```bash
vagrant status             # VM status
vagrant ssh web01          # SSH into app server
vagrant halt               # Stop all VMs
vagrant destroy -f         # Remove all VMs (data lost)
vagrant provision          # Re-run Ansible without reprovisioning VMs
```

---

## Local build (without VMs)

```bash
# Requires: MySQL running locally, db_backup.sql imported
mysql -u <user> -p accounts < src/main/resources/db_backup.sql

cd proton
mvn clean package
# Deploy target/vprofile.war to local Tomcat
```

---

## Jenkins pipeline

`Jenkinsfile` implements a declarative pipeline:

| Stage | Action |
|---|---|
| Fetch code | Checkout from SCM |
| Unit tests | `mvn clean test` |
| Code style | `mvn checkstyle:checkstyle` |
| Static analysis | SonarQube scan |
| Build | `mvn clean package` (WAR) |
| Deploy | Transfer and deploy WAR to staging |

---

## Technical decisions

**Vagrant + VMs over Docker Compose:** Each service runs on a separate VM to accurately simulate production network topology. Service-to-service communication uses real IP addressing, not Docker bridge networking — this surfaces real configuration issues earlier.

**Ansible for configuration management:** Idempotent configuration ensures consistent environments regardless of host state. The same playbooks can be adapted for cloud provisioning (EC2 + Ansible) without rewriting.

**db_backup.sql imported at provision time:** Schema and seed data are applied automatically during `vagrant provision` — no manual database setup step, no "it works on my machine" database state issues.

**Static private IPs in Vagrantfile:** Hard-coded IPs (e.g., `192.168.56.x`) allow `application.properties` to reference hostnames without a local DNS server. Matches the approach used in AWS VPC with private hosted zones.

---

## Security considerations

- Default credentials (MySQL root, RabbitMQ guest) are for local development only — **change all passwords before any network-accessible deployment**
- `db_backup.sql` contains schema and seed data only — no production data
- Use Ansible Vault for secrets when adapting playbooks for team or CI environments
- Review `ansible.cfg` `host_key_checking = False` — acceptable for local VMs, not for remote hosts

---

## Systems Design

### Multi-Tier Architecture with VM Isolation
```text
web01   (Tomcat + app)    ← application tier
db01    (MySQL 8)         ← persistence tier
mc01    (Memcached)       ← cache tier
rmq01   (RabbitMQ)        ← messaging tier
search01 (ElasticSearch)  ← search tier
```
Each service runs on a dedicated VM with a static private IP, mirroring production topology more accurately than single-host Docker Compose. Network latency, DNS resolution, and firewall rules behave like real infrastructure.

### Configuration Management as Desired State
Ansible playbooks describe desired state, not a sequence of commands. Running a playbook on an already-configured machine is safe — Ansible checks before changing (idempotency). This is the same principle as Terraform for infrastructure, applied at the OS/service layer.

### Cache-Aside Pattern (Memcached)
```text
Request → check Memcached → HIT: return cached data
                           → MISS: query MySQL → populate cache → return data
```
TTL-based expiry manages cache invalidation without explicit purge logic.

### Asynchronous Decoupling (RabbitMQ)
Operations that don't require immediate results are published to RabbitMQ and consumed asynchronously. The HTTP thread returns immediately, improving perceived application responsiveness.

### VM-to-Cloud Portability
Ansible playbooks are cloud-agnostic. The same playbooks used to configure local Vagrant VMs can be applied to EC2 instances with no changes to the task definitions — only the inventory changes. This is the stepping stone pattern from local development to cloud production.

---

## Work with me

I help teams automate local development environments and implement Ansible-based configuration management that scales from developer laptop to cloud production.

**Typical engagements:**
- Vagrant + Ansible local environment automation
- Ansible playbook design for multi-tier applications
- VM-to-cloud IaC migration (Vagrant → Terraform + Ansible)
- Configuration management strategy and implementation

[LinkedIn](https://www.linkedin.com/in/emmanuelcomia/) · [Email](mailto:emman_job@yahoo.com)
