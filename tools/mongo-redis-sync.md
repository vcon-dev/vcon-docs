---
description: Archived script that copies vCons from Redis into MongoDB; read this before choosing it for a new deployment.
---

# 🔁 Redis to Mongo Sync

**Repo:** [vcon-dev/mongo-redis-sync](https://github.com/vcon-dev/mongo-redis-sync)

{% hint style="warning" %}
This repository is archived on GitHub and is read-only. For new deployments, configure a Mongo storage in the conserver instead. See [Storage](../conserver/storage.md).
{% endhint %}

The script copies in one direction only, from Redis to MongoDB. Despite the repo name it does not sync Mongo back into Redis.

- It scans Redis keys matching `vcon:*` at startup and writes each RedisJSON document to a Mongo collection with `replace_one(..., upsert=True)`, using the Redis key as `_id`.
- It then subscribes to Redis keyspace notifications and repeats the write on `set`, `hset` and `json.set` events. Deletions and expirations are not propagated.

Redis needs keyspace notifications enabled (`notify-keyspace-events ExA`) and RedisJSON. Settings come from environment variables, and the repo ships a Dockerfile and compose file.

## See also

- [Conserver Storage](../conserver/storage.md)
- [Production Deployment](../conserver/production-deployment.md)
