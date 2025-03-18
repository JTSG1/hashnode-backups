---
title: "Quick tip: using curl to pipe into docker compose"
datePublished: Tue Mar 18 2025 16:16:45 GMT+0000 (Coordinated Universal Time)
cuid: cm8ep4pko000308ju3f6aahlt
slug: quick-tip-using-curl-to-pipe-into-docker-compose
tags: docker, github, docker-compose

---

Ever needed to spin up a Docker Compose setup from a GitHub repo, but didn't want to clone the whole thing? I recently ran into this with Netflix Conductor. Instead of cloning or manually copying, try this:  
  
Turns out we can use curl and pipe the contents into docker compose

```bash
curl https://raw.githubusercontent.com/Netflix/conductor/refs/heads/main/docker/docker-compose-postgres.yaml | docker compose -f - up -d
```

Just replace the URL with your desired `docker-compose.yml` location. Saves time and space!