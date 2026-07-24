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
