# Forgejo & Chainguard Postgres via Podman Quadlets

Based on [Podman in Production: Quadlets, Secrets, Auto-Updates, and Docker Compatibility](https://blog.hofstede.it/podman-in-production-quadlets-secrets-auto-updates-and-docker-compatibility/#real-world-deployment-a-complete-stack)

## 🛠️ Local Testing (Lima-VM)

To test this playbook in a clean environment without messing up your host machine, use [Lima](https://github.com/lima-vm/lima).

### Test Workflow

Run these commands to execute the test:

```bash
# Spin up the VM
limactl start --yes fedora-test.yaml

limactl shell fedora-test sudo podman network create frontend

# Run the playbook from your host
ansible-playbook -i inventory.yml deploy_forgejo.yml
```

## 🔐 Security & Secrets

This playbook follows a **"Generate-on-Host"** strategy.

1. The playbook checks if the `forgejo_db_password` secret exists.
2. If it's missing, it runs `pwgen -s 32 1 | tr -d '\n' | podman secret create ...`.
3. The password never leaves the target server's memory/disk.

**To retrieve the password for your password manager:**

```bash
limactl shell fedora-test sudo podman secret inspect --showsecret forgejo_db_password --format '{{.SecretData}}'
```

---

## 📂 Data & Permissions

The playbook manages volumes at `/opt/forgejo/`. Because containers run as specific UIDs, we use **ACLs** instead of simple `chown` to allow the host and container to share access gracefully:

* **Forgejo (UID 1000):** Owns `/opt/forgejo/forgejo`.
* **Postgres (UID 65532):** Owns `/opt/forgejo/postgres`.

---

## 🔍 Verification

Once the playbook completes, verify the stack is healthy:

| Action | Command |
| :--- | :--- |
| **Check Services** | `systemctl status forgejo-server.service` |
| **View Logs** | `journalctl -u forgejo-server.service -f` |
| **Check Containers** | `podman ps` |
| **Verify Network** | `podman network inspect forgejo-backend` |

Open the web ui at <http://localhost:3000>

### Cleanup

To destroy the test environment and start over:

```bash
limactl delete -f fedora-test
```
