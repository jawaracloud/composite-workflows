# Build & Push Multi-Arch ECR Images with Buildah + Mirrored Cache

---

## 📌 What it does
- Builds **multi-arch** Docker images (`linux/amd64` + `linux/arm64`)  
- Uses **Buildah** (daemon-less, rootless, fast)  
- Pushes to **Amazon ECR** (or any registry you log in to)  
- Keeps a **mirrored cache image** so subsequent builds re-use layers across runners  
- Requires **only Docker + Buildah** on the runner—works on GitHub-hosted or fresh self-hosted runners

---

## 🚀 Quick start

```yaml
- uses: jawaracloud/build-push-ecr@v1
  with:
    img_name:           my-service
    ecr_registry:       ${{ secrets.ecr_registry }}
    aws_region:         ${{ secrets.aws_region }}
    aws_profile:        ${{ secrets.aws_profile }}
    build_args:         "NODE_ENV=production BUILD_DATE=2024-07-24"
    build_locations:    ./docker
    use_env_file:       true      # optional
```

Result:  
`123456789012.dkr.ecr.us-east-1.amazonaws.com/my-service:1.2.3`  
is a multi-arch manifest ready to `docker run` anywhere.

---

## 📥 Inputs

| Name               | Required | Default | Description |
|--------------------|----------|---------|-------------|
| `img_name`         | ✅       | —       | Image repository name (no registry or tag) |
| `build_args`       | ❌       | —       | Space-separated `KEY=value` pairs passed to `buildah build` |
| `build_locations`  | ❌       | `.`     | Build context directory |
| `use_env_file`     | ❌       | `false` | If `true`, downloads artifact `env-file` into the build context |
| `ecr_registry`     | ✅       | —       | Registry URL, e.g. `123456789012.dkr.ecr.us-east-1.amazonaws.com` |
| `aws_region`       | ✅       | —       | AWS region |
| `aws_profile`      | ✅       | —       | IAM role ARN or AWS profile name to assume |
| `cache_image_name` | ❌       | `cache` | Name of the dedicated cache image inside the same registry |
| `cache_ttl`        | ❌       | `24h`   | How long Buildah trusts cached layers |

---

## 📤 Outputs

| Name        | Description |
|-------------|-------------|
| `image_tag` | Full `<registry>/<image>:<tag>` that was pushed |
| `version`   | Version string read from the `VERSIONS` file |

Use them in later steps:

```yaml
- id: build
  uses: jawaracloud/build-push-ecr@v1
  with: …
- run: echo "Built ${{ steps.build.outputs.image_tag }}"
```

---

## 🗂️ Required files

* `VERSIONS` (in repo root) – single line containing the semantic tag, e.g. `1.2.3`  
* (Optional) Upload an artifact named `env-file` if `use_env_file: true`

---

## 🔧 Runner prerequisites

Minimal:
- Docker Engine (for registry login helpers)  
- Buildah (auto-installed by the action if missing)

No Docker daemon is used for the actual build.

---

## 🧪 Example workflow

```yaml
name: ci
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build & push
        id: build
        uses: jawaracloud/build-push-ecr@v1
        with:
          img_name: api
          ecr_registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com
          aws_region: us-east-1
          aws_profile: arn:aws:iam::123456789012:role/gha-ecr-push
          build_args: "COMMIT_SHA=${{ github.sha }}"

      - name: Deploy
        run: |
          echo "New image: ${{ steps.build.outputs.image_tag }}"
```

---

## 🔄 Cache behaviour

- Layers are cached in a separate image:  
  `<ecr_registry>/<cache_image_name>:<VERSION>`  
- `--cache-ttl` keeps the cache image fresh  
- First build after cache expiry will re-create the cache image  
- Cache image is **not** part of the final manifest, so no runtime bloat