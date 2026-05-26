# AstraXK ERP - Production Setup & Deployment Guide

This guide provides a comprehensive, step-by-step walkthrough to build, deploy, configure, and maintain the white-labeled **AstraXK ERP** system in a secure, multi-tenant production environment using Docker containers and Cloudflare R2 object storage.

---

## Architecture Overview

In **development**, code is locally mounted inside containers for quick editing.  
In **production**, we follow Docker best practices:
1. **Immutable Images**: All applications (`frappe`, `erpnext`, `hrms`, and your custom `erpnext_customizations`) are compiled and **baked directly into the Docker image** at build time.
2. **Stateless Containers**: Containers can be destroyed and recreated instantly. Permanent uploads go directly to Cloudflare R2, and database entries go to a secure MariaDB instance.
3. **Automated Migrations**: Upgrades automatically migrate database schemas without manual CLI interaction.

---

## Step 1: Build the Custom Production Image

Your custom apps are defined in [apps.json](file:///c:/Users/artan/OneDrive/Desktop/erp/frappe_docker/apps.json) which points to your GitHub repository:
* `erpnext` (version-16)
* `hrms` (version-16)
* `erpnext_customizations` (master)
* `offsite_backups` (develop)

Run the following command in your terminal from the root `frappe_docker` directory to build your production image. We use the `--secret` flag to pass your app list securely, and `CACHE_BUST` to force Docker to pull any new commits from your GitHub repositories:

```bash
docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-15 \
  --build-arg=CACHE_BUST="$(date +%s)" \
  --secret=id=apps_json,src=apps.json \
  --tag=astraxk-erp:production \
  --file=images/layered/Containerfile .
```

---

## Step 2: Configure the Production Stack (Compose)

We configure a multi-container stack including MariaDB, Redis, Caddy (for automatic SSL), and the Frappe backend.

### 1. Create a `compose.yaml` file
In your production directory, write your Docker Compose configuration. Ensure you reference your newly built image `astraxk-erp:production`:

```yaml
services:
  mariadb:
    image: mariadb:11.8
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --skip-character-set-client-handshake
    environment:
      MYSQL_ROOT_PASSWORD: YourSecureRootPassword123
    volumes:
      - mariadb-data:/var/lib/mysql

  redis-cache:
    image: redis:alpine

  redis-queue:
    image: redis:alpine

  backend:
    image: astraxk-erp:production
    environment:
      - DB_HOST=mariadb
      - REDIS_CACHE=redis://redis-cache:6379
      - REDIS_QUEUE=redis://redis-queue:6379
    volumes:
      - assets-data:/home/frappe/frappe-bench/sites/assets
      - sites-data:/home/frappe/frappe-bench/sites

  web:
    image: astraxk-erp:production
    command: nginx -g "daemon off;"
    environment:
      - FRAPPE_BACKEND=backend:8000
      - FRAPPE_SOCKETIO=websocket:9000
    volumes:
      - assets-data:/usr/share/nginx/html/assets
      - sites-data:/usr/share/nginx/html/sites
    ports:
      - "80:80"
      - "443:443"

volumes:
  mariadb-data:
  assets-data:
  sites-data:
```

### 2. Boot the Production Stack
```bash
docker compose up -d
```

---

## Step 3: Production Site Creation & Installation

Execute these commands to bootstrap your production site (e.g. `erp.astraxk.com`):

### 1. Create the Production Site
Initialize the site inside the active `backend` container under the secure `frappe` user:
```bash
docker compose exec -u frappe backend bench new-site \
  --db-root-password YourSecureRootPassword123 \
  --admin-password YourAdminPortalPassword123 \
  --mariadb-user-host-login-scope=% \
  erp.astraxk.com
```

### 2. Set Developer Mode off (Production Safety)
Keep developer mode disabled in production to protect metadata:
```bash
docker compose exec -u frappe backend bench --site erp.astraxk.com set-config developer_mode 0
```

### 3. Install the Applications
Install the baked core apps and your custom whitelabeled app onto your site:
```bash
docker compose exec -u frappe backend bench --site erp.astraxk.com install-app erpnext hrms erpnext_customizations offsite_backups
```

---

## Step 4: Configure Cloudflare R2 Custom Storage

To enable the custom storage system and automated whitelabeling for the new production site:

### 1. Edit the site config
Navigate to your production volume or edit the configuration inside the container:
```bash
docker compose exec -u frappe backend nano sites/erp.astraxk.com/site_config.json
```

Add your S3/R2 keys to `site_config.json`:
```json
{
  "db_host": "mariadb",
  "db_name": "your_db_name",
  "db_password": "your_db_password",
  "db_type": "mariadb",
  "db_user": "your_db_user",
  
  "s3_access_key": "YOUR_R2_ACCESS_KEY_ID",
  "s3_secret_key": "YOUR_R2_SECRET_ACCESS_KEY",
  "s3_bucket": "astra-erp-files",
  "s3_endpoint_url": "https://<account-id>.r2.cloudflarestorage.com",
  "s3_public_domain": "https://cdn.astraxk.com",
  "s3_prefix": "astra-production"
}
```

### 2. Run cache-clearing & migrations
Apply changes immediately:
```bash
docker compose exec -u frappe backend bench --site erp.astraxk.com clear-cache
docker compose exec -u frappe backend bench --site erp.astraxk.com migrate
```

---

## Step 5: Configure Automatic Database Backups to R2

Secure your database daily:
1. Log in to `https://erp.astraxk.com` as **Administrator**.
2. Search for **S3 Backup Settings** in the Awesomebar.
3. Configure it using your Cloudflare R2 details:
   * **Access Key**: Your R2 Access Key ID.
   * **Secret Key**: Your R2 Secret Access Key.
   * **Bucket Name**: `astra-erp-backups` (Ensure this bucket is created in Cloudflare R2!).
   * **Endpoint URL**: `https://<account-id>.r2.cloudflarestorage.com`
   * **Frequency**: `Daily`
   * **Backup Limit**: `14` (Removes backups older than 14 days automatically).
4. Save and check **Enable Automatic Backup**.

---

## Step 6: Enable Automated Updates & CI/CD Pipelines

When you push updates to `erpnext_customizations` on GitHub, you can automate deployments securely:

1. **Rebuild the image**: Run the build command in **Step 1** (this will fetch the new changes from master automatically).
2. **Start the Migrator container**: Use the official migrator service override. In your `compose.yaml`, define a service that automatically updates database schemas and clear caches upon container restart:

```yaml
  migrator:
    image: astraxk-erp:production
    command: bench --site all migrate
    volumes:
      - assets-data:/home/frappe/frappe-bench/sites/assets
      - sites-data:/home/frappe/frappe-bench/sites
    environment:
      - DB_HOST=mariadb
      - REDIS_CACHE=redis://redis-cache:6379
      - REDIS_QUEUE=redis://redis-queue:6379
```

When you deploy a new image, running `docker compose up -d` will automatically launch the `migrator` service, migrate all active databases cleanly, and exit!
