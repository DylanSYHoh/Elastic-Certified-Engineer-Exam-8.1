# Cluster Management (8.15)

**8.15 objectives covered here:**
1. Diagnose shard issues and repair a cluster's health
2. Backup and restore a cluster and/or specific indices
3. Configure a snapshot to be searchable
4. Configure a cluster for cross-cluster search
5. Implement cross-cluster replication
6. Automate snapshots with Snapshot Lifecycle Management :sparkles: *(new since 8.1)*

> :books: "Define role-based access control using Elasticsearch Security" is not a listed objective; it stays at the bottom as bonus material.

---

# Diagnose shard issues and repair a cluster's health

:bulb: Pre-requisites:  Add a broken index (this is more about the diagnosis commands than the actual broken index)

```json
# delete if already in place
DELETE broken_index

# add a new "broken" index
PUT broken_index
{
  "settings": {
    "number_of_replicas": 2,
    "number_of_shards": 2
  }
}

# add a document
PUT broken_index/_doc/1
{
  "field1" : "data"
}
      
```

## Check Health

:question: Check The Cluster Health

<details>
  <summary>View Solution (click to reveal)</summary>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cluster-health.html

```json
GET _cluster/health

// output 

{
  "cluster_name" : "elastic-cluster",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 48,
  "active_shards" : 48,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 16,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 75.0
}
```

## Or
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cat-health.html
```json
GET _cat/health?v

// output

epoch      timestamp cluster         status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1633541301 17:28:21  elastic-cluster yellow          1         1     48  48    0    0       16             0                  -                 75.0%

```
</details>
<hr>

## Find Broken Indices

index health usually matches the cluster health

<details>
  <summary>View Solution (click to reveal)</summary>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cat-indices.html

> **Query Parameters**
> **health**
> (Optional, string) Health status used to limit returned indices. Valid values are:
>
> - green
> - yellow
> - red

```json
GET _cat/indices?health=yellow

// output

yellow open broken_index               xHfY0EcGRq-RRZv_cQONCw 2 2      1 0     4kb     4kb

```
</details>
<hr>

## :sparkles: The Health API — start here in 8.15

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/health-api.html

Added in 8.7 and **the fastest route to a diagnosis** on a modern cluster: it does not just tell you the status, it tells you the *cause* and gives you the remediation steps and the doc link.

```json
GET _health_report

// drill into just one indicator
GET _health_report/shards_availability
GET _health_report/disk
GET _health_report/ilm
```

Indicators it reports on: `master_is_stable`, `repository_integrity`, `disk`, `shards_capacity`, `shards_availability`, `data_stream_lifecycle`, `ilm`, `slm`.

Each unhealthy indicator returns a `diagnosis` array containing `cause`, `action`, `help_url`, and the affected resources. If a task says "diagnose and repair", run this first, then confirm with `_cluster/allocation/explain`.

<hr>

:question: Diagnose the fault in index `broken_index`

<details>
  <summary>View Solution (click to reveal)</summary>

Explain the allocation of a specific shard.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cluster-allocation-explain.html

:warning: You must specify `index`, `shard` **and** `primary` — otherwise you get
`Validation Failed: 1: shard must be specified;2: primary must be specified;`

```json
GET _cluster/allocation/explain
{
  "index": "broken_index",
  "shard": 0,
  "primary": false
}
```

:bulb: With an empty body, Elasticsearch picks an arbitrary unassigned shard for you — which is usually exactly what you want:

```json
GET _cluster/allocation/explain
```

Read these fields in the response:
- `unassigned_info.reason` — `INDEX_CREATED`, `ALLOCATION_FAILED`, `NODE_LEFT`, `CLUSTER_RECOVERED`…
- `can_allocate` — `no`, `yes`, `throttled`, `awaiting_info`
- `allocate_explanation` — plain English
- `node_allocation_decisions[].deciders[]` — **the actual reason each node refused the shard**

Common decider messages and their fix:

| Decider says | Real cause | Fix |
| --- | --- | --- |
| `the shard cannot be allocated to the same node on which a copy of the shard already exists` | more replicas than nodes | reduce `number_of_replicas` or add a node |
| `the node is above the high watermark cluster setting [...disk.watermark.high]` | disk full | free space, or raise the watermark |
| `node does not match index setting [index.routing.allocation...]` | tier/attribute filter cannot be satisfied | fix the routing setting, or add a node with that role |
| `shard has exceeded the maximum number of retries [5]` | repeated allocation failure | fix the root cause, then `POST _cluster/reroute?retry_failed=true` |
</details>
<hr>

:question:  View the shard allocation for the index

<details>
  <summary>View Solution (click to reveal)</summary>

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cat-shards.html

```json

GET _cat/shards/broken_index?v&s=index

// output

index        shard prirep state      docs store ip         node
broken_index 1     p      STARTED       0  208b 172.20.0.2 esnode
broken_index 1     r      UNASSIGNED                       
broken_index 1     r      UNASSIGNED                       
broken_index 0     p      STARTED       1 3.8kb 172.20.0.2 esnode
broken_index 0     r      UNASSIGNED                       
broken_index 0     r      UNASSIGNED    

```

Here we can see that a shard is `UNASSIGNED`.

Why is that?

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cluster-allocation-explain.html

A lot of information is presented here, if you have a larger underlying issue you will see many explanations across many indices.  Try to keep that in mind.

For each index and then each node, read the explanations

</details>
<hr>



## Repair

:question: Repair by reducing the number of replicas required.  This matches the number of replica nodes available.  In this case 0 as we are running on a single node cluster.

```json
PUT /broken_index/_settings
{
    "number_of_replicas": 0
    
}

// output

{
  "acknowledged" : true
}
```

Check the index health again


```json
GET _cat/indices/broken_index?v

// output

health status index        uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   broken_index xHfY0EcGRq-RRZv_cQONCw   2   0          1            0        4kb            4kb

```

## The repair toolbox

What the colours mean:

| Status | Meaning |
| --- | --- |
| **green** | all primaries and all replicas assigned |
| **yellow** | all primaries assigned, at least one **replica** unassigned — *data is intact, availability is reduced* |
| **red** | at least one **primary** unassigned — *data is unavailable* |

:bulb: A single-node cluster with `number_of_replicas > 0` is **permanently yellow**, by design. On the exam, "make the cluster green" on a one-node cluster almost always means "set replicas to 0".

```json
// yellow: too many replicas for the number of nodes
PUT /broken_index/_settings
{ "number_of_replicas": 0 }

// apply to every index at once
PUT /*/_settings
{ "index.number_of_replicas": 0 }

// shard failed 5 times and gave up - fix the cause, then:
POST _cluster/reroute?retry_failed=true

// find which indices are the problem
GET _cat/indices?v&health=red&h=index,health,status,pri,rep,docs.count
GET _cat/shards?v&h=index,shard,prirep,state,unassigned.reason,node&s=state

// disk watermarks - the #1 real-world cause of unassigned shards
GET _cat/allocation?v
GET _cluster/settings?include_defaults=true&flat_settings&filter_path=**.disk.watermark*

PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low":  "85%",
    "cluster.routing.allocation.disk.watermark.high": "90%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
  }
}

// flood stage puts indices into read-only-allow-delete; clear the block after freeing space
PUT /*/_settings
{ "index.blocks.read_only_allow_delete": null }

// is allocation switched off entirely?
PUT _cluster/settings
{ "persistent": { "cluster.routing.allocation.enable": "all" } }

// pending tasks / hot threads when the cluster is just slow
GET _cluster/pending_tasks
GET _nodes/hot_threads
GET _cat/nodes?v&h=name,node.role,heap.percent,ram.percent,cpu,disk.avail
```

:warning: Last resort only, and it **loses data**: `POST _cluster/reroute` with `allocate_empty_primary` / `allocate_stale_primary` and `"accept_data_loss": true`. Know it exists; do not reach for it before the health API and allocation explain.

<hr>

# Backup and restore a cluster and/or specific indices

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshots-take-snapshot.html

> You cannot back up an Elasticsearch cluster by simply taking a copy of the data directories of all of its nodes. 
> 
> Elasticsearch may be making changes to the contents of its data directories while it is running; copying its data directories cannot be expected to capture a consistent picture of their contents. 
>
> If you try to restore a cluster from such a backup, it may fail and report corruption and/or missing files. Alternatively, it may appear to have succeeded though it silently lost some of its data. 
> 
> The only reliable way to back up a cluster is by using the snapshot and restore functionality.

## To have a complete backup for your cluster:

- Back up the data
- Back up the cluster configuration
- Back up the security configuration

## To restore your cluster from a backup:

- Restore the data
- Restore the security configuration

### Version Compatibility
> Version compatibility refers to the underlying Lucene index compatibility. Follow the Upgrade documentation when migrating between versions.
>
> A snapshot contains a copy of the on-disk data structures that make up an index. This means that snapshots can only be restored to versions of Elasticsearch that can read the indices.

The rule to remember: **a snapshot can be restored to the same major version, or to the next major version — one hop only.**

- An index created in 7.x can be restored into 8.x. :white_check_mark:
- An index created in 6.x **cannot** be restored directly into 8.x. :x: (You must reindex it in 7.x first, or use an archive index.)
- You cannot restore a snapshot taken by a *newer* Elasticsearch into an older one at all.

### Repositories

> You must register a snapshot repository before you can perform snapshot and restore operations. We recommend creating a new snapshot repository for each major version. The valid repository settings depend on the repository type.

| Type | Notes |
| --- | --- |
| `fs` | shared filesystem — needs `path.repo` in `elasticsearch.yml` on **every** node, and the path must be identical everywhere |
| `s3`, `gcs`, `azure` | need the corresponding repository plugin (bundled in 8.x) plus credentials in the keystore |
| `url` | read-only, over http/https/file/ftp |
| `source` | stores only `_source` — smaller, but needs a reindex to restore |

```json
GET _snapshot                                    // all registered repositories
POST _snapshot/my_test_backup/_verify            // can every node actually reach it?
POST _snapshot/my_test_backup/_cleanup           // remove orphaned data
GET _snapshot/my_test_backup/_status             // in-progress snapshot progress
```


:question: Backup and restore an index using `snapshots` <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/modules-snapshots.html (references other documentation)  <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshot-restore.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshots-register-repository.html#self-managed-repo-types <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/restore-snapshot-api.html#restore-snapshot-api-index-settings <br>

1. :question: Backup the `shakespeare` index to a snapshot called `shakespeare_snapshot_<current_date>`

<details>
  <summary>View Solution (click to reveal)</summary>

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshot-restore.html

You will need to make sure that the `path.repo` setting has been applied to each ElasticSearch node before doing this.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshots-register-repository.html

The docker images in this Git Repository have this set in the `1es-1kb-xpackSec.yml` single node cluster.  Which was used predominantly throughout the other sections.

Normally you would save the snapshots to share storage like NFS, AWS S3 etc.   In this demo we use the local filesystem `/tmp` this is not recommended in production.


This can all be done in the kibana GUI https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshot-restore.html

## Check that path.repo is set

```json
GET /_nodes?pretty&filter_path=nodes.*.settings.path

// output

{
  "nodes" : {
    "HKbyLT8xRMC08bJO32XNFg" : {
      "settings" : {
        "path" : {
          "logs" : "/usr/share/elasticsearch/logs",
          "home" : "/usr/share/elasticsearch",
          "repo" : "/tmp"
        }
      }
    }
  }
}
```
As you can see the `path.repo` is set to `/tmp`.

## Register the backup location
```json
PUT /_snapshot/my_test_backup
{
    "type": "fs",
    "settings": {
        "location": "/tmp/test1",
        "compress": true
    }
}

// output

{
  "acknowledged" : true
}
```
Notice that `/tmp` needed to be available and that you can then append a path to that.  eg. `/tmp/test1`


## Make a snapshot, to that registered location

Date math requires the snapshot name to be enclosed in angled brackets '<>' that is: '%3C' '%3E'

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/api-conventions.html#api-date-math-index-names
  > You must enclose date math names in angle brackets. If you use the name in a request path, `special characters must be URI encoded.`

```json
PUT /_snapshot/my_test_backup/%3Cshakespeare-snapshot-%7Bnow%2Fd%7D%3E
{
  "indices": "shakespeare",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

## Check the snapshot 

List all snapshots

```json
GET /_snapshot/my_test_backup/_all

// or

GET /_snapshot/my_test_backup/*

// output 

{
  "snapshots" : [
    {
      "snapshot" : "shakespeare-snapshot-2021.05.13",
      "uuid" : "t0Qk_gDtT1uqvmrbNg7yIQ",
      "version_id" : 7020099,
      "version" : "7.2.0",
      "indices" : [
        "shakespeare"
      ],
      "include_global_state" : false,
      "state" : "SUCCESS",
      "start_time" : "2021-05-13T18:59:27.117Z",
      "start_time_in_millis" : 1620932367117,
      "end_time" : "2021-05-13T18:59:28.050Z",
      "end_time_in_millis" : 1620932368050,
      "duration_in_millis" : 933,
      "failures" : [ ],
      "shards" : {
        "total" : 1,
        "failed" : 0,
        "successful" : 1
      }
    }
  ]
}
```

</details>
<hr>

2. :question: Restore the `shakespeare_snapshot_<current_date>` index snapshot to the name `restored_index_shakespeare`

<details>
  <summary>View Solution (click to reveal)</summary>

```json
POST /_snapshot/my_test_backup/shakespeare-snapshot-2021.05.13/_restore
{
  "indices": "shakespeare",
  "ignore_unavailable": true,
  "include_global_state": true,
  "rename_pattern": "(.+)",
  "rename_replacement": "restored_index_$1"
}

// output

{
  "accepted" : true
}
```

Check the restored index

```json
GET _cat/indices/*shakespeare

// output

health status index                      uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   shakespeare                h21mMC7ZRWGB1OPjgW0VuQ   1   1     111396            0     20.5mb         20.5mb
yellow open   restored_index_shakespeare gKIdyU4jSnqmuBLRqPpZLw   1   1     111396            0     20.5mb         20.5mb
```

:warning: Health is yellow due to the number of replicas is above 0 for a single node deployment.

</details>
<hr>

3. :closed_book: Backup/Restore the cluster configuration

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshots-take-snapshot.html#back-up-config-files
> We recommend that you take regular (ideally, daily) backups of your Elasticsearch config ($ES_PATH_CONF) directory using the file backup software of your choice.

Normally `/etc/elasticsearch`

> Some settings in configuration files might be overridden by cluster settings. You can capture these settings in a data backup snapshot by specifying the include_global_state: true (default) parameter for the snapshot API. 
>
>Alternatively, you can extract these configuration values in text format by using the get settings API:

```json
GET _cluster/settings?pretty&flat_settings&filter_path=persistent
```

So, it's a probably good idea to backup your `/etc/elasticsearch/` folder and run an external API call to download the persistent settings to a text file.

4. :closed_book: Backup/Restore the Security configuration

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/security-backup.html (just a reference to the following 2 links)
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshots-take-snapshot.html#back-up-config-files
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshots-take-snapshot.html#cluster-state-snapshots  
## Back up file-based security configuration
> Elasticsearch security features are configured using the xpack.security namespace inside the elasticsearch.yml and elasticsearch.keystore files. In addition there are several other extra configuration files inside the same ES_PATH_CONF directory. These files define roles and role mappings and configure the file realm. 

## Back up index-based security configuration

> Elasticsearch security features store system configuration data inside a dedicated index. This index is named .security-6 in the Elasticsearch 6.x versions and .security-7 in the 7.x releases. The .security alias always points to the appropriate index. This index contains the data which is not available in configuration files and cannot be reliably backed up using standard filesystem tools. This data describes:
>
> - the definition of users in the native realm (including hashed passwords)
> - role definitions (defined via the create roles API)
> - role mappings (defined via the create role mappings API)
> - application privileges
> - API keys
>
> The .security index thus contains resources and definitions in addition to configuration information. All of that information is required in a complete security features backup.

> Use the standard Elasticsearch snapshot functionality to backup .security, as you would for any other data index.
>
> Snapshot the .security index in a dedicated repository, where read and write access is strictly restricted and audited.

So, backup the `/etc/elasticsearch` folder,

and

snapshot the `.security` index alias.


## Restore the Security index

See https://www.elastic.co/guide/en/elasticsearch/reference/8.15/restore-security-configuration.html

## Restore options worth knowing

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/restore-snapshot-api.html

```json
POST /_snapshot/my_test_backup/my-snapshot/_restore
{
  "indices": "shakespeare,accounts-*",
  "ignore_unavailable": true,
  "include_global_state": false,
  "rename_pattern": "(.+)",
  "rename_replacement": "restored_index_$1",
  "include_aliases": false,
  "index_settings": {
    "index.number_of_replicas": 0,
    "index.blocks.read_only": false
  },
  "ignore_index_settings": [ "index.refresh_interval" ]
}
```

:warning: **You cannot restore over an open index of the same name.** If a task says "restore `shakespeare` back over itself", you must first either:
```json
POST shakespeare/_close      // then restore, then _open
// or
DELETE shakespeare           // then restore
```
Restoring into a *renamed* index (as above) sidesteps this entirely and is usually the safer exam answer.

Monitor a restore in progress:
```json
GET _cat/recovery?v&active_only=true
GET _cluster/health?wait_for_status=green&timeout=60s
```

Other snapshot housekeeping:
```json
GET _snapshot/my_test_backup/my-snapshot                 // details of one snapshot
GET _snapshot/my_test_backup/_current                    // what's running right now
DELETE _snapshot/my_test_backup/my-snapshot              // delete a snapshot (also cancels if running)
DELETE _snapshot/my_test_backup                          // unregister the repository (does NOT delete data)
```

---

# :sparkles: Automate snapshots with Snapshot Lifecycle Management (SLM)

**New objective in 8.15** — it was not on the 8.1 list, so make sure you can do this cold.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshots-take-snapshot.html#automate-snapshots-slm <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/slm-api-put-policy.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/getting-started-snapshot-lifecycle-management.html

> SLM lets you automatically take snapshots at a set schedule and automatically delete them when they age out, so you don't have to script it yourself.

## Prerequisites

1. A **registered repository** (SLM cannot create one for you).
2. The `manage_slm` cluster privilege, plus `manage` on the indices you snapshot. (The built-in `slm-history-ilm-policy` handles the audit trail.)

## The anatomy of a policy

```json
PUT _slm/policy/<policy_id>
{
  "schedule":   "<cron expression>",
  "name":       "<snapshot name, supports date math>",
  "repository": "<registered repository>",
  "config":     { ...same body as the snapshot API... },
  "retention":  { ... }
}
```

| Field | Notes |
| --- | --- |
| `schedule` | **Cron with seconds**: `<sec> <min> <hour> <day-of-month> <month> <day-of-week> [year]`. `0 30 1 * * ?` = 01:30 UTC daily. (8.15 also accepts a simple time-unit interval like `"1d"`.) |
| `name` | Date math, URL-encoding **not** needed here because it's in the body: `<nightly-snap-{now/d}>`. A UUID is appended automatically so names never clash. |
| `repository` | must already exist |
| `config.indices` | array or pattern, e.g. `["*"]` or `["logs-*", "-logs-debug*"]` |
| `config.include_global_state` | `true` to include cluster settings, templates, ILM/SLM policies, and (with `feature_states`) the security index |
| `config.feature_states` | e.g. `["security", "kibana"]`, or `["none"]` |
| `config.partial` | `true` = snapshot succeeds even if some shards are unavailable |
| `retention.expire_after` | delete snapshots older than this |
| `retention.min_count` | never go below this many, even if expired |
| `retention.max_count` | never keep more than this many, even if not expired |

:warning: `retention` is only enforced by the **retention task**, which runs on its own schedule (`slm.retention_schedule`, default `0 30 1 * * ?` — 01:30 UTC daily), *not* at snapshot time.

<hr>

:question: Register a repository and create an SLM policy called `nightly-snapshots` that snapshots **all** indices plus the cluster state to it every day at 01:30 UTC, names snapshots `nightly-snap-<date>`, and keeps snapshots for 30 days, never fewer than 5 and never more than 50.

<details>
  <summary>View Solution (click to reveal)</summary>

### 1. Register the repository (if it doesn't exist)

```json
PUT _snapshot/my_repository
{
  "type": "fs",
  "settings": {
    "location": "/tmp/my_repository",
    "compress": true
  }
}

POST _snapshot/my_repository/_verify
```

### 2. Create the policy

```json
PUT _slm/policy/nightly-snapshots
{
  "schedule": "0 30 1 * * ?",
  "name": "<nightly-snap-{now/d}>",
  "repository": "my_repository",
  "config": {
    "indices": ["*"],
    "include_global_state": true
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}
```

### 3. Don't wait 24 hours — run it now

```json
POST _slm/policy/nightly-snapshots/_execute

// output
{ "snapshot_name": "nightly-snap-2026.08.14-abc123def456ghi789jkl" }
```

### 4. Verify

```json
GET _slm/policy/nightly-snapshots

// look for:
//   "next_execution"          - when it will next fire
//   "last_success.snapshot_name"
//   "last_failure"
//   "stats.snapshots_taken"

GET _snapshot/my_repository/_all?verbose=false
GET _cat/snapshots/my_repository?v
```
</details>
<hr>

## Cron quick reference (SLM cron has a **seconds** field — this trips people up)

| Expression | Fires |
| --- | --- |
| `0 30 1 * * ?` | every day at 01:30:00 |
| `0 0 */6 * * ?` | every 6 hours, on the hour |
| `0 15 2 ? * MON` | every Monday at 02:15 |
| `0 0 0 1 * ?` | first day of every month at midnight |
| `0 0/30 * * * ?` | every 30 minutes |

:bulb: `?` means "no specific value" and is required in **exactly one** of day-of-month / day-of-week — you cannot put `*` in both.

## The rest of the SLM API

```json
GET _slm/policy                                  // all policies + stats
GET _slm/policy/nightly-snapshots                // one policy
DELETE _slm/policy/nightly-snapshots             // delete the policy (NOT its snapshots)

POST _slm/policy/nightly-snapshots/_execute      // run now
POST _slm/_execute_retention                     // apply retention rules now

GET _slm/stats                                   // taken/failed/deleted counters
GET _slm/status                                  // RUNNING or STOPPED
POST _slm/stop                                   // pause SLM (e.g. before an upgrade)
POST _slm/start
```

:warning: Exam traps:
- Deleting an SLM policy does **not** delete the snapshots it already took.
- If `GET _slm/policy/x` shows `last_failure` with `repository_missing_exception`, you created the policy before the repository.
- SLM policies are part of cluster state, so `include_global_state: true` in a snapshot backs up your SLM policies too.
- `_health_report` has an `slm` indicator — use it if a task says "SLM is not working, fix it".

---


# Configure a snapshot to be searchable
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/searchable-snapshots.html
> Searchable snapshots let you use snapshots to search infrequently accessed and read-only data in a very cost-effective fashion. The cold and frozen data tiers use searchable snapshots to reduce your storage and operating costs.
> 
> Searchable snapshots eliminate the need for replica shards, potentially halving the local storage needed to search your data. Searchable snapshots rely on the same snapshot mechanism you already use for backups and have minimal impact on your snapshot repository storage costs.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ilm-searchable-snapshot.html  <br>
Well worth watching: https://www.youtube.com/watch?v=nN6JNP9i3qQ

:question:  Create a searchable snapshot of the Kibana eCommerce data.

<details>
  <summary>View Solution (click to reveal)</summary>

## Set cache size

> Defaults to 90% of total disk space for dedicated frozen data tier nodes. Otherwise defaults to 0b.

`xpack.searchable.snapshot.shared_cache.size=100mb`

## create snapshot repository

```json
PUT /_snapshot/my_snapshots
{
    "type": "fs",
    "settings": {
        "location": "/tmp/snapshots",
        "compress": true
    }
}
```

## check repo

```json
GET _snapshot/my_snapshots
```

## create a snapshot

```json
PUT /_snapshot/my_snapshots/%3Cecomm-snapshot-%7Bnow%2Fd%7D%3E
{
  "indices": "kibana_sample_data_ecommerce",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

## list snapshots

```json
GET _snapshot/my_snapshots/ecomm*
```

## recover snapshot into local cache by mounting it

This is the Pièce de résistance - the snapshot is `mounted` in local shared cache.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/searchable-snapshots-api-mount-snapshot.html
```json
POST _snapshot/my_snapshots/ecomm-snapshot-2021.10.07/_mount?storage=shared_cache
{
  "index" : "kibana_sample_data_ecommerce",
  "renamed_index": "mounted-ecomm"
}
```


## test mounted snapshot

```json
GET _cat/indices/mounted-ecomm?v

GET _cat/count/mounted-ecomm
GET mounted-ecomm/_count
GET mounted-ecomm/_search
{
  "size": 5, 
  "query": {
    "match_all": {}
  }
}
```

## clean up
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/searchable-snapshots-api-clear-cache.html
clear the cache
```json
POST /mounted-ecomm/_searchable_snapshots/cache/clear
```

and delete mounted index
```json
DELETE mounted-ecomm
```

</details>
<hr>




# Configure a cluster for cross cluster search
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/modules-cross-cluster-search.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/remote-clusters.html

:bulb: Writing the actual cross-cluster *query* is a Searching Data objective — see [Searching_Data.md](Searching_Data.md). This objective is the **configuration**.

## Two security models in 8.15

| Model | Notes |
| --- | --- |
| **API key based** (added 8.14, the recommended one) | You create a cross-cluster API key on the *remote*, store it in the local cluster's keystore under `cluster.remote.<alias>.credentials`, and the remote grants fine-grained access. Requires the remote cluster to have `remote_cluster_server.enabled: true` (port **9443** by default). |
| **Certificate based** (the classic one) | Mutual TLS on the transport port (**9300**). The local user's *role names* are sent to the remote, which must have identically-named roles. Simpler, but a superuser on one side is a superuser on the other. |

## Two connection modes

| Mode | Setting | Use when |
| --- | --- | --- |
| `sniff` (default) | `cluster.remote.<alias>.seeds` | you can reach the remote nodes' publish addresses directly |
| `proxy` | `cluster.remote.<alias>.mode: proxy` + `.proxy_address` | the remote sits behind a load balancer / TCP proxy (this is what Elastic Cloud uses) |

## Update the cluster settings with the seed node of each remote cluster

```json
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_one": {
          "seeds": [
            "10.0.0.1:9300"
          ]
        },
        "cluster_two": {
          "seeds": [
            "2.0.0.1:9300"
          ]
        },
        "cluster_three": {
          "seeds": [
            "3.0.0.1:9300"
          ]
        }
      }
    }
  }
}
```
Here we have three clusters, one on the local network somewhere and two others out on the internet.

:bulb: This setup can be easily achieved within the Kibana GUI, and is part of the cross cluster replication below.

See [this section](#create-cluster-replication)

```json
GET _cluster/settings?pretty&flat_settings

// output

{
  "persistent" : {
    "cluster.remote.west-cluster.mode" : "sniff",
    "cluster.remote.west-cluster.node_connections" : "3",
    "cluster.remote.west-cluster.seeds" : [
      "esnode-west:9300"
    ],
    "cluster.remote.west-cluster.skip_unavailable" : "false"
  },
  "transient" : { }
}
```

```json
GET west-cluster:follower-kibana_sample_data_ecommerce/_count

// output

{
  "count" : 4675,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  }
}
```

## Perform a remote cluster search

```json
GET /cluster_one:twitter/_search
{
  "query": {
    "match": {
      "user": "kimchy"
    }
  }
}
```
Here we search one of the remote clusters from inside our local cluster.

Remote clusters are accessed as such <remote_name>:<index_name>

## Perform a multiple cross cluster search

```json
GET /twitter,cluster_one:twitter,cluster_two:twitter/_search
{
  "query": {
    "match": {
      "user": "kimchy"
    }
  }
}
```
Here we search the local cluster and two remote clusters.

## Verify the configuration

These three are the "prove it works" commands for this objective:

```json
// which remotes are registered, are they connected, how many nodes?
GET _remote/info

// which clusters and indices does a pattern actually resolve to, and are they up? (8.13+)
GET _resolve/cluster/*:kibana_sample_data_*

// read back the settings you applied
GET _cluster/settings?filter_path=persistent.cluster.remote
```

## Per-remote settings you may be asked for

```json
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.cluster_one.seeds": ["10.0.0.1:9300"],
    "cluster.remote.cluster_one.skip_unavailable": true,       // don't fail the whole search if it's down
    "cluster.remote.cluster_one.transport.compress": true,
    "cluster.remote.cluster_one.node_connections": 3           // sniff mode only
  }
}
```

Proxy mode instead of sniff:
```json
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.cluster_two.mode": "proxy",
    "cluster.remote.cluster_two.proxy_address": "proxy.example.com:9400",
    "cluster.remote.cluster_two.proxy_socket_connections": 18
  }
}
```

Remove a remote by setting it to `null`:
```json
PUT _cluster/settings
{ "persistent": { "cluster.remote.cluster_one.seeds": null } }
```

:warning: The node doing the cross-cluster work needs the **`remote_cluster_client`** role. On a default node with all roles this is already true, but on a cluster with dedicated node roles it is a very common cause of "remote cluster is not connected".

# Implement cross-cluster replication
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/xpack-ccr.html
> With cross-cluster replication, you can replicate indices across clusters to:
>
> - Continue handling search requests in the event of a datacenter outage
> - Prevent search volume from impacting indexing throughput
> - Reduce search latency by processing search requests in geo-proximity to the user
>
> Cross-cluster replication uses an active-passive model. You index to a leader index, and the data is replicated to one or more read-only follower indices. Before you can add a follower index to a cluster, you must configure the remote cluster that contains the leader index.

This is a heavily involved process - follow this link - with the guidelines below <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ccr-getting-started.html (reference to the following) <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ccr-getting-started-tutorial.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/remote-clusters-connect.html   <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ccr-apis.html <br>

## :warning: Do this with the API, not just the GUI

The walkthrough below is Kibana-driven, which is fine for understanding it — but **the exam is done in Dev Tools**, and this is the section people most often only ever practise in the UI. Learn these calls:

```json
// 0. PREREQUISITES (on the FOLLOWER cluster)
//    - a platinum/enterprise licence or a 30-day trial
//    - the leader registered as a remote cluster
PUT _cluster/settings
{ "persistent": { "cluster.remote.leader-cluster.seeds": ["esnode-east:9300"] } }

GET _remote/info

// 1. FOLLOW A SINGLE INDEX  (run on the FOLLOWER)
PUT /follower-my-index/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster": "leader-cluster",
  "leader_index": "my-index"
}

// 2. CHECK IT
GET /follower-my-index/_ccr/stats     // per-index: leader/follower global checkpoints, lag, read exceptions
GET /_ccr/stats                       // cluster-wide, plus auto-follow stats
GET /follower-my-index/_ccr/info      // status: active | paused

// 3. AUTO-FOLLOW PATTERN  (run on the FOLLOWER) - replicates any NEW index matching the pattern
PUT /_ccr/auto_follow/east-auto-follow
{
  "remote_cluster": "leader-cluster",
  "leader_index_patterns": ["test-ccr-index-*"],
  "leader_index_exclusion_patterns": ["test-ccr-index-ignore-*"],
  "follow_index_pattern": "follower-{{leader_index}}"
}

GET /_ccr/auto_follow/east-auto-follow
DELETE /_ccr/auto_follow/east-auto-follow

// 4. LIFECYCLE OF A FOLLOWER INDEX
POST /follower-my-index/_ccr/pause_follow
POST /follower-my-index/_ccr/resume_follow
{ "max_read_request_operation_count": 5120 }

// converting a follower into a normal writeable index is a THREE step dance:
POST /follower-my-index/_ccr/pause_follow
POST /follower-my-index/_close
POST /follower-my-index/_ccr/unfollow
POST /follower-my-index/_open
```

:warning: Things the exam will punish:
- A follower index is **read-only**. You cannot index into it — writes must go to the leader.
- The leader index must have **soft deletes enabled** (`index.soft_deletes.enabled`, default `true` in 7.x+ and not configurable in 8.x, so this is mostly a "know why it matters" point).
- CCR needs a **platinum/enterprise licence** — start a 30-day trial in a lab: `POST /_license/start_trial?acknowledge=true`
- Both clusters need the `remote_cluster_client` node role on the coordinating nodes.
- `follow_index_pattern` uses `{{leader_index}}` — double braces, mustache style.

:question: Replicate `kibana_sample_data_ecommerce` from the `east` cluster to the `west cluster`

<details>
  <summary>View Solution (click to reveal)</summary>

## Docker compose

Use the `2es-2kibana-xpack-cluster-713.yml` docker compose file.
This also requires `kibana-east.yml` and `kibana-west.yml`.

The docker-compose file has the correct node settings applied.

`node.roles=master,data,ingest,remote_cluster_client`

## Licencing

You will have to setup a 30day trial licence, to be able to do this part of the lab. This can easily be done once the nodes are up.

`Stack Management -> License management -> Start a 30-day trial -> Start trial {click} -> Start my trial {click}`

## Create cluster replication

Do this in Kibana as per the instructions above.

:warning: You will have to start one incognito/private browsing session or use different browsers to connect to both kibana instances at once.  This is due to the browser security cookies overwriting each other and subsequently logging you out everytime. :)

`Stack Management -> Remote Clusters -> Add a remote cluster`


| Cluster | Kibana URL | Remote Name | Seed Nodes | Direction |
| --- | --- | --- | --- | --- |
| East | http://\<your host ip>:5601 | west-cluster | esnode-west:9300 | Leader | 
| West | http://\<your host ip>:5602 | east-cluster | esnode-east:9300 | Follower |


## Add sample data

Add the sample data to the Leader (east)

Add the Kibana eCommerce sample data.

## Set up cross-cluster replication

Replicate the following index from East to West.

`kibana_sample_data_ecommerce`


> The index status changes to Paused. When the remote recovery process is complete, the index following begins and the status changes to Active.


## Create an auto-follow pattern and add documents to the index

On West, create an auto-follow pattern `test-ccr-index-*`

On East, create a timeseries index `test-ccr-index-000000`

Add docs to the new time series index, and confirm they are on West


| Name | Remote Cluster | Index Patterns | Prefix | Suffix | 
| --- | --- | --- | --- | --- |
| east-auto-follow | east-cluster | test-ccr-index-* | follower- | `none` |


### Create the index and add the docs to the leader index (east)

```json
PUT test-ccr-index-000000
{
  "settings": {
    "number_of_replicas": 0,
    "number_of_shards": 1
  },
  "mappings": {
    "properties": {
      "@timestamp" : {
        "type": "date"
      },
      "data": {
        "type": "text"
      }
    }
  }
}
```

Add the docs
```json
POST test-ccr-index-000000/_doc
{
  "@timestamp": "2021-10-07T11:50:00.000Z",
  "data" : "test data"
}

POST test-ccr-index-000000/_doc
{
  "@timestamp": "2021-10-07T11:55:00.000Z",
  "data" : "test data"
}
```

Test the index
```json
GET test-ccr-index-000000/_search
{
  "query": {
    "match_all": {}
  }
}
```


### Check data in follower index (west)

```json
GET _cat/indices/follow*

// output

health status index                                 uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   follower-kibana_sample_data_ecommerce RwN36YDWRNuLffDGftF0eA   1   0       4675            0      4.4mb          4.4mb
green  open   follower-test-ccr-index-000000        K8gUG9L2Rz2oqTX-OB_P3A   1   0          2            0      3.8kb          3.8kb

```

test the index

```json
GET follower-test-ccr-index-000000/_search
{
  "query": {
    "match_all": {}
  }
}
```


</details>
<hr>
---
---

# :books: BONUS — not part of the 8.15 exam objectives
# (Bonus) Define role-based access control using Elasticsearch Security

>  :warning: :warning: :warning: IMPORTANT NOTE: from here on it is assumed you have a working kibana node to work from the "development console" and that you have imported the sample data.

> For cluster setup, see [Part 0 of example.md](example.md). For the Kibana sample data sets used below, add them from **Kibana → Home → Try sample data**.

:question: To do this section you need to:

- create roles and users

:warning: Not all of the security features can be used due to Basic Licence limitations.

You will see an error like the below if you have a Basic licence.

The following role parameters are not available under Basic licence.

```yaml
  "field_security" : { ... }, # field level security
  "query": "..." # document level security
```

Error message:
```yaml
{
  "error": {
    "root_cause": [
      {
        "type": "security_exception",
        "reason": "current license is non-compliant for [field and document level security]",
        "license.expired.feature": "field and document level security"
      }
    ],
    "type": "security_exception",
    "reason": "current license is non-compliant for [field and document level security]",
    "license.expired.feature": "field and document level security"
  },
  "status": 403
}
```

## Create a role

:warning: This assumes you have ingested the Kibana sample flight data (**Kibana → Home → Try sample data → Sample flight data**).

### :question:  Create role and user for the following:

- a role called `flights_all` for read only access on the Flight sample data
- the role should have cluster monitor access
- a user called `flight_reader_all` that has the role applied
- the user password should be `flight123`

<details>
  <summary>View Solution (click to reveal)</summary>

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/built-in-roles.html
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/security-privileges.html

```json
PUT _security/role/flights_all
{
  "cluster": [ "monitor" ],
  "indices": [
    {
      "names": ["kibana_sample_data_flights"],
      "privileges": ["read","view_index_metadata", "monitor"]
    }
  ]
}

PUT _security/user/flight_reader_all
{
  "password": "flight123",
  "roles": [ "kibana_user", "flights_all" ],
  "full_name": "flights all",
  "email": "fa@abc.com"
}
```

Test the user access

- Logout as elastic 
- Login to Kibana as `flight_reader_all` password `flight123`
- Go to dev console and see what indices you have access to.

Check that we can access the index stats (monitor)
and only the index we have allowed access to.

```json
GET _cat/indices

green open kibana_sample_data_flights R1AptZYHTrivEUEmtftubg 1 0 13059 0 6.6mb 6.6mb
```

Check that we can query the index.  Get the document count

```json
GET kibana_sample_data_flights/_count

{
  "count" : 13059,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  }
}
```


</details>




### :question:  Create a role with field and document level security, plus a user for the following:

:warning: can only be applied if you have purchased an elastic licence. This will not work on the Basic/Free licence.

- a role called `flights_australia` for read only access on the Flight data that only allows access to data that has a Destination Country of Australia.
- the following fields are allowed to be displayed: Flight Number, Country of Origin and City of Origin
- a user called `flight_reader_au` should have the role applied to it

<details>
  <summary>View Solution (click to reveal)</summary>

```json

# test your query first
POST kibana_sample_data_flights/_search
{
  "query": {
        "match": {
          "DestCountry": "AU"
        }
      }
}

PUT _security/role/flights_australia
{
  "indices": [
    {
      "names": [
        "kibana_sample_data_flights"
      ],
      "privileges": [
        "read"
      ],
      "field_security": {
        "grant": ["FlightNum", "OriginCountry", "OriginCityName"]
      }, 
      "query": {
        "match": {
          "DestCountry": "AU"
        }
      }
    }
  ]
}
```

Create a user for that role

```yaml
PUT _security/user/flight_reader_au
{
  "password": "flight123",
  "roles": "flights_australia",
  "full_name": "flights australia",
  "email": "fau@abc.com"
}
```

Test

- Logout as elastic 
- Login to Kibana as `flight_reader_au`
- Go to dev console and see what indices you have access to.

#TODO:  write a query to count the number of documents accessible

</details>
