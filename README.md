# Twigex

**One space for all your work — files, projects, chat, video, and data, self-hosted.**

Twigex is an all-in-one private digital workspace created by [Twigex](https://twigex.com). Instead of switching between a file drive, a task tracker, a chat app, and a BI tool, teams get one platform for content, processes, and data — deployable on your own Kubernetes cluster with full data isolation.

[![Try the demo](https://img.shields.io/badge/demo-try%20it%20now-1976d2)](https://cospacedemo.twigex.com)
[![Artifact Hub](https://img.shields.io/badge/Artifact%20Hub-cospace-417598)](https://artifacthub.io/packages/helm/cospace/cospace)
[![Reddit](https://img.shields.io/badge/community-r%2Fcospace-FF4500)](https://www.reddit.com/r/cospace/)

![Cospace workspace screenshot](https://cospace.mytwigex.com/screenshot_2025-11-05_at_18.48.10.png)

## What's inside

- **Files** — real-time co-editing, MS Office–compatible formats, version history, smart search & metadata tagging.
- **Projects** — customizable workspaces, Kanban/Table views, custom fields, task assignment, connected tables, no-code forms.
- **Chat & video** — messaging and calls in one secure space, linked directly to tasks and files.
- **Data** — integrate APIs/databases, clean and transform data, build dashboards, automate workflows.
- **Deployment flexibility** — cloud, hybrid, or fully self-hosted/on-premises.

Built for teams of any size and industry — marketing, HR, freelancers, dev teams, and government/public sector deployments that need full digital autonomy over their data.

## Try it first

No signup, no setup — same version you'd self-host:

**[👉 Try the live demo](https://cospacedemo.twigex.com)**

Username: `cospace`

Password: `cospace`

(Don't post personal data — it resets every 30 minutes.)

## Quickstart — self-host it

Requires Helm v3.8+ and a Kubernetes 1.24+ cluster.

```sh
helm install cospace oci://ghcr.io/twigex/helm-charts/cospace --version 0.0.34 \
  --namespace cospace --create-namespace \
  --set domain=example.com \
  --set cospace.admin.password=your_password \
  --set cospace.server.encryptKey=$(openssl rand -hex 16) \
  --set mariadb.password=your_mariadb_password \
  --set mariadb.rootPassword=your_mariadb_root_password \
  --set redis.auth.password=your_redis_password
```

Full install guide — including MariaDB deployment modes, ingress/TLS, LiveKit (voice/video), and EuroOffice/Collabora (office editing) — is in **[INSTALLATION.md](./INSTALLATION.md)** and the [Cospace manual](https://cospace.mytwigex.com/en/Installation).

## For AI coding agents

This repo ships agent-facing operating instructions:

- **[AGENTS.md](./AGENTS.md)** — rules, conventions, and guardrails for AI coding agents (Claude Code, etc.) working in this repo.
- **[SKILL.md](./SKILL.md)** — a Claude Skill covering install, upgrade, and troubleshooting for the Cospace Helm chart.
- **[llms.txt](./llms.txt)** — machine-readable summary and resource map for LLMs/agentic tools.

## Editions

- **Community Edition** — free, self-hosted, core collaboration functionality. This repo's Helm chart deploys this edition.
- **Commercial/Organization Edition** — on-prem, hybrid, or cloud with support, customization, and complete data isolation controls. [Contact Twigex](https://twigex.com) for pricing.

## Community & links

- 📖 Docs: [cospace.mytwigex.com](https://cospace.mytwigex.com)
- 🏢 About Twigex: [twigex.com](https://twigex.com)
- 💬 Reddit: [r/cospace](https://www.reddit.com/r/cospace/)
- 📦 Artifact Hub: [artifacthub.io/packages/helm/cospace/cospace](https://artifacthub.io/packages/helm/cospace/cospace)

## License

See [LICENSE](./LICENSE). *(Note: this license covers the Helm chart and deployment tooling in this repository — refer to [twigex.com](https://twigex.com) for the licensing terms of the Cospace application itself.)*


