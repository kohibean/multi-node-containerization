# Multi-Node Ansible Lab — Containerized

> Reproducible multi-node Ansible lab using Docker, with heterogeneous managed nodes
> (Ubuntu 22.04 + Rocky Linux 9) so playbooks have to handle real-world distro differences.

**Status:** 🚧 Work in progress. Dockerfiles are complete; `docker-compose.yml`,
inventory, and a sample playbook are next.

---

## Why this project

Most Ansible labs use a single distro and a fleet of identical VMs — heavy, slow,
and impossible to hand someone else as a working environment. This project rebuilds
the same kind of multi-node lab using Docker, with two goals:

1. **Reproducible from code.** Anyone with Docker can clone the repo and have an
   identical working lab in two minutes — no 20 GB VM image, no "works on my machine."
2. **Heterogeneous on purpose.** The managed nodes intentionally use both Ubuntu and
   Rocky Linux so playbooks have to handle the real-world differences between distro
   families (`apt` vs `dnf`, `sudo` vs `wheel`, SSH host-key auto-generation, etc.) —
   the same situation any production agency hits when supporting multiple clients.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Docker network: lab-network                │
│                                                             │
│   ┌────────────┐      ┌────────┐ ┌────────┐ ┌────────────┐  │
│   │  control   │─────▶│ node1  │ │ node2  │ │   node3    │  │
│   │ (Ansible)  │ ssh  │ Ubuntu │ │ Ubuntu │ │ Rocky 9    │  │
│   └────────────┘      └────────┘ └────────┘ └────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

- **control** — Ubuntu 22.04 + Ansible + SSH client + the private SSH key. This is
  where playbooks are run from.
- **node1, node2** — Ubuntu 22.04, SSH server, `ansible` user with passwordless sudo,
  authorized public key.
- **node3** — Rocky Linux 9, same managed-node setup, different package manager and
  privilege group (`wheel` instead of `sudo`).

All four containers share a custom Docker bridge network. Docker's built-in DNS
resolves service names (`node1`, `node2`, `node3`) to container IPs, so Ansible
inventory references nodes by hostname — no IP hardcoding.

---

## What's done

- ✅ `Dockerfile.node` — Ubuntu 22.04 managed node, key-based SSH, passwordless sudo
- ✅ `Dockerfile.node-rocky` — Rocky Linux 9 managed node, same pattern adapted for the
  RHEL family (`dnf`, `wheel` group, explicit `ssh-keygen -A` for host keys)
- ✅ `Dockerfile.control` — control node with Ansible and SSH client
- ✅ ed25519 SSH key pair generation (excluded from git)

## What's next

- ⬜ `docker-compose.yml` — wire all three services onto a shared `lab-network`
- ⬜ `inventory/hosts.yml` — Ansible inventory referencing nodes by service name
- ⬜ `ansible.cfg` — point Ansible at the private key, skip strict host key checking for the lab
- ⬜ Sample playbook proving end-to-end connectivity (e.g. install nginx across all nodes)
- ⬜ Healthchecks on node containers so `depends_on` waits for sshd readiness, not just container start

---

## Design decisions and tradeoffs

A few choices in here are deliberate lab simplifications, called out so reviewers know
what would change in production:

- **Passwordless sudo for the `ansible` user.** Lab convenience so playbooks using
  `become: true` don't hang on prompts. In production this would be replaced with
  tightly-scoped sudoers rules in `/etc/sudoers.d/` and credentials managed via
  Ansible Vault or a secrets manager.
- **`StrictHostKeyChecking=no` (planned in the Ansible config).** Skips the "trust this
  host?" prompt for ephemeral containers. In production, host keys would be properly
  managed via `ssh-keyscan` and a maintained `known_hosts` file.
- **Key-based SSH with an ed25519 key.** Production pattern, even in the lab. The
  private key is gitignored; the public key is what gets baked into managed-node images.
- **No `version:` field in the compose file.** Modern Docker Compose ignores the legacy
  version field, so it's intentionally omitted.
- **`depends_on` without health conditions.** For initial bring-up this is fine — by
  the time you `exec` into the control container and run a playbook, sshd is up. A
  later iteration will add `healthcheck` + `condition: service_healthy` for strict
  ordering.

---

## Repository layout (planned)

```
ansible-docker-lab/
├── Dockerfile.node           # Ubuntu managed node
├── Dockerfile.node-rocky     # Rocky managed node
├── Dockerfile.control        # control node with Ansible
├── docker-compose.yml        # (WIP)
├── ansible.cfg               # (WIP)
├── inventory/
│   └── hosts.yml             # (WIP)
├── playbooks/
│   └── site.yml              # (WIP) sample playbook
├── roles/                    # (WIP) reusable roles
├── keys/
│   └── ansible_key.pub       # public key (private key gitignored)
├── .gitignore
└── README.md
```

---

## Usage (target — once compose is in place)

```bash
# One-time setup: generate the SSH key pair
mkdir -p keys
ssh-keygen -t ed25519 -f keys/ansible_key -N ""

# Build and start the whole lab
docker compose up -d --build

# Jump into the control container
docker compose exec control bash

# From inside the control container, verify connectivity
ansible -i inventory/hosts.yml all -m ping

# Tear down when done
docker compose down
```

---

## Things I learned building this

A few non-obvious gotchas this project surfaced — worth noting because they're the
kind of distro/Docker quirks you only hit by doing the work:

- **`apt-get update` and `dnf install` aren't symmetric.** On Ubuntu the package list
  must be refreshed before install; on Rocky 9, `dnf install` handles that internally,
  so a separate update step is unnecessary (and undesirable in a Dockerfile, because
  `dnf update` upgrades all packages — slow and non-deterministic).
- **Rocky doesn't auto-generate SSH host keys** the way Ubuntu's `openssh-server`
  package does. Without an explicit `ssh-keygen -A` during the build, `sshd` starts,
  finds no host keys, and exits immediately — silently breaking the container.
- **Foreground vs background as the keep-alive pattern.** Managed nodes use
  `sshd -D` (foreground SSH = the container's main process), so the container lives
  as long as SSH does. The control node has no long-running service, so it uses
  `tail -f /dev/null` as a "stay alive so I can `exec` in" trick.
- **Privileged group names differ.** Ubuntu uses `sudo`; the RHEL family uses `wheel`.
  Same concept, different name — easy to miss until your `useradd -G sudo` quietly
  fails on Rocky.

---

## Skills demonstrated

- Containerization with Docker (multi-stage thinking, foreground-process patterns,
  build-time vs runtime instruction separation)
- Multi-distro Linux administration (Debian-family and RHEL-family differences)
- SSH key management and least-privilege user setup
- Ansible architecture (control node / managed nodes, agentless model via SSH+Python)
- Infrastructure-as-code mindset: the entire environment is described by ~3 short
  text files and a compose manifest

---

