# Plane 1.5.0 — Configurable static asset rate limits (Web and Admin)

These instructions make Plane's static frontend asset rate limits configurable through `.env`, covering both normal users and administrators (God mode).

The configuration is stored on the host, so it survives container recreation and can be adjusted without editing either Caddyfile.

Two independent settings are introduced:

| Environment variable | Applies to | Default |
|---|---|---:|
| `WEB_ASSET_RATE_LIMIT` | Normal frontend users | 300 |
| `ADMIN_ASSET_RATE_LIMIT` | Admin (God mode) frontend | 300 |

Both limits are requests per minute per client IP.

## 1. Create a persistent Caddy configuration directory

From your Plane installation directory:

```bash
sudo mkdir -p caddy
```

## 2. Create the persistent Caddyfiles

We need separate persistent Caddyfiles for the two frontend services.

### 2.1 Web Caddyfile

Use the original Caddyfile as the starting point, excluding the temporary diagnostic header added during troubleshooting.

A backup was previously created at `/tmp/plane-Caddyfile.backup`.

Copy it into the new directory:

```bash
sudo cp /tmp/plane-Caddyfile.backup caddy/web.Caddyfile
```

### 2.2 Admin Caddyfile

Extract the original Caddyfile from the running admin container:

```bash
sudo docker compose exec -T admin cat /etc/caddy/Caddyfile | sudo tee caddy/admin.Caddyfile > /dev/null
```

This must be done before mounting the replacement Caddyfile over the container's original configuration.

### 2.3 Check both files

Check that the temporary diagnostic header is absent:

```bash
sudo grep -n 'X-Debug-Client-IP' caddy/web.Caddyfile caddy/admin.Caddyfile
```

This should return no output.

Check that both files contain the original rate-limit setting:

```bash
sudo grep -n 'events 300' caddy/web.Caddyfile caddy/admin.Caddyfile
```

Both files should contain the `events 300` directive.

## 3. Make both rate limits configurable

Edit the Web Caddyfile:

```bash
sudo nano caddy/web.Caddyfile
```

Find:

```caddy
events 300
```

Replace it with:

```caddy
events {$WEB_ASSET_RATE_LIMIT:300}
```

Save and exit.

Now edit the Admin Caddyfile:

```bash
sudo nano caddy/admin.Caddyfile
```

Find:

```caddy
events 300
```

Replace it with:

```caddy
events {$ADMIN_ASSET_RATE_LIMIT:300}
```

Save and exit.

The default remains 300 if the corresponding environment variable is not defined.

## 4. Add the settings to `.env`

Add these lines to the Plane `.env` file:

```ini
# Static frontend asset rate limits (requests per minute per client IP)
WEB_ASSET_RATE_LIMIT=1000
ADMIN_ASSET_RATE_LIMIT=1000
```

Both limits are initially set to 1,000 requests per minute.

They can subsequently be adjusted independently without editing either Caddyfile.

## 5. Modify `docker-compose.yml`

We need to pass the appropriate environment variable and mount the corresponding Caddyfile into each frontend service.

### 5.1 Web service

Find the existing `web` service:

```yaml
web:
  image: makeplane/plane-frontend:${APP_RELEASE:-v1.5.0}
  deploy:
    replicas: ${WEB_REPLICAS:-1}
    restart_policy:
      condition: any
  depends_on:
    - api
    - worker
```

Add the `environment` and `volumes` sections:

```yaml
web:
  image: makeplane/plane-frontend:${APP_RELEASE:-v1.5.0}
  environment:
    WEB_ASSET_RATE_LIMIT: ${WEB_ASSET_RATE_LIMIT:-300}
  volumes:
    - ./caddy/web.Caddyfile:/etc/caddy/Caddyfile:ro
  deploy:
    replicas: ${WEB_REPLICAS:-1}
    restart_policy:
      condition: any
  depends_on:
    - api
    - worker
```

### 5.2 Admin service

Find the existing `admin` service:

```yaml
admin:
  image: makeplane/plane-admin:${APP_RELEASE:-v1.5.0}
```

Add the `environment` and `volumes` sections:

```yaml
admin:
  image: makeplane/plane-admin:${APP_RELEASE:-v1.5.0}
  environment:
    ADMIN_ASSET_RATE_LIMIT: ${ADMIN_ASSET_RATE_LIMIT:-300}
  volumes:
    - ./caddy/admin.Caddyfile:/etc/caddy/Caddyfile:ro
```

**Important:** Preserve all other existing settings in the admin service. The example above shows only the relevant additions.

The `:ro` suffix mounts each host Caddyfile read-only inside its respective container.

## 6. Validate the Compose configuration

Run:

```bash
sudo docker compose config -q
```

No output indicates successful validation.

## 7. Recreate both frontend containers

Run:

```bash
sudo docker compose up -d --no-deps --force-recreate web admin
```

This recreates only the two frontend services, leaving the database, API, worker and other services untouched.

## 8. Verify the configuration

### 8.1 Check the environment variables

Web:

```bash
sudo docker compose exec -T web printenv WEB_ASSET_RATE_LIMIT
```

Expected output:

```text
1000
```

Admin:

```bash
sudo docker compose exec -T admin printenv ADMIN_ASSET_RATE_LIMIT
```

Expected output:

```text
1000
```

### 8.2 Check the mounted Caddyfiles

Web:

```bash
sudo docker compose exec -T web cat /etc/caddy/Caddyfile
```

Confirm it contains:

```caddy
events {$WEB_ASSET_RATE_LIMIT:300}
```

Admin:

```bash
sudo docker compose exec -T admin cat /etc/caddy/Caddyfile
```

Confirm it contains:

```caddy
events {$ADMIN_ASSET_RATE_LIMIT:300}
```

### 8.3 Validate both Caddy configurations

Web:

```bash
sudo docker compose exec -T web caddy validate --config /etc/caddy/Caddyfile
```

Admin:

```bash
sudo docker compose exec -T admin caddy validate --config /etc/caddy/Caddyfile
```

Both commands should report successful validation.

Finally, test Plane in the browser using both a normal user account and an administrator account.

Monitor the frontend container logs for further rate-limit messages or HTTP 429 responses.

## 9. Adjusting the limits later

The two settings can be changed independently.

For example:

```ini
WEB_ASSET_RATE_LIMIT=1500
ADMIN_ASSET_RATE_LIMIT=2000
```

After changing `.env`, recreate both frontend services:

```bash
sudo docker compose up -d --no-deps --force-recreate web admin
```

If only one limit has changed, it is sufficient to recreate the corresponding service.

For example, changing only the Web limit:

```bash
sudo docker compose up -d --no-deps --force-recreate web
```

Or changing only the Admin limit:

```bash
sudo docker compose up -d --no-deps --force-recreate admin
```

The limits are per client IP, not per browser or machine. Devices sharing the same public IP therefore share the same allowance within each respective frontend.

## 10. Upgrade consideration

Because the host Caddyfiles are mounted over the Caddyfiles supplied by the frontend images, changes supplied by future Plane frontend images will not automatically appear in the persistent copies.

When upgrading Plane, compare both new image Caddyfiles with their corresponding host copies and incorporate any upstream changes that are needed.

Keep the original Caddyfiles for comparison and rollback.

## Rollback

To restore the original rate limits, replace the configurable directives in both host Caddyfiles:

Web:

```caddy
events 300
```

Admin:

```caddy
events 300
```

Alternatively, restore the original Caddyfiles from their respective backups.

Then recreate both frontend services:

```bash
sudo docker compose up -d --no-deps --force-recreate web admin
```

This restores the original limit of 300 requests per minute per client IP for both frontends, assuming the restored files are the unmodified original configurations.
