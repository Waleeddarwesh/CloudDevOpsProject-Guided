# 🤖 Module 03 — Ansible

**Time:** ~5 hours · **Cost:** no new resources

---

## 🎯 What you'll build

Terraform created a bare Ubuntu box. Ansible turns it into a CI server: Java, Jenkins, Docker, Trivy, AWS CLI, kubectl, Helm, SonarQube — via **9 roles**, **dynamic inventory**, and **Vault**.

**No IP address will appear anywhere in your code.**

---

## 📖 Why not just do this in `user_data`?

You could. You shouldn't:

| | `user_data` (Terraform) | Ansible |
|---|---|---|
| Runs | **once**, at first boot | any time |
| Fix a typo | **destroy + recreate the instance** — losing all Jenkins build history | edit, re-run, 30 seconds |
| Idempotent | no | yes |

That's why your `user_data.sh` does the absolute minimum (patches, Python, swap) and writes a marker file the playbook waits on.

> 💡 **Provisioning ≠ configuration.** Terraform owns the *machine*; Ansible owns the *software on it*. Keeping them separate is why both tools exist.

---

# Part 1 · Dynamic inventory

## 🔨 The problem with static inventory

```ini
[jenkins]
54.211.103.22
```

That IP is wrong the moment Terraform replaces the instance. Worse — it might now belong to *someone else's* server.

## The solution

Create `workspace/03-Ansible/inventory/aws_ec2.yml`:

```yaml
# ⚠️ THE FILENAME MATTERS. It must end in `aws_ec2.yml` or `aws_ec2.yaml`
# or the plugin refuses to load it, and the play reports
# "skipping: no hosts matched" with no explanation.

plugin: amazon.aws.aws_ec2

regions:
  - us-east-1

filters:
  # Set by Terraform's server module. The two must stay in sync —
  # change either string and Ansible finds nothing.
  tag:Role: jenkins

  # ⚠️ Without this, TERMINATED instances are still returned — AWS keeps
  # them visible in the API for ~1 hour — and the playbook wastes 30
  # seconds timing out against a machine that no longer exists.
  instance-state-name:
    - running

hostnames:
  - tag:Name        # readable output instead of ec2-54-x-x-x.compute-1...
  - dns-name

keyed_groups:
  # tag Role=jenkins  →  group `role_jenkins`  (what playbook.yml targets)
  - key: tags.Role
    prefix: role
    separator: "_"

compose:
  ansible_host: public_ip_address
  # ⚠️ NOTE THE DOUBLE QUOTING. These are evaluated as Jinja expressions,
  # so a literal string must be quoted twice. Writing "ubuntu" would be
  # treated as an undefined VARIABLE and silently resolve to nothing.
  ansible_user: "'ubuntu'"
  ansible_ssh_private_key_file: "'~/.ssh/ivolve-key.pem'"

cache: true
cache_plugin: jsonfile
cache_connection: ./.ansible_inventory_cache
cache_timeout: 300
```

> ⚠️ **Never put `aws_access_key` here.** This file is committed. The plugin resolves credentials through the normal boto3 chain (env vars → `~/.aws/credentials` → IAM role).

## `ansible.cfg`

```ini
[defaults]
inventory        = ./inventory
remote_user      = ubuntu
private_key_file = ~/.ssh/ivolve-key.pem

# EC2 instances are created fresh, so their host keys are never in
# known_hosts and a strict check blocks every first run.
host_key_checking = False

become = True

# The default 'minimal' callback prints an unreadable JSON blob per task.
stdout_callback   = yaml
callbacks_enabled = timer, profile_tasks

vault_password_file = ./.vault_pass

[inventory]
# ⚠️ WITHOUT THIS LINE the aws_ec2 plugin is silently ignored.
enable_plugins = amazon.aws.aws_ec2, ini, yaml

[ssh_connection]
# ControlPersist keeps ONE SSH connection open and multiplexes every task
# over it. Typically cuts playbook runtime by 30-50%.
ssh_args  = -o ControlMaster=auto -o ControlPersist=300s -o PreferredAuthentications=publickey

# Sends module code over the existing session instead of writing a temp
# file to the remote host first — one fewer round trip per task.
pipelining = True
```

## `requirements.yml`

```yaml
collections:
  - name: amazon.aws        # the aws_ec2 inventory plugin
    version: ">=9.0.0"
  - name: community.docker
    version: ">=4.0.0"
  - name: community.general
    version: ">=10.0.0"
  - name: ansible.posix     # sysctl
    version: ">=1.6.0"
```

```bash
cd workspace/03-Ansible
ansible-galaxy collection install -r requirements.yml
pip install boto3 botocore      # the plugin needs the Python SDK
```

## ✅ Test it before writing any playbook

```bash
ansible-inventory --graph
```

```text
@all:
  |--@role_jenkins:
  |  |--ivolve-dev-jenkins
```

```bash
ansible role_jenkins -m ping
```

```text
ivolve-dev-jenkins | SUCCESS => { "ping": "pong" }
```

> ⚠️ **Empty group?** The EC2 tag and your filter disagree. Check with:
> ```bash
> aws ec2 describe-instances --filters "Name=tag:Role,Values=jenkins" \
>   --query 'Reservations[].Instances[].[InstanceId,State.Name]'
> ```
> Stale cache? `ansible-inventory --graph --flush-cache`

---

# Part 2 · Ansible Vault

Secrets must not sit in a committed file.

```bash
cd workspace/03-Ansible

# 1 — the vault password (gitignored)
openssl rand -base64 32 > .vault_pass
chmod 600 .vault_pass

# 2 — the secrets file
mkdir -p group_vars/all
cat > group_vars/all/vault.yml <<'EOF'
---
vault_sonarqube_admin_password: "CHANGE_ME"
vault_github_username: "YourGitHubUser"
vault_github_token: "CHANGE_ME_pat"
EOF

# edit it, then encrypt
ansible-vault encrypt group_vars/all/vault.yml

# 3 — prove it
head -1 group_vars/all/vault.yml
```

```text
$ANSIBLE_VAULT;1.1;AES256
```

```bash
echo -e ".vault_pass\n*.retry\nansible.log\n.ansible_*" >> ../.gitignore
```

> ⚠️ **Store the vault password in a password manager.** Lose it and the file is unrecoverable — AES-256 with no reset.

### The `vault_` prefix convention

An encrypted file is opaque to `grep`. When a task references a bare `sonarqube_admin_password` you can't tell where it came from. So prefix every secret and indirect through a plain variable:

```yaml
# roles/sonarqube/defaults/main.yml
sonarqube_admin_password: "{{ vault_sonarqube_admin_password }}"
```

Daily use:

```bash
ansible-vault view group_vars/all/vault.yml    # read without decrypting to disk
ansible-vault edit group_vars/all/vault.yml    # edit, re-encrypts on save
ansible-vault rekey group_vars/all/vault.yml   # change password
```

---

# Part 3 · Roles

## Anatomy

```text
roles/docker/
├── tasks/main.yml       what to do
├── defaults/main.yml    overridable variables (LOWEST precedence)
├── handlers/main.yml    restart triggers — run ONCE, only if notified
├── templates/           Jinja2
└── meta/main.yml        dependencies
```

**Handlers are the idea worth understanding.** A handler runs only when notified, and only *once per play* no matter how many tasks notified it. Change five config files → restart the service once. Change nothing → restart nothing.

## Role: `common`

`roles/common/tasks/main.yml`:

```yaml
---
- name: Update the APT cache
  ansible.builtin.apt:
    update_cache: true
    # Only refresh if older than an hour — otherwise every role triggers a
    # full index download.
    cache_valid_time: 3600
  # Ansible cannot detect a "change" in a cache refresh, so without this it
  # reports changed=1 every run and breaks idempotency reporting.
  changed_when: false

- name: Install baseline packages
  ansible.builtin.apt:
    name:
      - apt-transport-https
      - ca-certificates
      - gnupg
      - curl
      - git
      - unzip
      - jq
      - python3-pip
      - python3-apt      # required by the ansible.builtin.apt module itself
    state: present
  register: apt_result
  retries: 3
  delay: 5
  until: apt_result is succeeded

- name: Create the APT keyring directory
  # Modern APT reads keys from /etc/apt/keyrings with each repo pinned to
  # its own key via `signed-by=`.
  #
  # ⚠️ The deprecated `apt_key` module wrote everything into one shared
  # trusted.gpg, where ANY trusted key could sign ANY repository — a real
  # supply-chain risk. That is why apt_key is removed in newer Ansible.
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    mode: "0755"

- name: Configure kernel parameters
  ansible.posix.sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-ivolve.conf
    reload: true
  loop:
    # Docker/containerd/Jenkins register thousands of inotify watches;
    # the 8192 default is exhausted quickly and surfaces as a misleading
    # "too many open files".
    - { name: fs.inotify.max_user_watches, value: "524288" }
    # SonarQube embeds Elasticsearch, which refuses to start below this.
    - { name: vm.max_map_count, value: "262144" }
```

## Role: `docker` — the APT repository pattern

`roles/docker/tasks/main.yml`:

```yaml
---
- name: Remove distribution Docker packages
  # Ubuntu's docker.io conflicts with the official packages and ships no
  # Compose v2 plugin — leaving a confusing half-working setup.
  ansible.builtin.apt:
    name: [docker.io, docker-compose, containerd, runc]
    state: absent
    purge: true

- name: Download the Docker GPG key
  ansible.builtin.get_url:
    url: https://download.docker.com/linux/ubuntu/gpg
    dest: /etc/apt/keyrings/docker.asc
    mode: "0644"
    force: false            # don't re-download every run → idempotency

- name: Convert the key to binary keyring format
  ansible.builtin.command:
    cmd: gpg --batch --yes --dearmor -o /etc/apt/keyrings/docker.gpg /etc/apt/keyrings/docker.asc
    # `creates` makes this idempotent with no shell test — Ansible skips
    # the task entirely when the file exists.
    creates: /etc/apt/keyrings/docker.gpg

- name: Ensure the keyring is world-readable
  # ⚠️ APT fetches indexes as the unprivileged _apt user and cannot read a
  # 0600 keyring. The symptom is a signature error that looks like a
  # corrupt key.
  ansible.builtin.file:
    path: /etc/apt/keyrings/docker.gpg
    mode: "0644"

- name: Add the Docker APT repository
  ansible.builtin.apt_repository:
    # signed-by binds this repo to that ONE key.
    # distribution_release resolves to jammy/noble automatically.
    repo: >-
      deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg]
      https://download.docker.com/linux/ubuntu
      {{ ansible_facts['distribution_release'] }} stable
    filename: docker
    state: present

- name: Install Docker Engine
  ansible.builtin.apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
    state: present
    update_cache: true
  notify: Restart docker

- name: Configure the Docker daemon
  ansible.builtin.copy:
    dest: /etc/docker/daemon.json
    mode: "0644"
    content: |
      {
        "log-driver": "json-file",
        "log-opts": { "max-size": "10m", "max-file": "3" }
      }
    # ⚠️ Reject invalid JSON BEFORE writing. A malformed daemon.json stops
    # dockerd starting at all, and recovering means SSH-ing in to hand-edit.
    validate: "python3 -c 'import json,sys; json.load(open(sys.argv[1]))' %s"
  notify: Restart docker

- name: Ensure Docker is running and enabled
  ansible.builtin.systemd:
    name: docker
    state: started
    enabled: true
```

> 💡 **Docker's default log driver has NO size limit.** A container logging in a loop fills the root disk and takes Jenkins down with it. Those four lines of `log-opts` prevent a whole class of 3am incident.

`roles/docker/handlers/main.yml`:

```yaml
---
- name: Restart docker
  ansible.builtin.systemd:
    name: docker
    state: restarted
    daemon_reload: true
```

## Role: `jenkins` — ordering matters

`roles/jenkins/meta/main.yml`:

```yaml
---
dependencies:
  # ⚠️ The Jenkins .deb declares NO JRE dependency. Install it without a
  # JVM and the service fails to start with a bare
  # "Failed with result 'exit-code'" and nothing useful in the journal.
  - role: java
  # The `docker` group must exist before the jenkins user joins it.
  - role: docker
```

> 💡 Ansible runs each role **once per play** even when several roles depend on it — no duplicate work.

Key tasks:

```yaml
- name: Wait for Jenkins to accept HTTP connections
  ansible.builtin.uri:
    url: "http://127.0.0.1:8080/login"
    # ⚠️ 403 is EXPECTED and healthy — it means Jenkins is up and enforcing
    # authentication. Treating it as failure is a very common mistake.
    status_code: [200, 403]
  register: jenkins_ready
  retries: 30
  delay: 10
  until: jenkins_ready.status in [200, 403]

- name: Add the jenkins user to the docker group
  ansible.builtin.user:
    name: jenkins
    groups: docker
    # ⚠️ `append` is essential. Without it this REPLACES the user's entire
    # supplementary group list.
    append: true
  notify: Restart jenkins
```

> ⚠️ **Docker group = root.** Any member can run `docker run -v /:/host --privileged` and own the machine. Acceptable for the CI user on a *dedicated* build server — and that's precisely why it's dedicated.

Systemd config goes in a **drop-in override**, never the packaged unit:

`roles/jenkins/templates/override.conf.j2` → `/etc/systemd/system/jenkins.service.d/override.conf`

```ini
[Service]
Environment="JAVA_OPTS=-Xms{{ jenkins_java_heap }} -Xmx{{ jenkins_java_heap }} -Djava.awt.headless=true"

# Jenkins + plugins keep thousands of files open. The systemd default of
# 1024 surfaces as "Too many open files" mid-build.
LimitNOFILE=65536

# First boot with a large plugin set exceeds systemd's 90s default, which
# would kill Jenkins mid-initialisation and corrupt the plugin state.
TimeoutStartSec=300
```

Editing `/lib/systemd/system/jenkins.service` directly works until the next `apt upgrade` silently overwrites it.

---

# Part 4 · The playbook

`workspace/03-Ansible/playbook.yml`:

```yaml
---
- name: Configure Jenkins CI server
  hosts: role_jenkins       # ← group from the dynamic inventory
  become: true
  any_errors_fatal: true

  pre_tasks:
    - name: Wait for SSH
      # Terraform returns as soon as EC2 reports "running" — well before
      # sshd is listening.
      ansible.builtin.wait_for_connection:
        delay: 5
        timeout: 300

    - name: Verify the target is Ubuntu 22.04+
      # Every role uses APT. Running against Amazon Linux would fail deep
      # inside a role with a confusing error; fail here with the reason.
      ansible.builtin.assert:
        that:
          - ansible_facts['distribution'] == 'Ubuntu'
          - ansible_facts['distribution_major_version'] is version('22', '>=')
        fail_msg: "This playbook targets Ubuntu 22.04+."

    - name: Wait for the Terraform bootstrap to finish
      # ⚠️ user_data and Ansible BOTH call apt-get. Overlapping produces
      # "Could not get lock /var/lib/dpkg/lock-frontend" — an intermittent
      # failure that is painful to diagnose.
      ansible.builtin.wait_for:
        path: /etc/ivolve-bootstrap-complete
        timeout: 600

  roles:
    - { role: common,    tags: [common] }
    - { role: java,      tags: [java] }
    - { role: docker,    tags: [docker] }
    - { role: jenkins,   tags: [jenkins] }
    - { role: trivy,     tags: [trivy] }
    - { role: aws_cli,   tags: [aws] }
    - { role: kubectl,   tags: [kubectl] }
    - { role: helm,      tags: [helm] }
    - { role: sonarqube, tags: [sonarqube] }

  post_tasks:
    - name: Read the Jenkins initial admin password
      # Jenkins deletes this file once the setup wizard completes, so a
      # missing file just means setup is already done.
      ansible.builtin.slurp:
        src: /var/lib/jenkins/secrets/initialAdminPassword
      register: jenkins_pw
      failed_when: false

    - name: Show the summary
      ansible.builtin.debug:
        msg: |
          Jenkins   http://{{ ansible_host }}:8080
          SonarQube http://{{ ansible_host }}:9000
          Password  {{ (jenkins_pw.content | b64decode | trim)
                        if jenkins_pw.content is defined
                        else 'already consumed' }}
```

---

# ✅ Checkpoint

```bash
cd workspace/03-Ansible

ansible-playbook playbook.yml --check --diff     # dry run first
ansible-playbook playbook.yml
```

**~10 minutes.** Save the admin password from the output.

## The real test — idempotency

```bash
ansible-playbook playbook.yml | tail -3
```

```text
PLAY RECAP ******************************************
ivolve-dev-jenkins : ok=48  changed=0  unreachable=0  failed=0
```

**`changed=0`.** That is the property that makes it safe to re-run the playbook to add a tool or repair drift.

> ⚠️ **Any `changed=N` on a second run is a bug in your role.** Usually a `command:` without `creates:`/`changed_when:`, or `get_url` without `force: false`. Find it and fix it — this is the discipline that separates working automation from reliable automation.

Verify the tools:

```bash
ansible role_jenkins -m shell -a "docker --version && trivy --version | head -1 && kubectl version --client && helm version --short"
```

---

## 🧠 Before you continue

1. Why does `user_data` do so little, and Ansible so much?
2. What does the tag `Role=jenkins` connect?
3. Why must the inventory file be named `aws_ec2.yml`?
4. Why `ansible_user: "'ubuntu'"` with two sets of quotes?
5. Why must `java` run before `jenkins`?
6. Why is HTTP 403 a *success* when waiting for Jenkins?
7. What does `changed=0` on the second run prove?
8. Why a systemd drop-in instead of editing the unit file?

---

<div align="center">

**Next → [Module 04 — Kubernetes](04-kubernetes.md)**

</div>
