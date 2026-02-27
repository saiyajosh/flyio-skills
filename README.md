# flyio-skills

Agent skills for working with [Fly.io](https://fly.io) — the platform for running full-stack apps and databases close to your users.

## Skills

### `flyctl`

A full-featured skill for using [flyctl](https://fly.io/docs/flyctl/), Fly.io's CLI tool.

**Covers:**
- Installation and authentication
- App creation (`fly launch`) and deployment (`fly deploy`)
- `fly.toml` configuration reference (build, services, health checks, deploy strategies)
- Machine and VM management
- Horizontal and vertical scaling
- Secrets management
- Persistent volumes
- Networking: IP allocation, custom domains, TLS certificates, private networking, `fly proxy`
- SSH and console access
- Logs and monitoring
- Managed Postgres (create, attach, backups, failover, local proxy)
- Extensions (Tigris, Upstash Redis, Neon, Sentry, PlanetScale)
- Regions and placement
- Rollbacks
- CI/CD with GitHub Actions
- Multi-environment management (staging vs. production)

## Usage

Install a skill by copying the contents of a `SKILL.md` file into your Claude Code project's skills configuration, or reference it directly as a custom skill.

## Structure

```
skills/
  flyctl/
    SKILL.md
```
