---
title: "Quick tip: using curl to pipe into docker compose"
seoTitle: "Save Time: Use curl to Run Docker Compose Without Cloning the Repo"
seoDescription: "Skip the git clone! Learn how to quickly launch Docker Compose environments from a GitHub repo using curl—no local copy needed. Ideal for quick testing,"
datePublished: Mon Jun 16 2025 23:00:00 GMT+0000 (Coordinated Universal Time)
cuid: cm8ep4pko000308ju3f6aahlt
slug: quick-tip-using-curl-to-pipe-into-docker-compose
canonical: https://tesh.digital/quick-tip-using-curl-to-pipe-into-docker-compose
tags: docker, github, tips, devops, containers, docker-compose, quicktips

---

Ever needed to spin up a Docker Compose setup from a GitHub repo, but didn't want to clone the whole thing? I recently ran into this with Netflix Conductor. Instead of cloning or manually copying, try this:

Turns out we can use curl and pipe the contents into docker compose

```bash
curl https://raw.githubusercontent.com/Netflix/conductor/refs/heads/main/docker/docker-compose-postgres.yaml | docker compose -f - up -d
```

Just replace the URL with your desired `docker-compose.yml` location. Saves time and space!

*The -f - flag tells Docker Compose to read the file from* ***stdin****, so nothing ever hits disk.*

Only run this against YAML you trust; you’re effectively ‘curl-piping’ commands that can start arbitrary containers