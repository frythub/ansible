# Frythub Ansible Playbooks

This folder contains all automation to provision runtimes, databases, TLS, and deploy Frythub apps. Playbooks are grouped into collections:

- `frythub.platform` — base runtimes, Nginx, certbot, Postgres install/opening, connectivity checks, bulk Let's Encrypt.
- `frythub.db` — database creation playbooks.
- `frythub.apps` — app deployments (web/PM2, .NET API) and API TLS.

## Prerequisites
- Ansible installed on the control node.
- Inventory at `ansible/inventory/hosts.ini` with groups `web` and `db` as needed.
- SSH key access and passwordless sudo for the `ansible` user on targets.
- `ansible/ansible.cfg` already points to `./inventory/hosts.ini` and `./collections`.

## Platform playbooks (runtimes, connectivity, TLS)
Run with:
```
ansible-playbook -i ansible/inventory/hosts.ini ansible/collections/ansible_collections/frythub/platform/playbooks/<playbook>.yml
```

- `test-orchestration.yml` — quick ping + uptime check on all hosts.
  - Example: `.../test-orchestration.yml`
- `setup-nginx.yml` — install/start Nginx, deploys a simple landing page.
  - Example: `.../setup-nginx.yml`
  - Override: `-e nginx_welcome_text="Hello from Frythub"`
- `setup-certbot.yml` — install certbot + nginx plugin (no cert issuance).
  - Example: `.../setup-certbot.yml`
- `setup-node-pm2.yml` — install Node.js (default major 20) and PM2.
  - Example: `.../setup-node-pm2.yml`
  - Override Node major: `-e nodejs_major=18`
- `setup-dotnet.yml` — install ASP.NET Core runtime and dotnet runtime (default 8.0; falls back to jammy feed on noble).
  - Example: `.../setup-dotnet.yml`
  - Override runtime: `-e dotnet_release=6.0`
- `setup-postgres.yml` — install PostgreSQL from PGDG repo and start service.
  - Example: `.../setup-postgres.yml`
- `enable-postgres-remote.yml` — set `listen_addresses='*'` and add pg_hba rules.
  - **Danger**: defaults to open CIDRs (`0.0.0.0/0`, `::/0`). Lock down:
    ```
    ansible-playbook -i ansible/inventory/hosts.ini ansible/collections/ansible_collections/frythub/platform/playbooks/enable-postgres-remote.yml \
      -e "allowed_cidr4=1.2.3.4/32" -e "allowed_cidr6="
    ```
- `letsencrypt-all-domains.yml` — bulk cert issuance for multiple domains (if configured).
  - Example: `.../letsencrypt-all-domains.yml`

## Database playbooks
Run with:
```
ansible-playbook -i ansible/inventory/hosts.ini ansible/collections/ansible_collections/frythub/db/playbooks/<playbook>.yml
```

- `create-frythub-test-db.yml` — create `frythub_test` (owner default `postgres`).
- `create-frythub-prod-db.yml` — create `frythub_prod` (owner default `postgres`).
  - Override owner: `-e db_owner=appuser`
  - Requires Postgres already installed/running.

## App playbooks (deployments & API TLS)
Run with:
```
ansible-playbook -i ansible/inventory/hosts.ini ansible/collections/ansible_collections/frythub/apps/playbooks/<playbook>.yml
```

- `deploy-web-apps.yml` — build/deploy Next.js apps with PM2 + Nginx templating.
  - Ensure `ansible/group_vars/web/apps.yml` is filled out.
- `deploy-dotnet-api.yml` — checkout/publish .NET API, install systemd unit, Nginx proxy.
  - Override environment: `-e "target_env=test"` or `-e "target_env=prod"` (default test).
  - Creates a blank `.env` if missing at the repo root for that environment.
- `letsencrypt-api.yml` — issue/renew Let's Encrypt cert for the API domain.
  - Prod (default): `.../letsencrypt-api.yml`
  - Test: `.../letsencrypt-api.yml -e "target_env=test"`

## Ansible Vault (secrets)
- Copy example and edit: `cp ansible/group_vars/all/vault.example.yml ansible/group_vars/all/vault.yml`
- Encrypt: `ansible-vault encrypt ansible/group_vars/all/vault.yml`
- Edit: `ansible-vault edit ansible/group_vars/all/vault.yml`
- Run with vault prompt: append `--ask-vault-pass` to playbook commands.

## Notes and tips
- Inventory lives at `ansible/inventory/hosts.ini`; groups referenced above (`web`, `db`).
- Collection path is set in `ansible/ansible.cfg`; use full relative paths shown above or add to `ANSIBLE_COLLECTIONS_PATHS`.
- For Postgres remote access, always narrow `allowed_cidr4/6` when not testing.
- For cert issuance, ensure DNS points at the target host and Nginx is serving the domain on port 80 first.
