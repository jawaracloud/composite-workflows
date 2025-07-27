# SSH-deploy a Docker-Compose service in one step

---

## 📌 What it does
- Reads the version from a local `VERSIONS` file  
- Logs in to the remote server via SSH  
- Updates the **image tag** for a single service inside a `docker-compose.yml`  
- Pulls the new image and restarts the service **without downtime**  
- Works on **any runner** (GitHub-hosted or self-hosted) — only needs SSH access

---

## 🚀 Quick start

```yaml
- uses: jawaracloud/deploy-docker-compose@v1
  with:
    img_name:      my-service
    service_name:  api
    compose_path:  /opt/apps/docker-compose.yml
    ssh_key:       ${{ secrets.SSH_KEY }}
    ssh_host:      ${{ secrets.SSH_HOST }}
    ssh_username:  ${{ secrets.SSH_USER }}
    ecr_registry:  123456789012.dkr.ecr.us-east-1.amazonaws.com
```

---

## 📥 Inputs

| Name            | Required | Description |
|-----------------|----------|-------------|
| `img_name`      | ✅       | Docker image repository name (no registry or tag) |
| `service_name`  | ✅       | Service key inside the compose file |
| `compose_path`  | ✅       | Absolute path to `docker-compose.yml` on the target server |
| `ssh_key`       | ✅       | Private key for SSH (PEM format) |
| `ssh_host`      | ✅       | Hostname or IP of the server |
| `ssh_username`  | ✅       | Username to SSH as |
| `ecr_registry`  | ✅       | Registry URL (used to build the full image tag) |

---

## 📤 Outputs

| Name      | Description |
|-----------|-------------|
| `version` | Version string read from the `VERSIONS` file |

Use it later:

```yaml
- id: deploy
  uses: jawaracloud/deploy-docker-compose@v1
  with: …
- run: echo "Deployed version ${{ steps.deploy.outputs.version }}"
```

---

## 🗂️ Required files

* `VERSIONS` (in repo root) – single line with the tag, e.g. `1.2.3`  
* Remote server must have:
  - Docker + Docker Compose v2  
  - `yq` (≥ 4.x) – installed automatically by the action if missing

---

## 🧪 Example workflow

```yaml
name: deploy
on:
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: jawaracloud/deploy-docker-compose@v1
        with:
          img_name:      api
          service_name:  api
          compose_path:  /opt/apps/docker-compose.yml
          ssh_key:       ${{ secrets.SSH_KEY }}
          ssh_host:      ${{ secrets.SSH_HOST }}
          ssh_username:  ${{ secrets.SSH_USER }}
          ecr_registry:  123456789012.dkr.ecr.us-east-1.amazonaws.com
```

---

## 🔐 SSH key setup
1. Generate key locally:  
   `ssh-keygen -t ed25519 -C "gha-deploy" -f deploy_key`  
2. Add the **public key** to the server:  
   `ssh-copy-id -i deploy_key.pub user@host`  
3. Store the **private key** in GitHub Secrets (`SSH_KEY`).

---

## 🔄 What happens on the server

```text
1. cd /opt/apps
2. yq eval '.services.api.image = "<registry>/my-service:1.2.3"' docker-compose.yml
3. docker compose pull api
4. docker compose up -d --remove-orphans api
```

---

## 📄 License

MIT © Jawara Cloud