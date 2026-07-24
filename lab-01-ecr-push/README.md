# Lab 01 — Push an nginx image to Amazon ECR with GitHub Actions

Practice lab: a GitHub Actions workflow that builds a custom nginx image
("Hello World.. From GitHub actions!") and pushes it to Amazon ECR on demand
(`workflow_dispatch`).

- **ECR repository:** `hello-nginx`
- **Region:** `us-east-1`
- **Registry URI:** `840282986436.dkr.ecr.us-east-1.amazonaws.com/hello-nginx`
- **Workflows:**
  - `.github/workflows/deploy-ecr.yml` — baseline, hardcoded `latest` tag
  - `.github/workflows/deploy-ecr-tagged.yml` — tag chosen at run time via input

## App

- `app/Dockerfile` — `FROM nginx:alpine`, copies in `index.html`
- `app/index.html` — the page served

---

## AWS CLI cheat-sheet (ECR)

> `--region us-east-1` is on every command because ECR is **regional** — an image
> in one region is invisible to a command targeting another.
> `--query` uses **JMESPath** to trim the raw JSON to the fields you care about;
> `--query` and `--output` are universal AWS CLI flags, not ECR-specific.

### Authentication / identity

```bash
# Confirm which AWS identity your CLI is using (catches bad/expired keys)
aws sts get-caller-identity
```

### Create the registry repository

```bash
# Create the ECR repo (one repo per image name), with scan-on-push enabled
aws ecr create-repository \
  --repository-name hello-nginx \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1
```

### Authenticate Docker to ECR

```bash
# Get a 12-hour token and pipe it into docker login
# (this is what the amazon-ecr-login@v2 action does under the hood)
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin 840282986436.dkr.ecr.us-east-1.amazonaws.com
```

### Inspect what's in the repo

```bash
# Simple list of images with tags, push time, size
aws ecr describe-images --repository-name hello-nginx --region us-east-1 \
  --query 'imageDetails[].{tags:imageTags,pushedAt:imagePushedAt,sizeMB:imageSizeInBytes}' \
  --output table

# Sorted by push time, showing digests
# — reveals tag pointers moving and orphaned (untagged) images
aws ecr describe-images --repository-name hello-nginx --region us-east-1 \
  --query 'sort_by(imageDetails,&imagePushedAt)[].{digest:imageDigest,tags:imageTags,pushedAt:imagePushedAt}' \
  --output json
```

### List repositories

```bash
aws ecr describe-repositories --region us-east-1 --output table
```

### Delete images (cleanup)

```bash
# Delete a specific image by tag
aws ecr batch-delete-image --repository-name hello-nginx --region us-east-1 \
  --image-ids imageTag=1.0

# Delete an orphaned (untagged) image by digest
aws ecr batch-delete-image --repository-name hello-nginx --region us-east-1 \
  --image-ids imageDigest=sha256:<digest>
```

---

## Local run (pull the image the runner built and serve it)

```bash
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin 840282986436.dkr.ecr.us-east-1.amazonaws.com

docker run --rm --name hello -p 8080:80 \
  840282986436.dkr.ecr.us-east-1.amazonaws.com/hello-nginx:latest
# open http://localhost:8080  → stop with: docker stop hello
```

---

## Testing an image locally — checklist

```bash
# ── Set once per session (edit these two) ──────────────────
REGISTRY=840282986436.dkr.ecr.us-east-1.amazonaws.com
IMAGE=hello-nginx:latest          # or hello-nginx@sha256:<digest>

# 1. Clear the port — stop any container still holding 8080
docker ps                          # look for 0.0.0.0:8080->80
docker stop hello 2>/dev/null      # ignore error if nothing's running

# 2. Authenticate to ECR (token lasts 12h; re-run is harmless)
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin $REGISTRY

# 3. Pull to defeat the stale-tag cache  ← skip only if using @sha256
docker pull $REGISTRY/$IMAGE

# 4. Run — --rm = auto-delete on stop, --name = predictable handle
docker run --rm --name hello -p 8080:80 $REGISTRY/$IMAGE

# 5. Test → open http://localhost:8080

# 6. Stop (from another terminal) — container self-deletes thanks to --rm
docker stop hello
```

| Flag | Does | If you omit it |
|------|------|----------------|
| `--rm` | Deletes container when it stops | Dead containers pile up; clean with `docker rm` |
| `--name hello` | Fixed name for `docker stop` | Docker assigns a random name; you must look it up |
| `-p 8080:80` | Maps localhost:8080 → container's port 80 | Container runs but is unreachable from the browser |
| `-d` (optional) | Detached — frees your terminal | Runs in foreground; `Ctrl+C` stops it |

> **Stale-tag trap:** a tag is a *local* cache pointer too, not just a remote one.
> If your laptop already cached an older `latest`, `docker run :latest` reuses the
> stale copy. Run `docker pull` first, or reference the image by `@sha256:<digest>`
> (a digest is the content, so it can never be stale).

### Stopping Docker itself (macOS)

Docker Desktop runs the Linux daemon inside a VM, so there is no `systemctl`.
Use the Docker Desktop CLI:

```bash
docker desktop stop       # gracefully stop the engine/VM (keeps the app ready)
docker desktop start      # bring it back up
docker desktop status     # running or stopped?
osascript -e 'quit app "Docker"'   # fully quit the app (graceful)
killall Docker            # force-kill — last resort if it's hung
```

---

## Tagging experiment (Runs A–C)

Goal: observe what happens to images and tags across repeated builds/pushes,
using `deploy-ecr-tagged.yml` (tag chosen at run time).

| Run | Edit | Tag pushed | Result in ECR |
|-----|------|-----------|---------------|
| A | — | `1.0` | `1.0` added alongside existing `latest` (clean coexistence) |
| B | `V2` | `latest` | new digest takes `latest`; **old `latest` becomes an untagged orphan** |
| C | `V3` | `1.1` | `1.1` added; **no new orphan** (new content got its own name) |

**What it proves**

- Editing content always produces a **new digest** (build captures the whole
  artifact — even the base `nginx:alpine` moving underneath can change the digest).
- **Re-pushing an existing tag** (`latest` in Run B) *moves the pointer* and
  strands the previous image as an untagged orphan — not an overwrite, not a
  duplicate tag.
- **Using a new tag** (`1.1` in Run C) avoids orphans entirely → the case for
  versioned/immutable tags over reusing `latest`.

Watch it live with the digest-sorted `describe-images` command in the cheat-sheet
above; the orphan shows up as `"tags": null`.

---

## Lifecycle policy (automated cleanup)

Applied to `hello-nginx` so orphans clean themselves up — no manual sweeping:

```bash
aws ecr put-lifecycle-policy --repository-name hello-nginx --region us-east-1 \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Expire untagged images older than 1 day",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "days",
          "countNumber": 1
        },
        "action": { "type": "expire" }
      }
    ]
  }'

# View / remove the policy
aws ecr get-lifecycle-policy    --repository-name hello-nginx --region us-east-1
aws ecr delete-lifecycle-policy --repository-name hello-nginx --region us-east-1
```

---

## Key concepts

- **Tag = pointer, digest = content.** A tag (`latest`, `1.1`) is a *mutable
  label* pointing at an image **digest** (`sha256:...`), just like a git branch
  name points at a commit SHA.
- **Re-pushing the same tag doesn't overwrite** — the new image gets a new digest,
  the tag *moves* to it, and the old image stays in ECR as an `<untagged>` orphan
  (costs storage until cleaned).
- **Versioned tags** (`1.0`, `1.1`, git SHA) avoid orphans and give traceability.
  Treat `latest` as a floating "most recent" alias, never as a version.
- **Guardrails:** enable *tag immutability* to reject re-pushes of an existing tag;
  use *lifecycle policies* to auto-delete untagged/old images.
