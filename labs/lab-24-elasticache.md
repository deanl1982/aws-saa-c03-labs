# Lab 24 — Caching with Amazon ElastiCache

| | |
|---|---|
| **Module** | 6 — Databases |
| **Exam mapping** | Domain 3 · Task 3.3 (caching strategies) · Domain 2 · Task 2.1 (stateless apps / session store) · Domain 4 · Task 4.3 (offload DB) |
| **Difficulty** | ★★☆ Intermediate |
| **Time** | 60 minutes |
| **Estimated cost** | ~$0.05–0.10/hour (2 × cache.t4g.micro, or Serverless at its minimum) |

- [ ] **Lab complete**

[⬅ Previous lab](lab-23-dynamodb.md) · [Back to index](../README.md) · [Next lab ➡](lab-25-sqs-sns.md)

---

## Learning outcomes

When you finish this lab you will be able to:

- Deploy **ElastiCache for Valkey/Redis OSS** with a **replica** and **Multi-AZ automatic failover**.
- Implement the **lazy loading (cache-aside)** and **write-through** patterns, with **TTLs**.
- Use a cache as a **session store** so web tiers become stateless.
- Choose between **Valkey/Redis OSS**, **Memcached**, **DAX**, and **CloudFront** caching.

## Engine comparison

| | Valkey / Redis OSS | Memcached |
|---|---|---|
| Data types | Strings, hashes, lists, sets, **sorted sets** (leaderboards), streams | Simple strings |
| Replication / Multi-AZ failover | **Yes** | No |
| Persistence / backups | **Yes** (snapshots) | No |
| Multi-threaded | Valkey and Redis 6+ have I/O threads | **Yes** |
| Pub/Sub, Lua, transactions | Yes | No |
| Typical exam trigger | HA cache, sessions, leaderboards, pub/sub | Simple, multi-threaded, sharded cache with no HA needed |

## Caching patterns

| Pattern | How | Pros | Cons |
|---|---|---|---|
| **Lazy loading** | Read cache → on a miss, read the DB and populate the cache | Only caches requested data, and survives cache failure | First read is slow, data can be stale |
| **Write-through** | Write to the DB **and** the cache on every update | Cache is always fresh | Write penalty, caches data nobody reads |
| **TTL** | Expire keys after N seconds | Limits staleness and memory | Extra misses |

---

## Prerequisites

- `saa-vpc` data subnets and `saa-app-sg`.
- The `saa-db-client` instance from Lab 21 (public subnet, SG `saa-app-sg`). Recreate it with the Lab 21 prerequisite commands if you need to.

---

## Part A — Console

### A1. Security group and subnet group

- [ ] Create SG `saa-cache-sg` in `saa-vpc` with inbound **TCP 6379** from **`saa-app-sg`**.
- [ ] Open **ElastiCache → Subnet groups → Create**: `saa-cache-subnets` with **saa-data-a** and **saa-data-b**.

### A2. Create a node-based Valkey cluster with a replica

- [ ] Open **ElastiCache → Valkey caches → Create**.
  - Deployment option: **Node-based cluster**, creation method **Cluster cache** (standard create)
  - Cluster mode **Disabled** (one shard)
  - Name `saa-valkey`, node type **cache.t4g.micro**, **1 replica**
  - **Multi-AZ: Enabled**, **Auto-failover: Enabled**
  - Subnet group `saa-cache-subnets`, SG `saa-cache-sg`
  - Encryption in transit **on** and at rest **on**. Access control **none**, or a user group for AUTH.
  - Backups: automatic, 1 day
- [ ] Wait about 10 minutes. Note the **primary endpoint** and the **reader endpoint**.

### A3. Connect and explore

- [ ] On `saa-db-client` (Session Manager), install the CLI client:

```bash
sudo dnf -y install valkey || sudo dnf -y install redis6
CLI=$(command -v valkey-cli || command -v redis6-cli)
$CLI -h <primary-endpoint> --tls -p 6379 PING
```

- [ ] Try the core data types:

```bash
$CLI -h <primary-endpoint> --tls SET greeting "hello" EX 60
$CLI -h <primary-endpoint> --tls TTL greeting
$CLI -h <primary-endpoint> --tls ZADD leaderboard 1200 alice 950 bob 1800 carol
$CLI -h <primary-endpoint> --tls ZREVRANGE leaderboard 0 2 WITHSCORES
$CLI -h <primary-endpoint> --tls HSET session:abc123 user alice cart 3
$CLI -h <primary-endpoint> --tls HGETALL session:abc123
```

### A4. Test failover

- [ ] Select the cluster and choose **Actions → Failover primary** (or *Test failover* on the primary node).
- [ ] Reconnect with the **primary endpoint**. It now points at the promoted replica, and your data is still there.

### A5. Compare with ElastiCache Serverless (read-through)

- [ ] Open **Create → Serverless** and look at the options. There's no node sizing, it scales automatically, and it's billed per GB-hour and per ECPU. **Don't create it** unless you want to compare costs.

---

## Part B — CLI

```bash
source ~/saa-vpc.env
CACHE_SG=$(aws ec2 describe-security-groups --filters Name=group-name,Values=saa-cache-sg --query 'SecurityGroups[0].GroupId' --output text)
```

### B1. Describe the replication group

- [ ] Look at the endpoints and failover settings:

```bash
aws elasticache describe-replication-groups --replication-group-id saa-valkey \
  --query 'ReplicationGroups[0].{Status:Status,MultiAZ:MultiAZ,Failover:AutomaticFailover,Primary:NodeGroups[0].PrimaryEndpoint.Address,Reader:NodeGroups[0].ReaderEndpoint.Address,Members:NodeGroups[0].NodeGroupMembers[].{Node:CacheClusterId,Role:CurrentRole,AZ:PreferredAvailabilityZone}}'
```

### B2. Create a Memcached cluster from the CLI (for comparison)

- [ ] Create it:

```bash
aws elasticache create-cache-cluster --cache-cluster-id saa-memcached \
  --engine memcached --cache-node-type cache.t4g.micro --num-cache-nodes 2 \
  --az-mode cross-az --cache-subnet-group-name saa-cache-subnets \
  --security-group-ids $CACHE_SG
```

> Memcached spreads keys across nodes (sharding) **without replication**. If a node is lost, its keys are lost.

### B3. Lazy loading vs write-through in code

- [ ] On `saa-db-client`, install the Python client and create a demo. A `sleep` simulates a slow database query:

```bash
sudo dnf -y install python3-pip && pip3 install --user redis
cat > cache_demo.py <<'EOF'
import time, json, sys, redis
r = redis.Redis(host=sys.argv[1], port=6379, ssl=True, decode_responses=True)
FAKE_DB = {"p1": {"name": "Keyboard", "price": 49}, "p2": {"name": "Mouse", "price": 19}}

def db_get(pid):
    time.sleep(0.5)                       # pretend this is an expensive SQL query
    return FAKE_DB.get(pid)

def get_product_lazy(pid):                # cache-aside / lazy loading
    cached = r.get(f"product:{pid}")
    if cached:
        return json.loads(cached), "HIT"
    item = db_get(pid)
    r.set(f"product:{pid}", json.dumps(item), ex=30)   # TTL 30s
    return item, "MISS"

def update_product_write_through(pid, price):
    FAKE_DB[pid]["price"] = price          # write to DB
    r.set(f"product:{pid}", json.dumps(FAKE_DB[pid]), ex=30)  # and to cache

for i in range(3):
    t = time.time(); item, s = get_product_lazy("p1")
    print(f"lazy read {i}: {s} {item} {1000*(time.time()-t):.1f} ms")

update_product_write_through("p1", 39)
t = time.time(); item, s = get_product_lazy("p1")
print(f"after write-through: {s} {item} {1000*(time.time()-t):.1f} ms")
EOF
python3 cache_demo.py <primary-endpoint>
```

> Expected: the first read is a **MISS** (~500 ms), later reads are **HITs** (~1 ms), and after the write-through update the read is a fresh **HIT** with the new price.

### B4. Scale the replica count and take a snapshot

- [ ] Add a replica, then take a snapshot:

```bash
aws elasticache increase-replica-count --replication-group-id saa-valkey --new-replica-count 2 --apply-immediately
aws elasticache create-snapshot --replication-group-id saa-valkey --snapshot-name saa-valkey-snap
aws elasticache describe-snapshots --snapshot-name saa-valkey-snap --query 'Snapshots[0].SnapshotStatus'
```

### B5. Look at the key CloudWatch metrics

- [ ] Check cache hits:

```bash
NODE=$(aws elasticache describe-replication-groups --replication-group-id saa-valkey --query 'ReplicationGroups[0].MemberClusters[0]' --output text)
aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name CacheHits \
  --dimensions Name=CacheClusterId,Value=$NODE --statistics Sum --period 300 \
  --start-time $(date -u -v-1H +%FT%TZ 2>/dev/null || date -u -d '1 hour ago' +%FT%TZ) --end-time $(date -u +%FT%TZ)
```

> Other metrics worth watching: `CacheMisses`, `Evictions` (the cache is too small), `CPUUtilization` / `EngineCPUUtilization`, and `ReplicationLag`.

---

## Verify

- [ ] You read and wrote strings, hashes, and sorted sets over TLS.
- [ ] After a failover, the primary endpoint still works and the data is intact.
- [ ] The Python demo shows MISS, then HIT, with a big latency difference.

## Exam tips

- **"Reduce read load on RDS / improve read latency for repeated queries"** → **ElastiCache** (lazy loading with a TTL).
- **"Store user sessions so web servers can be stateless"** → **ElastiCache (Valkey/Redis)** or **DynamoDB**.
- **"Real-time leaderboard"** → **Redis/Valkey sorted sets**.
- **"HA cache with automatic failover"** → Valkey/Redis with **Multi-AZ + replicas**. Memcached can't do this.
- **DynamoDB-specific caching** → **DAX**. **Caching static/dynamic HTTP content at the edge** → **CloudFront**.
- A cache only helps if the app code uses it. It's never a drop-in fix with "no code changes" (DAX is the closest, since it's API-compatible).

## Challenge (optional)

- [ ] Turn on **cluster mode** (several shards) and explain how it differs from adding replicas: shards scale **writes and memory**, replicas scale **reads and HA**.

## Cleanup

- [ ] Delete the caches and the snapshot:

```bash
aws elasticache delete-replication-group --replication-group-id saa-valkey --no-retain-primary-cluster
aws elasticache delete-cache-cluster --cache-cluster-id saa-memcached
aws elasticache delete-snapshot --snapshot-name saa-valkey-snap
```

- [ ] When they're gone, delete the `saa-cache-subnets` subnet group and the `saa-cache-sg` security group.
- [ ] Terminate `saa-db-client` if you've finished with the database labs:

```bash
aws ec2 terminate-instances --instance-ids $CLIENT
```

---

## Knowledge check

**Q1.** A product catalogue on RDS gets 95% reads, and the same products are requested repeatedly. Database CPU is at 90%. What's the most effective fix?

<details><summary>Answer</summary>

Add **ElastiCache** using **lazy loading with TTLs**, so repeated reads are served from memory. Read replicas also help, but caching gives lower latency.

</details>

**Q2.** A web application stores sessions in instance memory, so users get logged out when the ASG scales in. What should you change?

<details><summary>Answer</summary>

Externalise sessions to **ElastiCache (Valkey/Redis)** or **DynamoDB** so any instance can serve any user.

</details>

**Q3.** A cache must survive the loss of a node or an AZ without losing data or needing manual work. Which engine and configuration?

<details><summary>Answer</summary>

**Valkey / Redis OSS** with **replicas**, **Multi-AZ**, and **automatic failover** turned on.

</details>
