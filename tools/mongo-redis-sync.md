---
description: Keep a MongoDB vCon store and a Redis cache in sync — useful for hybrid deployments.
---

# 🔁 Mongo ↔ Redis Sync

**Repo:** [vcon-dev/mongo-redis-sync](https://github.com/vcon-dev/mongo-redis-sync)

A small utility that mirrors vCons between a MongoDB long-term store and a Redis cache. Useful when:

- You're using Mongo as durable storage but the conserver pipeline expects vCons in Redis.
- You're migrating a deployment between the two and want bidirectional sync during the transition.
- You want hot vCons in Redis for low-latency reads while keeping the canonical copy in Mongo.

This tool predates much of the current conserver storage layer and is most useful for deployments that pre-date the unified Supabase backend used by `vcon-mcp`. For new deployments, prefer the conserver's native storage configuration — see [Storage](../conserver/storage.md).

## See also

- [Conserver Storage](../conserver/storage.md)
- [Production Deployment](../conserver/production-deployment.md)
