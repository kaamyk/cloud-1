# cloud-1
  > Deploy a website and the necessary docker infrastructure in parallel on one or more instances provided by a cloud provider. WordPress + MySQL + phpMyAdmin behind nginx (HTTPS), in Docker containers, configured with Ansible.  
## Quick start
  1. Clone the repo:
      ```sh
      git clone https://github.com/kaamyk/cloud-1.git
      cd cloud-1
      ```
  1. Type the IP(s) of the instance(s) in **inventory.yml**.  
      > Note: if a new group is added in the inventory, a new corresponding directory in **group_vars/** is required containing an encrypted **secret.yml** file.  
      Example: **group_vars/\<group\>/secret.yml** (before encryption)
      ```yaml
      ansible_user: ubuntu
      ```
  1. Add the **.pem file(s)** with the right restrictions from aws instance(s) to the ssh agent:  
      ```sh
      chmod 400 /path/to/file.pem
      ssh-add /path/to/file.pem
      ```
  1. Create the vault password:  
      > The content of the file will be the en/decryption key for **secret.yml** files.  
      ```sh
      echo '<Thepassword>' > .vault_pass
      chmod 600 .vault_pass
      ```
  1. Edit the **./group_vars/\*/secret.yml** files with the variables for : **MySQL**, **WordPress** and more specific values.  
      ```sh
      ansible-vault edit group_vars/all/secret.yml
      ```
      > The list of the required variables is in [group_vars/all/secret.yml.example](group_vars/all/secret.yml.example) (fake values, not encrypted).  
      > Tips: Create a **clear.yml** file *ignored by Git* that contains the values in clear. Use the following command to encrypt the file. It makes the variables easier to follow during development.  
      ```sh
      ansible-vault encrypt --output group_vars/all/secret.yml group_vars/all/clear.yml
      ```
  1. Install the required collections:
      ```sh
      ansible-galaxy collection install -r requirements.yml
      ```
  1. Run the playbook:
      ```sh
      ansible-playbook playbook.yml 
      ```
  1. Check it works by opening `https://<IPAddress>/` and *accept the self-signed warning*.  
  
## Prerequisites
  ### 1. On the local machine
- Ansible (tested with `ansible-core 2.20.7`)
- Git
- Python 3
- SSH (client and agent)
- The two ansible collections (installed in Quick Start's step 6)
- Nothing else: Docker runs in the instances not locally.
  ### 2. On the servers
Tested on **Ubuntu 22.04 LTS (Jammy), amd64** (AWS image `ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*`)
- **Ubuntu** : hard coded in [roles/docker/tasks/main.yml:24](roles/docker/tasks/main.yml#L24)
- **Architecture** : arm64 or amd64: both handled in [roles/docker/tasks/main.yml line 23](roles/docker/tasks/main.yml#L23)
- **User with sudo** : The [playbook.yml line 3](playbook.yml#L3) uses "become: true"
- **Internet access** : download apt packages, docker images
- **Size** : deployed on t3.micro and t3.small. A swap file is created in [roles/system/tasks/main.yml line 11](roles/system/tasks/main.yml#L11)
  ### 3. On the Cloud provider
- An **account**
- **Running instances** with a **key pair** whose **.pem** file you have
- A **security group** allowing inbound 22, 80 and 443. Needed even with ufw, security group filters the traffic first.
- The **public IP** of the instance
## Configuration
  ### Inventory
**inventory.yml** lists the servers to deploy to: **groups** → **hosts** → **ansible_host** (the public IP).
```yaml
all:
  children:
    cloud1:            # group: its variables are in group_vars/cloud1/
      hosts:
        ubuntu:        # host name: only a label shown in Ansible's output
          ansible_host: <IP of instance 1>
    cloud221:
      hosts:
        cloud:
          ansible_host: <IP of instance 2>
```
- **One group per instance**: each group can have its own variables (e.g. a different SSH user if the instances use different images).
- **Add a server to an existing group**: add a host under its **hosts:**, nothing else.
- **Add a new group**: also create **group_vars/\<group\>/secret.yml** with **ansible_user** (see [SSH access](#ssh-access)).
- The public IP of an AWS instance **changes after a stop/start** (unless an Elastic IP is used): update **ansible_host** and run the playbook again.

  ### SSH access
- **User**: **ansible_user**, set per group in **group_vars/\<group\>/secret.yml** (**ubuntu** on AWS Ubuntu images).
- **Key**: loaded in **ssh-agent** (Quick start step 3). No key path is stored in the repository, so it stays portable and reveals nothing about the local machine. Alternatives:
  - on the command line: `ansible-playbook playbook.yml --private-key /path/to/file.pem`
  - in **~/.ssh/config**: a `Host <IP>` entry with an `IdentityFile /path/to/file.pem` line
- **Host keys**: **ansible.cfg** uses `StrictHostKeyChecking=accept-new`. A new server is trusted automatically on the first connection, but SSH refuses a known IP that presents a different key (e.g. after an instance reset). Remove the old key with `ssh-keygen -R <IP>`.
- After the first deployment, the **firewall** role disables SSH password login: only the key works.

  ### Secrets (Ansible Vault)
**Where the variables are**

| File | Applies to | Contains |
|---|---|---|
| **group_vars/all/secret.yml** | all servers | application directory, MySQL, WordPress |
| **group_vars/\<group\>/secret.yml** | the servers of one group | **ansible_user** |

Both files are encrypted with Ansible Vault. The list of keys, with fake values, is in [group_vars/all/secret.yml.example](group_vars/all/secret.yml.example).

**What each variable does**

| Variable | Role |
|---|---|
| **app_dir** | Directory on the server holding the stack (compose file, nginx config, **.env**, certificates) |
| **vault_mysql_root_password** | MySQL root password |
| **vault_mysql_database** | Database used by WordPress |
| **vault_mysql_user** / **vault_mysql_user_password** | MySQL user used by WordPress and phpMyAdmin |
| **wp_url** | Site URL. `https://{{ ansible_host }}`: built from the inventory, so it follows IP changes |
| **wp_title** | Site title |
| **wp_boss** / **wp_boss_password** / **wp_boss_mail** | WordPress administrator account |

**How the vault works**
- **.vault_pass** contains the vault password. It is ignored by Git and **ansible.cfg** points to it (`vault_password_file`), so no `--vault-password-file` option is needed.
- Useful commands:
  ```sh
  ansible-vault view   group_vars/all/secret.yml   # read
  ansible-vault edit   group_vars/all/secret.yml   # modify
  ansible-vault encrypt group_vars/all/secret.yml  # encrypt a clear file
  ansible-vault rekey  group_vars/*/secret.yml     # change the vault password
  ```
- Development workflow: keep the values in a **clear.yml** (ignored by Git) next to **secret.yml** and encrypt it with the command of Quick start step 5.

**On the server**, the secrets are written only to **app_dir/.env** (mode **0600**, readable by root only) and read by the containers. The tasks that handle them use `no_log: true`, so they never appear in the Ansible output.

> **Warning**: the MySQL variables are only read when the database volume is **empty** (first deployment). Changing them in the vault later has no effect on the existing database: change them in MySQL too (`ALTER USER ...`) or delete the **db_data** volume (this deletes the data). The same goes for the WordPress administrator: the account is created on the first installation only, so a later password change must be done in WordPress itself.

## Deploy
  ### The command
```sh
ansible-playbook playbook.yml
```
The playbook ([playbook.yml](playbook.yml)) applies 6 roles, in this order, to every server of the inventory:

| # | Role | What it does on the server |
|---|---|---|
| 1 | **system** | Waits for cloud-init to finish, updates the apt cache, creates a 2 GB swap file |
| 2 | **firewall** | Hardens SSH (key only), installs ufw and opens only 22, 80 and 443 |
| 3 | **docker** | Installs Docker and the Compose plugin from Docker's official repository |
| 4 | **tls** | Creates the self-signed certificate, or renews it if it expires within 30 days or if the IP changed |
| 5 | **compose** | Writes the compose file, nginx config and **.env** to **app_dir**, starts the containers and waits until they are healthy |
| 6 | **wordpress** | Installs WordPress with wp-cli (first run only), deletes the sample content, updates the site URL if the IP changed |

**Why this order**: each role needs the previous ones. The apt cache must be ready before installing packages, the firewall is set before any service is exposed, Docker must exist before starting containers, the certificate must exist before nginx starts, and wp-cli needs the running database.

  ### Parallel deployment
- One run deploys to **all the servers of the inventory at the same time**: Ansible runs each task on every host in parallel (up to 5 hosts at once by default, setting `forks`).
- To deploy to only one group or host (e.g. after an instance reset):
  ```sh
  ansible-playbook playbook.yml --limit cloud1
  ```

  ### Idempotence
The playbook can be run as many times as needed and always gives the same result.
- **First run** on a new server: many tasks are **changed**. At the end the handlers **Restart SSH** and **Restart nginx** run once to apply the new configuration.
- **Next runs**: nothing to do, every task is **ok** or **skipped**:
  ```
  PLAY RECAP *********************************************************************
  cloud    : ok=...  changed=0  unreachable=0  failed=0  skipped=...
  ubuntu   : ok=...  changed=0  unreachable=0  failed=0  skipped=...
  ```

  ### Useful options
| Option | Use |
|---|---|
| `--syntax-check` | Checks the playbook without connecting to any server |
| `--diff` | Shows the changes made to the files (configs, templates) |
| `--check` | Dry run: shows what would change without changing anything (see note below) |
| `-v` … `-vvv` | More details for debugging (see [Troubleshooting](#troubleshooting)). Tasks handling secrets stay hidden (`no_log`) |
| `--limit <group or host>` | Deploys to part of the inventory only |

> **Note**: `--check --diff` (dry run: shows what would change, changes nothing) works on an **already deployed** server: the read-only tasks (openssl, wp-cli, swap check) are marked `check_mode: false` so they still run. On a **new** server it is expected to fail: Docker is not really installed in check mode, so the tasks that use it cannot run.

## Verify
Replace `<IP>` with the public IP of an instance.

| Check | Command / action | Expected result |
|---|---|---|
| HTTP redirects to HTTPS | `curl -I http://<IP>/` | `301 Moved Permanently` to `https://<IP>/` |
| WordPress answers | `curl -kI https://<IP>/` (`-k`: accept the self-signed certificate) | `200 OK` |
| phpMyAdmin answers | open `https://<IP>/phpmyadmin/` | login page. Log in with **vault_mysql_user** / **vault_mysql_user_password** (root login is refused) |
| Only 22, 80, 443 open | `nmap -Pn -p- <IP>` (from another machine) | only `22/tcp`, `80/tcp`, `443/tcp` open |
| The database is not reachable | `nc -zv -w 3 <IP> 3306` | connection fails |
| All containers running | on the server: `cd <app_dir> && sudo docker compose ps` | **mysql**, **wordpress** `healthy`, **nginx**, **phpmyadmin** `Up` |
| Idempotence | run `ansible-playbook playbook.yml` again | `changed=0` for every host |
| Restart after reboot | `sudo reboot`, wait 1-2 min, reload the site | site back online, no playbook run needed |
| Data persists | create a post and upload an image, reboot, reload | post and image still there |

## Project structure
```
.
├── ansible.cfg                  # Ansible settings: inventory, roles path, vault password file, SSH options
├── inventory.yml                # Servers to deploy to (groups -> hosts -> IP)
├── playbook.yml                 # Entry point: applies the 6 roles to all hosts
├── requirements.yml             # Ansible collections needed (community.general, community.docker)
├── group_vars/
│   ├── all/
│   │   ├── secret.yml           # Encrypted: app directory, MySQL and WordPress variables
│   │   └── secret.yml.example   # Same keys with fake values (documentation)
│   └── <group>/secret.yml       # Encrypted: ansible_user of the group
└── roles/
    ├── system/                  # cloud-init wait, apt cache, swap file
    ├── firewall/
    │   ├── files/00-hardening.conf   # sshd drop-in: key only, no root password login
    │   ├── defaults/main.yml         # path of the sshd drop-in
    │   └── handlers/main.yml         # reload SSH when the config changes
    ├── docker/                  # Docker Engine + Compose plugin from Docker's apt repository
    ├── tls/                     # self-signed certificate (created / renewed when needed)
    ├── compose/
    │   ├── templates/
    │   │   ├── docker-compose.yml.j2        # the 5 services, network, volumes
    │   │   ├── nginx.conf.j2                # HTTP -> HTTPS, WordPress on /, phpMyAdmin on /phpmyadmin/
    │   │   ├── env.j2                       # .env with the database secrets
    │   │   └── pma.config.user.inc.php.j2   # phpMyAdmin: refuse root login
    │   └── handlers/main.yml    # restart nginx when its config or the certificate changes
    └── wordpress/               # wp-cli: install, delete sample content, keep the site URL up to date
```
Each role has one responsibility and can be read, tested and changed on its own.

## Design choices
  ### Architecture
```
 Internet ── 22 (SSH), 80 / 443 (HTTP/HTTPS) only
    │
 AWS Security Group ── ufw
    │
    ▼            private Docker network "inception"
  nginx :80 :443 ──── fastcgi :9000 ───► wordpress (php-fpm) ───► mysql :3306
    │                                                               ▲
    └──────────── http :80 (/phpmyadmin/) ──► phpmyadmin ───────────┘
```
- **1 process = 1 container**: nginx, WordPress (php-fpm), MySQL, phpMyAdmin, plus **wpcli**, a one-shot container used only by Ansible to install and configure WordPress (compose profile `cli`, never running permanently).
- The containers talk to each other by service name on the private network **inception**. Only **nginx** publishes ports (80 and 443): MySQL, php-fpm and phpMyAdmin cannot be reached from the internet.
- Official images only, with **pinned versions** (`mysql:8.4.13`, `wordpress:6.9.4-fpm`, `nginx:1.30.8-alpine`, `phpmyadmin:5.2.3-apache`, `wordpress:cli-2.12.0`): every deployment gets exactly the same software.

  ### Subject requirements
| Requirement | How it is met | Where |
|---|---|---|
| Fully automated deployment | One command, no manual step on the server | [playbook.yml](playbook.yml) |
| Restart after reboot | Docker service enabled at boot, `restart: always` on every service | [roles/docker](roles/docker/tasks/main.yml), [docker-compose.yml.j2](roles/compose/templates/docker-compose.yml.j2) |
| Data preserved after reboot | Named volumes **db_data** (database) and **wordpress** (files, uploads) | [docker-compose.yml.j2](roles/compose/templates/docker-compose.yml.j2) |
| Several servers in parallel | Every host of the inventory is deployed in the same run | [inventory.yml](inventory.yml) |
| Fresh Ubuntu 22.04 with SSH + Python | Waits for cloud-init, refreshes apt, installs everything else | [roles/system](roles/system/tasks/main.yml) |
| Separate containers that communicate | 1 service per process on the **inception** network | [docker-compose.yml.j2](roles/compose/templates/docker-compose.yml.j2) |
| Limited and secure public access | Only nginx publishes ports, SSH key only, phpMyAdmin root login refused | [nginx.conf.j2](roles/compose/templates/nginx.conf.j2), [roles/firewall](roles/firewall/tasks/main.yml) |
| docker-compose.yml | Rendered from a template into **app_dir** | [docker-compose.yml.j2](roles/compose/templates/docker-compose.yml.j2) |
| Database works with WordPress and phpMyAdmin | Same MySQL user for both, WordPress installed with wp-cli | [env.j2](roles/compose/templates/env.j2), [roles/wordpress](roles/wordpress/tasks/main.yml) |
| TLS when possible | HTTPS with a self-signed certificate, HTTP redirected | [roles/tls](roles/tls/tasks/main.yml), [nginx.conf.j2](roles/compose/templates/nginx.conf.j2) |
| Redirect to the right site depending on the URL | `/` → WordPress, `/phpmyadmin/` → phpMyAdmin | [nginx.conf.j2](roles/compose/templates/nginx.conf.j2) |
| Only ports 22, 80, 443 | ufw on the server + AWS Security Group | [roles/firewall](roles/firewall/tasks/main.yml) |
| Organized in roles | 6 roles, one responsibility each | [Project structure](#project-structure) |
| Portable, works on a fresh instance | No IP, key path or secret hard-coded; tested on newly created instances | [Configuration](#configuration) |
| Idempotent | Second run gives `changed=0` | [Idempotence](#idempotence) |
| No hard-coded secrets | All secrets in Ansible Vault, rendered only into **.env** (0600) | [Secrets](#secrets-ansible-vault) |

  ### Network and firewall
- Two layers: the **AWS Security Group** (outside the instance) and **ufw** (on the instance), both allowing only 22, 80 and 443.
- Docker writes its own iptables rules for published ports and **bypasses ufw**. This is why no container except nginx publishes a port: the protection does not rely on ufw alone.
- SSH: a drop-in file **/etc/ssh/sshd_config.d/00-hardening.conf** disables password login. It is validated with `sshd -t` before being installed, so a mistake cannot lock the server. Its name starts with `00-` because sshd keeps the first value it reads and the files are read in alphabetical order (before the cloud provider's `50-cloud-init.conf`).

  ### TLS
- The certificate is **self-signed** because the project has no domain name: a public authority (e.g. Let's Encrypt) only signs certificates for a domain it can verify. The traffic is encrypted all the same; the browser warns only because it does not know who signed it.
- The certificate contains the server IP (SAN). The **tls** role regenerates it when the IP changes or when it expires within 30 days, at the next playbook run.
- Only TLS 1.2 and 1.3 are accepted. The private key is readable by root only (0600).
- Next steps with a domain name (e.g. DuckDNS): a Let's Encrypt certificate and automatic renewal (cron job or systemd timer).

  ### Restart and persistence
- `restart: always` on every service + Docker enabled at boot: after a reboot, the stack starts again without Ansible.
- **Healthchecks** + `depends_on: condition: service_healthy`: MySQL must answer before WordPress starts, and WordPress must have written its files before nginx and wp-cli use them. This avoids race conditions on the first start.
- A **2 GB swap file** (with low swappiness) keeps MySQL from being killed for lack of memory on 1 GB instances.

  ### How idempotence is achieved
How the tasks avoid doing the same work twice:
- Ansible modules that compare the wanted state with the current one (`apt`, `copy`, `template`, `ufw`, `docker_compose_v2`...).
- `command` tasks guarded by `creates:` or `when:` (swap, certificate, WordPress install).
- Read-only checks marked `changed_when: false` (e.g. `wp core is-installed`, reading the certificate).
- **Handlers**: nginx and SSH are restarted only when their configuration actually changed.

## Operations
  ### Start / stop the instances (cost)
The instances are real resources: they must run only when needed.
- **Before the defense**: start the instances in the AWS console (EC2 → Instances → Start). If the IPs changed, update **inventory.yml** and run the playbook (see below).
- **After the defense / after working**: stop them (EC2 → Instances → Stop). A stopped instance is not billed for computing time; only its disk is (within the free tier limits).
- Instance sizes: **t3.micro** / **t3.small**, eligible for the free tier.

  ### The public IP changed
After a stop/start, AWS gives a new public IP (unless an Elastic IP is used).
1. Update **ansible_host** in [inventory.yml](inventory.yml).
2. Run `ansible-playbook playbook.yml`. The **tls** role creates a certificate for the new IP and the **wordpress** role updates the site URL.

  ### Reset an instance to its initial state
EC2 → Instance → Actions → Monitor and troubleshoot → **Replace root volume** (restores the launch image; all data is lost). Then:
```sh
ssh-keygen -R <IP>             # the server has a new host key
ansible-playbook playbook.yml --limit <group>
```

  ### Useful commands on the server
```sh
cd <app_dir>
sudo docker compose ps                     # state of the containers
sudo docker compose logs -f <service>      # logs (mysql, wordpress, nginx, phpmyadmin)
sudo docker compose run --rm wpcli wp <command>   # run a wp-cli command (e.g. wp user list)
```

## Troubleshooting
| Symptom | Cause | Fix |
|---|---|---|
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | The instance was reset or the IP was reused: new host key | `ssh-keygen -R <IP>` |
| `UNREACHABLE` / SSH timeout | The public IP changed after a stop/start | Update **inventory.yml** |
| `Permission denied (publickey)` | Key not loaded or wrong user | `ssh-add /path/to/file.pem`, check **ansible_user** |
| `The vault password file ... was not found` | **.vault_pass** missing | Quick start step 4 |
| `couldn't resolve module/action 'community.docker...'` | Collections not installed | `ansible-galaxy collection install -r requirements.yml` |
| SSH works but the site times out | Security Group does not allow 80/443 | Add inbound rules 80 and 443 in AWS |
| Browser warning "not secure" | Self-signed certificate | Expected: accept the warning (see [TLS](#tls)) |
| A task fails with `the output has been hidden due to the fact that 'no_log: true' was specified` | The output is hidden because it contains secrets | Look at the containers instead: `sudo docker compose logs mysql wordpress` |
| WordPress install / database access fails after changing a password in the vault | MySQL only reads its variables when its volume is empty | `ALTER USER` in MySQL, or delete the **db_data** volume (data lost) |

## AI usage
An AI assistant (Claude) was used during this project:
- **Review**: checking the project against the subject point by point, and reviewing my roles (bugs, idempotence, security).
- **Debugging**: explaining errors met while deploying (SSH hardening, WordPress install race on a fresh volume, MySQL password on an existing volume, apt cache on fresh instances).
- **Explanations**: Ansible and Docker concepts (handlers, check mode, vault, sshd drop-ins, Docker and ufw).

The roles were written by me; every AI suggestion was read, understood and tested on the instances before being kept.
