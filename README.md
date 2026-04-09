Ansible Role SSH Hardening
==========================

Hardens the SSH daemon on Ansible-managed Linux hosts. Deploys a strict
`sshd_config` with modern algorithms only, restricts SSH access to a
dedicated group, and removes all non-Ed25519 host keys.

Designed to run as root. Server provisioning (including root's SSH key)
is handled by Terraform before this role runs.

Installation
------------

Install directly from GitHub:

```bash
ansible-galaxy role install git+https://github.com/your-org/ansible-role-ssh-hardening.git
```

To install into a local `./roles` directory instead of the default system path:

```bash
ansible-galaxy role install git+https://github.com/your-org/ansible-role-ssh-hardening.git \
  -p ./roles
```

Or pin to a specific version using a tag or commit:

```bash
ansible-galaxy role install git+https://github.com/your-org/ansible-role-ssh-hardening.git,v1.0.0 \
  -p ./roles
```

To avoid specifying `-p ./roles` every time, set the roles path in `ansible.cfg`:

```ini
[defaults]
roles_path = ./roles
```

To manage it declaratively, add it to `requirements.yml`:

```yaml
roles:
  - name: ansible-role-ssh-hardening
    src: https://github.com/your-org/ansible-role-ssh-hardening.git
    scm: git
    version: main
```

Then install with:

```bash
ansible-galaxy install -r requirements.yml -p ./roles
```

To update all roles in `requirements.yml` at once:

```bash
ansible-galaxy install -r requirements.yml -p ./roles --force
```

`--force` reinstalls every listed role regardless of whether it's already present. If a role has `version: main`, it pulls the latest commit.

To update only this role without affecting others:

```bash
ansible-galaxy role install git+https://github.com/your-org/ansible-role-ssh-hardening.git \
  -p ./roles --force
```

Requirements
------------

- OpenSSH 8.5 or later (required for `sntrup761x25519-sha512` key exchange)
- Server provisioned by Terraform with root's SSH public key in place
- Ansible 2.10+

Role Variables
--------------

| Variable | Default | Description |
|---|---|---|
| `ssh_group` | `ssh-users` | Group that `AllowGroups` gates SSH access on. Root is added to this group automatically. |

Dependencies
------------

None.

Example Playbook
----------------

```yaml
- hosts: all
  remote_user: root
  roles:
    - role: ansible-role-ssh-hardening
```

To override the SSH group:

```yaml
- hosts: all
  remote_user: root
  roles:
    - role: ansible-role-ssh-hardening
      vars:
        ssh_group: wheel
```

What the role does
------------------

1. Creates the `ssh_group` group
2. Adds root to `ssh_group`
3. Removes all non-Ed25519 host keys (RSA, ECDSA, DSA)
4. Deploys the hardened `sshd_config` (validated with `sshd -t` before writing)
5. Reloads sshd

Hardening applied
-----------------

- **Key exchange:** `sntrup761x25519-sha512` (post-quantum hybrid), `curve25519-sha256` fallback
- **Ciphers:** `chacha20-poly1305`, `aes256-gcm` — AEAD only, no CBC
- **MACs:** `hmac-sha2-512-etm` — ETM only
- **Host key:** Ed25519 exclusively; `HostKeyAlgorithms` and `PubkeyAcceptedAlgorithms` locked to `ssh-ed25519`
- **Authentication:** public key only — password, keyboard-interactive, GSSAPI, Kerberos all disabled
- **Root login:** `prohibit-password` — key only
- **Access control:** `AllowGroups` — all accounts not in `ssh_group` are blocked
- **Forwarding:** TCP, stream-local, agent, tunnel, X11 all disabled
- **Session:** 5-minute idle timeout, `MaxStartups 10:30:60` flood protection
- **Misc:** compression off, DNS lookup off, user environment disabled

License
-------

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
