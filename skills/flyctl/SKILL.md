---
name: flyctl
description: >
  Full-featured skill for using flyctl, Fly.io's CLI tool. Covers app setup,
  deployment, scaling, secrets, volumes, networking, Postgres, SSH access,
  monitoring, CI/CD, and multi-environment management.
---

# flyctl — Fly.io CLI Skill

## When to use

Use this skill whenever the user asks about deploying, managing, scaling, or operating applications on Fly.io using the `fly` / `flyctl` CLI. This includes initial setup, app configuration (`fly.toml`), deploy workflows, secrets, databases, volumes, networking, monitoring, rollbacks, SSH access, and CI/CD pipelines.

---

## Instructions

### 1. Installation

```bash
# macOS (Homebrew — preferred)
brew install superfly/tap/flyctl

# macOS / Linux (install script)
curl -L https://fly.io/install.sh | sh

# Windows (PowerShell)
iwr https://fly.io/install.ps1 -useb | iex

# Verify
fly version
```

---

### 2. Authentication

```bash
fly auth signup          # Create a new account
fly auth login           # Log in (opens browser)
fly auth token           # Print current auth token
fly auth docker          # Authenticate Docker CLI with Fly registry
fly auth logout
```

**Tokens for CI/CD:**
```bash
fly tokens create deploy --name "github-actions" --expiry 8760h   # App-scoped deploy token
fly tokens create org    --name "ci-org-token"                     # Org-scoped token
```

Store the token as a repository secret (`FLY_API_TOKEN`) — never hardcode it.

---

### 3. Creating and launching apps

```bash
fly launch                          # Detect framework, create fly.toml, optionally deploy
fly launch --no-deploy              # Create fly.toml without deploying
fly launch --name my-app            # Set app name upfront
fly launch --region lhr             # Set primary region
fly launch --config fly.toml        # Use existing config
```

`fly launch` generates a `fly.toml` with sensible defaults. Review it before deploying.

---

### 4. fly.toml — app configuration reference

```toml
app = "my-app"
primary_region = "iad"

[build]
  dockerfile = "Dockerfile"
  # Or use a buildpack:
  # builder = "paketobuildpacks/builder:base"

[build.args]
  NODE_ENV = "production"

[env]
  LOG_LEVEL = "info"
  PORT = "8080"
  # Do NOT put secrets here — use `fly secrets set`

[deploy]
  strategy = "rolling"           # rolling | canary | bluegreen | immediate
  release_command = "bin/rails db:migrate"   # Runs before new Machines start

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = true      # Stop idle Machines to save cost
  auto_start_machines = true
  min_machines_running = 1

  [http_service.concurrency]
    type = "requests"
    hard_limit = 200
    soft_limit = 150

[[vm]]
  size = "shared-cpu-1x"
  memory = "256mb"

[checks]
  [checks.health]
    port = 8080
    type = "http"
    interval = "15s"
    timeout = "10s"
    grace_period = "5s"
    path = "/health"

[mounts]
  source = "my_data"
  destination = "/data"

[processes]
  web    = "node server.js"
  worker = "node worker.js"
```

**Deployment strategies:**
- `rolling` — replace Machines one at a time (safe default)
- `bluegreen` — spin up new Machines, cut over when healthy, then remove old
- `canary` — deploy to one Machine first, promote if healthy
- `immediate` — replace all at once (risk of downtime)

---

### 5. Deploying

```bash
fly deploy                          # Build (remote by default) and deploy
fly deploy --remote-only            # Force remote build
fly deploy --local-only             # Build locally, push image
fly deploy --build-only             # Build and push without deploying
fly deploy --image registry.fly.io/my-app:v1.2.3   # Deploy a specific image
fly deploy --config fly.production.toml             # Use alternate config
fly deploy --detach                 # Fire-and-forget (don't tail logs)
fly deploy --strategy bluegreen     # Override strategy for this deploy
```

**Passing build arguments at deploy time:**
```bash
fly deploy --build-arg NODE_ENV=production --build-arg BUILD_ID=42
```

---

### 6. App status and management

```bash
fly status                    # Summary: Machines, versions, health
fly status --watch            # Continuously refresh
fly info                      # App metadata (hostname, IPs, org)
fly open                      # Open app in browser
fly open /dashboard           # Open a specific path

fly apps list                 # All apps in the org
fly apps create my-app        # Create app without launching
fly apps destroy my-app       # Permanently delete app + resources
fly apps restart              # Rolling restart of all Machines

fly releases                  # List release history
fly releases --image          # Include Docker image references
```

---

### 7. Machines

Fly.io apps run on Fly Machines (micro-VMs). `fly deploy` manages them automatically, but you can also control them directly.

```bash
fly machine list                         # List Machines for current app
fly machine status <machine-id>          # Detailed Machine info
fly machine start  <machine-id>
fly machine stop   <machine-id>
fly machine restart <machine-id>
fly machine restart <machine-id> --force-stop
fly machine delete  <machine-id>
fly machine update  <machine-id> --vm-size performance-2x --vm-memory 2048
fly machine run registry.fly.io/my-app:latest   # Run a one-off Machine
```

---

### 8. Scaling

**Count (horizontal scaling):**
```bash
fly scale count 3                     # Set 3 Machines (across default regions)
fly scale count 2 --region iad,lhr    # 2 Machines per region
fly scale count 1 --process-group worker   # Scale a specific process group
fly scale show                         # Show current scale
```

**VM size (vertical scaling):**
```bash
fly scale vm shared-cpu-1x            # Smallest shared CPU
fly scale vm shared-cpu-2x
fly scale vm performance-1x           # Dedicated CPU
fly scale vm performance-2x
fly scale memory 512                  # Set memory in MB
```

Common VM sizes: `shared-cpu-1x` (256 MB), `shared-cpu-2x`, `shared-cpu-4x`, `performance-1x` (2 GB), `performance-2x`, `performance-4x`, `performance-8x`.

**Autoscale with concurrency limits** — set `[http_service.concurrency]` in fly.toml and Fly will spin Machines up/down based on load.

---

### 9. Secrets

Secrets are encrypted at rest and injected as environment variables at runtime.

```bash
fly secrets set DATABASE_URL="postgres://..." SECRET_KEY="abc123"
fly secrets set --stage KEY=value      # Stage without triggering deploy
fly secrets list                        # Show names + digests (not values)
fly secrets unset KEY_NAME
fly secrets import < .env              # Bulk import from a .env file
```

Never put secrets in `fly.toml` `[env]` — use `fly secrets set` instead.

---

### 10. Volumes (persistent storage)

```bash
fly volumes create my_data --size 10 --region iad
fly volumes list
fly volumes extend <volume-id> --size 20   # Grow volume (cannot shrink)
fly volumes destroy <volume-id>            # Permanent — data is gone

fly volumes snapshots list <volume-id>     # List snapshots
fly volumes snapshots create <volume-id>   # Manual snapshot
```

Mount in fly.toml:
```toml
[mounts]
  source      = "my_data"
  destination = "/data"
```

One volume per Machine — if you scale to N Machines, you need N volumes. Fly will auto-provision volumes on scale-up when `auto_extend_size_threshold` is set.

---

### 11. Networking and IPs

```bash
fly ips list
fly ips allocate-v4 --shared        # Shared IPv4 (free, recommended for most apps)
fly ips allocate-v4                 # Dedicated IPv4 (billed)
fly ips allocate-v6                 # IPv6 (free)
fly ips release <ip-address>
```

**Custom domains / TLS certificates:**
```bash
fly certs create   example.com
fly certs list
fly certs show     example.com
fly certs remove   example.com
```

After `fly certs create`, add a CNAME (or A/AAAA) record per the shown DNS instructions. Fly auto-provisions Let's Encrypt certs once DNS propagates.

**Private networking (6PN):**
Apps on the same Fly.io org reach each other at `<app-name>.internal` on a private IPv6 network. No configuration needed.

```bash
fly proxy 5432 --app my-postgres-app      # Tunnel Postgres to localhost:5432
fly proxy 8080:80 --app other-app         # Tunnel another app's port locally
```

---

### 12. SSH and console access

```bash
fly ssh console                    # Shell into a running Machine
fly ssh console --pty -C "bash"    # Force PTY allocation, run bash
fly ssh console -s                 # Pick a specific Machine interactively
fly ssh console -r lhr             # Connect to a Machine in a specific region
fly ssh console -a other-app       # Access another app

fly console                        # Run the configured console_command from fly.toml

fly ssh sftp shell                 # SFTP session into a Machine
```

For manual SSH (outside flyctl):
```bash
fly ssh issue --username myuser    # Issue SSH certificate for your key
```

---

### 13. Logs and monitoring

```bash
fly logs                           # Tail live logs
fly logs --instance <machine-id>   # Filter to one Machine
fly logs --region lhr              # Filter by region
fly logs -a other-app              # Another app's logs

fly dashboard                      # Open Fly Web UI
fly dashboard metrics              # Open metrics dashboard
fly status --watch                 # Watch deployment/health in terminal
```

Fly also ships logs to third-party providers via log drains (Datadog, Papertrail, Logtail, etc.) — configure in the dashboard under **Monitoring → Log Shippers**.

---

### 14. Postgres

**Create a managed Postgres cluster:**
```bash
fly postgres create                         # Interactive
fly postgres create --name my-pg --region iad --vm-size shared-cpu-1x --volume-size 10
```

**Connect an app to Postgres:**
```bash
fly postgres attach my-pg --app my-app     # Sets DATABASE_URL secret automatically
fly postgres detach my-pg --app my-app
```

**Manage the cluster:**
```bash
fly postgres list
fly postgres status  --app my-pg
fly postgres connect --app my-pg           # psql shell
fly postgres db list --app my-pg
fly postgres db create mydb --app my-pg
fly postgres users list --app my-pg

fly postgres failover --app my-pg          # Promote a replica to primary
fly postgres backup list --app my-pg
fly postgres backup restore --app my-pg    # Restore from snapshot
```

**Proxy Postgres to your local machine:**
```bash
fly proxy 5432 --app my-pg
# Then: psql postgres://postgres:<password>@localhost:5432/<db>
```

Get the password: `fly secrets list --app my-pg` shows `OPERATOR_PASSWORD`.

---

### 15. Extensions and add-ons

```bash
fly extensions list                         # Show available extensions
fly extensions tigris create my-bucket     # Object storage (Tigris)
fly extensions sentry create               # Error tracking
fly extensions upstash redis create        # Redis
fly extensions neon create                 # Postgres (Neon)
fly extensions planetscale create          # Managed MySQL
```

Extensions auto-set the relevant secret (connection string) on your app.

---

### 16. Regions

```bash
fly platform regions                       # List all available regions + capacity

# Key regions:
# iad — Ashburn, VA (US East, default)
# ord — Chicago, IL
# lax — Los Angeles, CA
# lhr — London, UK
# ams — Amsterdam, NL
# fra — Frankfurt, DE
# cdg — Paris, FR
# nrt — Tokyo, JP
# sin — Singapore
# syd — Sydney, AU
# gru — São Paulo, BR
# bom — Mumbai, IN
# jnb — Johannesburg, ZA
```

Set the primary region in fly.toml with `primary_region = "lhr"`. This also sets the `PRIMARY_REGION` env var at runtime, which you can use to route writes to the right region.

---

### 17. Rollbacks

There is no single `fly rollback` command. Rollback by redeploying a previous image:

```bash
fly releases --image             # Find the image ref for the good release
fly deploy --image registry.fly.io/my-app:<previous-tag>
```

Use `--image-label` at deploy time to tag releases with readable names:
```bash
fly deploy --image-label v1.4.2
```

---

### 18. CI/CD with GitHub Actions

**.github/workflows/deploy.yml:**
```yaml
name: Deploy to Fly.io
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    concurrency: deploy-${{ github.ref }}
    steps:
      - uses: actions/checkout@v4
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - run: fly deploy --remote-only
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

**Generate a deploy token (app-scoped, safer than org token):**
```bash
fly tokens create deploy --name "github-main" --expiry 8760h
```

Add the output as a GitHub Actions secret named `FLY_API_TOKEN`.

**Multi-environment (staging + production):**
```yaml
# staging workflow — deploy to staging app
- run: fly deploy --config fly.staging.toml --app my-app-staging --remote-only

# production workflow — deploy to production app
- run: fly deploy --config fly.toml --app my-app --remote-only
```

---

### 19. Multi-environment management

**Pattern 1: Separate fly.toml files**
```
fly.toml             # production
fly.staging.toml     # staging
```
Deploy with: `fly deploy --config fly.staging.toml`

**Pattern 2: Separate apps in the same org**
```bash
fly launch --name my-app-staging --no-deploy
fly launch --name my-app --no-deploy
```

**Pattern 3: Separate organizations** (strongest isolation)
```bash
fly orgs create my-company-staging
fly deploy --org my-company-staging
```

Use app-scoped deploy tokens per environment to limit blast radius.

---

### 20. Useful patterns and tips

**Run a one-off task (migration, seed, etc.):**
```bash
fly console --command "bin/rails db:seed"
fly machine run --entrypoint "node scripts/migrate.js" registry.fly.io/my-app:latest
```

**Set release command for automatic migrations:**
```toml
[deploy]
  release_command = "bin/rails db:migrate"
```

**Environment-specific secrets:**
```bash
fly secrets set KEY=staging-value --app my-app-staging
fly secrets set KEY=prod-value    --app my-app
```

**Check app config that is currently deployed:**
```bash
fly config show
fly config display   # Show resolved, merged config
```

**Save config from a running app:**
```bash
fly config save
```

**Validate fly.toml before deploying:**
```bash
fly config validate
```

**Wireguard VPN (access private network locally):**
```bash
fly wireguard create                 # Create config
fly wireguard list
# Import the generated config into your WireGuard client
```

**Move an app to another region:**
```bash
# Update primary_region in fly.toml, then:
fly deploy
fly scale count 0 --region old-region   # Remove old Machines
```

**Watch what's happening during deploy:**
```bash
fly deploy && fly logs   # Deploy then tail logs
fly status --watch       # Poll status
```
