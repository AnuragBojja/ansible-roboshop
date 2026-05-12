# RoboShop – Ansible Automated Microservices Deployment

A fully idempotent, Ansible-based deployment suite for the **RoboShop** e-commerce platform. This project replaces shell script automation with declarative Ansible playbooks, leveraging certified Ansible collections (`amazon.aws`, `community.mysql`, `community.rabbitmq`, `community.general`) to manage a 10-service polyglot microservices architecture on AWS EC2.

---

## Why Ansible Over Shell Scripts

| Concern | Shell Scripts | Ansible Playbooks |
|---|---|---|
| Idempotency | Manual `if` checks | Built-in by design |
| Readability | Procedural, dense | Declarative YAML tasks |
| Error handling | Custom `VALIDATOR()` | Native task failure handling |
| AWS integration | AWS CLI raw calls | `amazon.aws` certified collection |
| DB operations | Raw `mysql` CLI commands | `community.mysql.mysql_db` module |
| Message broker setup | `rabbitmqctl` shell commands | `community.rabbitmq.rabbitmq_user` module |
| Inventory management | Hardcoded IPs in scripts | `inventory.ini` with groups & vars |

---

## Architecture Overview

```
Control Node (Ansible)
        │
        ├── 01-ec2-instance.yaml  →  Provisions EC2 + Route53 (amazon.aws)
        │
        └── inventory.ini  →  Groups all service hosts by role
                │
                ├── mongodb.yaml      →  mongodb.anuragaws.shop
                ├── redis.yaml        →  redis.anuragaws.shop
                ├── mysql.yaml        →  mysql.anuragaws.shop
                ├── rabbitmq.yaml     →  rabbitmq.anuragaws.shop
                ├── catalouge.yaml    →  catalouge.anuragaws.shop
                ├── user.yaml         →  user.anuragaws.shop
                ├── cart.yaml         →  cart.anuragaws.shop
                ├── shipping.yaml     →  shipping.anuragaws.shop
                ├── payment.yaml      →  payment.anuragaws.shop
                └── frontend.yaml     →  frontend.anuragaws.shop (public)
```

---

## Project Structure

```
roboshop-ansible/
├── 01-ec2-instance.yaml    # EC2 provisioning + Route53 DNS via amazon.aws
├── inventory.ini           # Host groups with ansible_user/password vars
├── mongodb.yaml            # MongoDB install, config, remote access
├── redis.yaml              # Redis 7 install, remote access, protected-mode off
├── mysql.yaml              # MySQL Server install + secure root setup
├── rabbitmq.yaml           # RabbitMQ install + user/permissions via collection
├── catalouge.yaml          # Node.js catalogue + MongoDB data seeding
├── user.yaml               # Node.js user service
├── cart.yaml               # Node.js cart service
├── shipping.yaml           # Java/Maven shipping + MySQL schema import
├── payment.yaml            # Python payment service + pip dependencies
├── frontend.yaml           # Nginx 1.24 frontend deployment
├── nginx.conf              # Nginx reverse proxy config (copied by frontend.yaml)
├── mongo.repo              # MongoDB yum repo (copied by mongodb/catalogue)
├── rabbitmq.repo           # RabbitMQ yum repo (copied by rabbitmq.yaml)
├── catalogue.service       # systemd unit for catalogue
├── cart.service            # systemd unit for cart
├── user.service            # systemd unit for user
├── shipping.service        # systemd unit for shipping
└── payment.service         # systemd unit for payment
```

---

## Ansible Collections Used

| Collection | Used For |
|---|---|
| `ansible.builtin` | dnf, copy, file, get_url, unarchive, service, systemd_service, replace, lineinfile, user, command, shell, pip |
| `amazon.aws` | `ec2_instance` (provision), `route53` (DNS A-records) |
| `community.mysql` | `mysql_db` (schema/data import) |
| `community.rabbitmq` | `rabbitmq_user` (user + vhost permissions) |
| `community.general` | `npm` (Node.js dependency install) |

---

## Services & Technology Stack

| Service    | Runtime      | Database       | Ansible Modules Used |
|------------|--------------|----------------|----------------------|
| Frontend   | Nginx 1.24   | —              | dnf, file, get_url, unarchive, copy, service |
| Catalogue  | Node.js 20   | MongoDB        | dnf, npm, copy, mongosh, systemd_service |
| User       | Node.js 20   | MongoDB, Redis | dnf, npm, copy, systemd_service |
| Cart       | Node.js 20   | Redis          | dnf, npm, copy, systemd_service |
| Shipping   | Java (Maven) | MySQL          | dnf, pip, command, mysql_db, systemd_service |
| Payment    | Python 3     | RabbitMQ       | dnf, pip, get_url, unarchive, systemd_service |
| MongoDB    | MongoDB Org  | —              | copy, dnf, service, replace |
| Redis      | Redis 7      | —              | command, dnf, replace, lineinfile, service |
| MySQL      | MySQL Server | —              | package, service, command |
| RabbitMQ   | RabbitMQ     | —              | copy, dnf, service, rabbitmq_user |

---

## Key Features

- **Fully declarative** — all infrastructure and application state expressed as YAML; no procedural shell logic
- **Native idempotency** — Ansible modules handle state checking; re-running playbooks is safe with no side effects
- **AWS EC2 provisioning** — `01-ec2-instance.yaml` uses `amazon.aws.ec2_instance` to spin up instances and `amazon.aws.route53` to register both public (frontend) and private (backend) DNS A-records
- **Certified collection modules** — `community.mysql.mysql_db` for SQL schema imports, `community.rabbitmq.rabbitmq_user` for broker user/vhost setup, `community.general.npm` for Node.js dependency management
- **Config file management** — `ansible.builtin.replace` and `ansible.builtin.lineinfile` for in-place config edits (Redis bind address, protected-mode; MongoDB bind IP)
- **Dynamic inventory groups** — `inventory.ini` organizes all hosts by role with shared `ansible_user` and `ansible_password` vars under `[all:vars]`
- **Conditional data seeding** — catalogue playbook registers MongoDB DB index output and conditionally loads master data only if the database doesn't exist
- **Loop-based installs** — multi-package installs and multi-file SQL imports use `loop:` to keep playbooks concise

---

## Prerequisites

- Ansible control node with Python 3.9+
- Ansible collections installed:
  ```bash
  ansible-galaxy collection install amazon.aws community.mysql community.rabbitmq community.general
  ```
- AWS credentials configured (`~/.aws/credentials` or IAM role on control node)
- SSH access to EC2 instances (key or password per `inventory.ini`)

---

## Usage

### Step 1 — Provision EC2 Instances
```bash
ansible-playbook -i inventory.ini 01-ec2-instance.yaml \
  -e "instances=['mongodb','redis','mysql','rabbitmq','catalogue','user','cart','shipping','payment','frontend']"
```

### Step 2 — Deploy Infrastructure Services
```bash
ansible-playbook -i inventory.ini mongodb.yaml
ansible-playbook -i inventory.ini redis.yaml
ansible-playbook -i inventory.ini mysql.yaml
ansible-playbook -i inventory.ini rabbitmq.yaml
```

### Step 3 — Deploy Application Services
```bash
ansible-playbook -i inventory.ini catalouge.yaml
ansible-playbook -i inventory.ini user.yaml
ansible-playbook -i inventory.ini cart.yaml
ansible-playbook -i inventory.ini shipping.yaml
ansible-playbook -i inventory.ini payment.yaml
ansible-playbook -i inventory.ini frontend.yaml
```

---

## Inventory Structure

```ini
[mongodb]          mongodb.anuragaws.shop
[redis]            redis.anuragaws.shop
[mysql]            mysql.anuragaws.shop
[rabbitmq]         rabbitmq.anuragaws.shop
[cart]             cart.anuragaws.shop
[catalouge]        catalouge.anuragaws.shop
[user]             user.anuragaws.shop
[shipping]         shipping.anuragaws.shop
[payment]          payment.anuragaws.shop
[frontend]         frontend.anuragaws.shop
[local]            localhost

[all:vars]
ansible_user=ec2-user
ansible_password=DevOps321
```

---

## DNS Configuration

| Service  | Record | IP Type |
|---|---|---|
| Frontend | `anuragaws.shop` | Public |
| All others | `<service>.anuragaws.shop` | Private |

---

## Author

**Anurag Bojja**  
Milwaukee, WI | [LinkedIn](https://www.linkedin.com/in/anurag-bojja-81a405192/) | [GitHub](https://github.com/AnuragBojja)
