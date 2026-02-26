# Self-Hosted n8n with Git Version Control

This repository contains exported workflows from a self-hosted **n8n** instance running on **Amazon EC2**.

Workflows are automatically exported from the running n8n container and synced to this **GitHub** repository for:

* Version control
* Backup
* Change tracking
* Git-based deployment workflows

---

## Architecture Overview

```
EC2 (Ubuntu)
 ├── Docker (n8n container)
 ├── Persistent volume (~/n8n-data)
 ├── Nginx (Reverse Proxy + SSL)
 └── Workflow Sync Script
        ↓
      GitHub
```

* n8n runs in Docker
* Nginx handles HTTPS
* Workflows are exported as JSON
* A cron job syncs changes to this repository

---

## Setup Overview

### Run n8n in Docker

```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v ~/n8n-data:/home/node/.n8n \
  -e N8N_BASIC_AUTH_ACTIVE=true \
  -e N8N_BASIC_AUTH_USER=admin \
  -e N8N_BASIC_AUTH_PASSWORD=admin \
  -e N8N_ENCRYPTION_KEY=your-secret-key \
  n8nio/n8n
```

---

### Export Workflows (Separate Files)

```bash
docker exec n8n n8n export:workflow \
  --all \
  --separate \
  --output=/home/node/.n8n/workflows/
```

Each workflow is exported as an individual JSON file.

---

### Sync Script

`sync-workflows.sh`

```bash
#!/bin/bash

set -e  # stop on error

EXPORT_DIR="$HOME/n8n-data/workflows"
REPO_DIR="$HOME/n8n-workflows"

echo "==> Ensuring export directory exists"
docker exec n8n mkdir -p /home/node/.n8n/workflows

echo "==> Exporting workflows separately"
docker exec n8n n8n export:workflow \
  --all \
  --separate \
  --output=/home/node/.n8n/workflows/

cd "$EXPORT_DIR" || exit 1

echo "==> Renaming workflow files"

for file in *.json; do
  [ -e "$file" ] || continue

  # Extract name + id safely
  name=$(jq -r '.name // empty' "$file")
  id=$(jq -r '.id // empty' "$file")

  if [[ -z "$name" || -z "$id" ]]; then
    echo "Skipping invalid file: $file"
    continue
  fi

  # Sanitize name
  safe_name=$(echo "$name" | tr ' ' '-' | tr -cd '[:alnum:]-')

  # Take first 6 chars of ID
  short_id=${id:0:6}

  new_name="${safe_name}_${short_id}.json"

  # Avoid renaming to same name
  if [[ "$file" != "$new_name" ]]; then
    mv -f "$file" "$new_name"
  fi
done

echo "==> Syncing to Git repository"

cd "$REPO_DIR" || exit 1

# Copy only workflow JSON files (no deletion, safe)
cp "$EXPORT_DIR"/*.json "$REPO_DIR"/ 2>/dev/null || true

git add *.json

if git diff --cached --quiet; then
  echo "No changes to commit"
else
  git commit -m "Auto-sync n8n workflows $(date)"
  git push
fi

echo "==> Sync complete."
```

---

### Automate with Cron

```bash
crontab -e
```

Add:

```
*/10 * * * * /home/ubuntu/sync-workflows.sh >> /home/ubuntu/sync.log 2>&1
```

This syncs workflows every 10 minutes.

---

## Security Notes

* Credentials are encrypted inside n8n database
* API keys are NOT exported in workflow JSON
* HTTPS is configured using Let's Encrypt
* Port 5678 should NOT be publicly exposed
* Access is protected via Basic Auth

---

## Repository Structure

```
.
├── workflow-1.json
├── workflow-2.json
├── workflow-3.json
└── README.md
```

Each JSON file represents a single n8n workflow.

---

## Restoring Workflows

To import workflows into a new n8n instance:

```bash
docker exec -it n8n n8n import:workflow \
  --separate \
  --input=/home/node/.n8n/workflows/
```
