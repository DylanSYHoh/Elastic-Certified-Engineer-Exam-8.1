# A Unified Example — Elastic Certified Engineer 8.15

## Goal

One dataset, one walkthrough, covering **every objective on the 8.15 exam** in the order the official objective list gives them. The data is 2024 solar eclipse totality information for US state parks.

The point of a unified example is that the indices build on each other, exactly like a real exam paper: the index you create in Part 1 is the index you search in Part 2, reindex in Part 4, and snapshot in Part 5.

## How to use this file

1. Work through **Part 0** to get a cluster and the data loaded. Everything after that depends on it.
2. Then work Parts 1–5 in order. Each objective has a :question: task; try it before reading the code.
3. Reference material for each objective lives in the per-section files: [Data_Management.md](Data_Management.md), [Searching_Data.md](Searching_Data.md), [Developing_Search_Applications.md](Developing_Search_Applications.md), [Data_Processing.md](Data_Processing.md), [Cluster_Management.md](Cluster_Management.md).

## :rotating_light: Version note

The exam is on **Elastic 8.15** until **31 August 2026**, then moves to 9.3. This file targets 8.15.

## Objective map

| # | Objective | Section |
| --- | --- | --- |
| **1. Data Management** | | |
| 1.1 | Define an index that satisfies a given set of requirements | [→](#11-define-an-index-that-satisfies-a-given-set-of-requirements) |
| 1.2 | Define and use a dynamic template | [→](#12-define-and-use-a-dynamic-template-that-satisfies-a-given-set-of-requirements) |
| 1.3 | Define an ILM policy for a time-series index | [→](#13-define-an-index-lifecycle-management-policy-for-a-time-series-index) |
| 1.4 | Define an index template that creates a new data stream | [→](#14-define-an-index-template-that-creates-a-new-data-stream) |
| **2. Searching Data** | | |
| 2.1 | Terms and/or phrases in one or more fields | [→](#21-write-and-execute-a-search-query-for-terms-andor-phrases-in-one-or-more-fields-of-an-index) |
| 2.2 | Boolean combination of queries and filters | [→](#22-write-and-execute-a-search-query-that-is-a-boolean-combination-of-multiple-queries-and-filters) |
| 2.3 | Asynchronous search | [→](#23-write-an-asynchronous-search) |
| 2.4 | Metric and bucket aggregations | [→](#24-write-and-execute-metric-and-bucket-aggregations) |
| 2.5 | Aggregations with sub-aggregations | [→](#25-write-and-execute-aggregations-that-contain-sub-aggregations) |
| 2.6 | Query across multiple clusters | [→](#26-write-and-execute-a-query-that-searches-across-multiple-clusters) |
| 2.7 | Search using a runtime field | [→](#27-write-and-execute-a-search-that-utilizes-a-runtime-field) |
| **3. Developing Search Applications** | | |
| 3.1 | Sort results | [→](#31-sort-the-results-of-a-query-by-a-given-set-of-requirements) |
| 3.2 | Pagination | [→](#32-implement-pagination-of-the-results-of-a-search-query) |
| 3.3 | Index aliases | [→](#33-define-and-use-index-aliases) |
| **4. Data Processing** | | |
| 4.1 | Define a mapping | [→](#41-define-a-mapping-that-satisfies-a-given-set-of-requirements) |
| 4.2 | Multi-fields with different types/analyzers | [→](#42-define-and-use-multi-fields-with-different-data-types-andor-analyzers) |
| 4.3 | Reindex API and Update By Query API | [→](#43-use-the-reindex-api-and-update-by-query-api-to-reindex-andor-update-documents) |
| 4.4 | Ingest pipeline | [→](#44-define-and-use-an-ingest-pipeline-that-satisfies-a-given-set-of-requirements) |
| 4.5 | Runtime fields using Painless | [→](#45-define-runtime-fields-to-retrieve-custom-values-using-painless-scripting) |
| **5. Cluster Management** | | |
| 5.1 | Diagnose shard issues and repair cluster health | [→](#51-diagnose-shard-issues-and-repair-a-clusters-health) |
| 5.2 | Backup and restore | [→](#52-backup-and-restore-a-cluster-andor-specific-indices) |
| 5.3 | Configure a snapshot to be searchable | [→](#53-configure-a-snapshot-to-be-searchable) |
| 5.4 | Configure a cluster for cross-cluster search | [→](#54-configure-a-cluster-for-cross-cluster-search) |
| 5.5 | Implement cross-cluster replication | [→](#55-implement-cross-cluster-replication) |
| 5.6 | Automate snapshots with SLM | [→](#56-automate-snapshots-with-snapshot-lifecycle-management) |
| **Bonus** | Dropped from the 8.15 list | [→](#bonus--no-longer-on-the-objective-list) |

---
---

# Part 0 — Environment and data

## 0.1 Install Elasticsearch 8.15

Two options:

**Docker**
```bash
docker pull docker.elastic.co/elasticsearch/elasticsearch:8.15.0
```
See https://www.docker.elastic.co/r/elasticsearch

**Manual install** — download from https://www.elastic.co/downloads/past-releases#elasticsearch, make sure the Java and OS prerequisites are met, then install and start.

You want Kibana too, purely for the **Dev Tools console**. Everything in this file is a Dev Tools request.

## 0.2 Multiple clusters (optional — only needed for 5.4 and 5.5)

Cross-cluster search and cross-cluster replication are the only two objectives that need a second cluster. Everything else runs on a single node. If you want to skip those, skip this step.

Steps for a multi-node/multi-cluster setup:
1. Install Elasticsearch on both hosts
2. Set a distinct `cluster.name` on each
3. Set descriptive `node.name` values
4. Disable memory swapping
5. Define node roles — **include `remote_cluster_client`** on nodes that will coordinate CCS/CCR
6. Bind to a non-loopback address (`network.host`)
7. Configure discovery and cluster formation
8. Set JVM heap size
9. Validate max processes and open file descriptors
10. Ensure enough virtual memory (`vm.max_map_count`)
11. Open the Elasticsearch ports on the firewall (9200 HTTP, 9300 transport, 9443 for API-key remote cluster access)
12. Start Elasticsearch

A good step-by-step: https://kifarunix.com/setup-multi-node-elasticsearch-cluster

## 0.3 Load the data

:bulb: **Do this before Parts 1–4.** Everything downstream queries `totality-raw`.

The dataset is `example-date/full-eclipse-data.json` — a bulk NDJSON file with **190 state parks** across four states, already targeting the index `totality-raw`.

```bash
curl -k -u "elastic:Password01" -s -H "Content-Type: application/x-ndjson" \
  -XPOST "localhost:9200/_bulk?refresh" --data-binary "@example-date/full-eclipse-data.json"
```

:warning: The file has `{"index":{"_index":"totality-raw","_id":N}}` action lines, so post it to `/_bulk`, **not** to `/<index>/_bulk`. Bulk files must also end with a newline.

Verify:
```json
GET totality-raw/_count

// expect
{ "count": 190 }
```

### Know your data

| Field | Type in the source JSON | Notes |
| --- | --- | --- |
| `name` | string | e.g. `"Ahern State Park"` |
| `street_address`, `city`, `state`, `zip_code` | string | `zip_code` has leading zeros — **must** be a keyword, not a number |
| `timezone` | string | `EDT` (150 docs) or `CDT` (39 docs) |
| `coverage` | string | e.g. `"100%"`, `"97.47%"` |
| `eclipse_date` | date | all `2024-04-08` |
| `totality_minutes`, `totality_seconds` | integer | `0` for parks outside the path |
| `partial_start_time`, `max_time`, `partial_end_time` | date | ISO 8601 UTC |

Baseline numbers you can check your answers against:

```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "by_state": { "terms": { "field": "state.keyword" } }
  }
}
```

| Fact | Value |
| --- | --- |
| Total documents | 190 |
| New Hampshire / Vermont / Oklahoma / Maine | 67 / 51 / 39 / 33 |
| Documents with `coverage` = `"100%"` | 51 |
| Documents with `totality_minutes` > 0 | 44 |
| Max `totality_minutes` | 4 |

:bulb: **Deliberate data quirk:** exactly **one** document (`_id: 0`, Ahern State Park) uses the field name `coverage_percent` instead of `coverage`. That single inconsistent field is a realistic mess, and it is what makes the dynamic-template and ingest-pipeline exercises below worth doing. A count of 189 rather than 190 on a `coverage`-based query is not a bug in your query.

Two other files in `example-date/`:
- `put-runtime-fields.json` — the same data with no `_index` in the action lines, so you can bulk it into any index you name in the URL.
- `unique-clusters.json` — the same records as a JSON array, one line per state. Used to load **different states into different clusters** so cross-cluster search actually returns different data from each. Do not bulk-load this file as-is.

---
---

# Part 1 — Data Management

## 1.1 Define an index that satisfies a given set of requirements

:question: Create an index `totality-2024-raw` with one primary shard, no replicas, a 30 second refresh interval, and a mapping where `zip_code` is a keyword, `eclipse_date` is a date, and `totality_minutes`/`totality_seconds` are integers.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT totality-2024-raw
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "refresh_interval": "30s"
  },
  "mappings": {
    "properties": {
      "name":             { "type": "keyword" },
      "street_address":   { "type": "keyword" },
      "city":             { "type": "keyword" },
      "state":            { "type": "keyword" },
      "zip_code":         { "type": "keyword" },
      "timezone":         { "type": "keyword" },
      "coverage":         { "type": "keyword" },
      "eclipse_date":     { "type": "date" },
      "totality_minutes": { "type": "integer" },
      "totality_seconds": { "type": "integer" },
      "partial_start_time": { "type": "date" },
      "max_time":           { "type": "date" },
      "partial_end_time":   { "type": "date" }
    }
  }
}

// expected response
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "totality-2024-raw"
}
```

:warning: `zip_code` **must** be `keyword`. As a number, `"03246"` becomes `3246` and you lose the leading zero — and you can no longer do prefix/wildcard queries on it. This is the classic "map a numeric identifier as a keyword" case.

Verify:
```json
GET totality-2024-raw/_settings
GET totality-2024-raw/_mapping
```
</details>
<hr>

### Background: index templates for a pattern

> :books: No longer a listed 8.15 objective on its own — but it is a hard prerequisite for 1.4 (data streams), so do it.

:question: Create a template so that **any** index matching `totality-2024-*` gets the mapping above automatically.

<details>
  <summary>View Solution (click to reveal)</summary>

:warning: Use `PUT _index_template/`, **not** the deprecated `PUT _template/`. Note that settings and mappings move inside a `template` block, and legacy templates are removed in 9.x.

```json
PUT _index_template/totality-2024-tmpl
{
  "index_patterns": ["totality-2024-*"],
  "priority": 200,
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0
    },
    "mappings": {
      "properties": {
        "name":             { "type": "keyword" },
        "street_address":   { "type": "keyword" },
        "city":             { "type": "keyword" },
        "state":            { "type": "keyword" },
        "zip_code":         { "type": "keyword" },
        "timezone":         { "type": "keyword" },
        "coverage":         { "type": "keyword" },
        "eclipse_date":     { "type": "date" },
        "totality_minutes": { "type": "integer" },
        "totality_seconds": { "type": "integer" },
        "partial_start_time": { "type": "date" },
        "max_time":           { "type": "date" },
        "partial_end_time":   { "type": "date" }
      }
    }
  }
}
```

:warning: Do **not** put `"_source": { "enabled": false }` in this template. It silently breaks the `_reindex` and `_update_by_query` exercises in Part 4.

Prove it works **without creating anything** — this is the single most useful template-debugging API in the exam:
```json
POST _index_template/_simulate_index/totality-2024-vermont
```
The response shows the exact settings and mappings a new index by that name would receive.

Read it back:
```json
GET _index_template/totality-2024-tmpl
GET _cat/templates?v
```

**Bonus:** add a `state_abbrev` keyword field to the template, then re-simulate to confirm it appears.
</details>
<hr>

## 1.2 Define and use a dynamic template that satisfies a given set of requirements

Dynamic templates control how fields you did **not** explicitly map get mapped when they first appear.

:question: Create an index `totality-dyn` where:
- any field whose name starts with `totality_` is mapped as `long`, even if the source sends it as a string
- any field whose name ends in `_time` or is `eclipse_date` is mapped as `date`
- every other string is mapped as `keyword` (no `text`, to save space)

<details>
  <summary>View Solution (click to reveal)</summary>

:warning: **Order matters.** `dynamic_templates` is an array and the first matching rule wins, so put the specific rules before the catch-all.

```json
PUT totality-dyn
{
  "mappings": {
    "dynamic_templates": [
      {
        "totality_counts_as_long": {
          "match": "totality_*",
          "mapping": { "type": "long" }
        }
      },
      {
        "times_as_date": {
          "match": "*_time",
          "mapping": { "type": "date" }
        }
      },
      {
        "eclipse_date_as_date": {
          "match": "eclipse_date",
          "mapping": { "type": "date" }
        }
      },
      {
        "strings_as_keywords": {
          "match_mapping_type": "string",
          "mapping": { "type": "keyword" }
        }
      }
    ]
  }
}
```

Test it by indexing a document with **string** totality values and a field the template never mentioned:

```json
PUT totality-dyn/_doc/1
{
  "name": "Boaty McBoatface State Park",
  "city": "Laconia",
  "totality_minutes": "5",
  "totality_seconds": "0",
  "eclipse_date": "2024-04-08",
  "max_time": "2024-04-08T15:29:32.000Z",
  "surprise_field": "not in any rule"
}

GET totality-dyn/_mapping
```

Expected: `totality_minutes`/`totality_seconds` are `long` (coerced from the strings), the time fields are `date`, and `surprise_field` is `keyword` via the catch-all.
</details>
<hr>

:question: **Bonus, using the real data quirk:** the source has one document with `coverage_percent` and 189 with `coverage`. Write a dynamic template that maps *any* field starting with `coverage` as a keyword, so both spellings behave the same.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT totality-dyn2
{
  "mappings": {
    "dynamic_templates": [
      {
        "any_coverage_as_keyword": {
          "match": "coverage*",
          "mapping": { "type": "keyword" }
        }
      }
    ]
  }
}
```
This does not *merge* the two fields — for that you need an ingest pipeline (see 4.4) — but it does guarantee both are exact-match keywords rather than one being analysed text.
</details>
<hr>

:question: Which `dynamic` setting would make Elasticsearch **reject** a document containing `coverage_percent` instead of silently mapping it?

<details>
  <summary>View Solution (click to reveal)</summary>

`"dynamic": "strict"`. Know all four values:

| `dynamic` | Behaviour |
| --- | --- |
| `true` (default) | new fields are added to the mapping and indexed |
| `runtime` | new fields become **runtime** fields — queryable, not indexed |
| `false` | new fields are ignored and not indexed, but still stored in `_source` |
| `strict` | indexing a document with an unknown field throws `strict_dynamic_mapping_exception` |

```json
PUT totality-strict
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "name":     { "type": "keyword" },
      "coverage": { "type": "keyword" }
    }
  }
}

PUT totality-strict/_doc/1
{ "name": "Ahern State Park", "coverage_percent": "97.47%" }
// -> strict_dynamic_mapping_exception
```
</details>
<hr>

## 1.3 Define an Index Lifecycle Management policy for a time-series index

:question: Create an ILM policy `totality_policy` where data is hot for 3 minutes then rolls over to warm, warm for 5 minutes then to cold, and is deleted 10 minutes after rollover.

<details>
  <summary>View Solution (click to reveal)</summary>

:warning: `min_age` in each phase is measured **from the rollover time** of that index, not from the end of the previous phase — so the values are cumulative from rollover.

```json
PUT _ilm/policy/totality_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "3m"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "3m",
        "actions": { "set_priority": { "priority": 50 } }
      },
      "cold": {
        "min_age": "5m",
        "actions": { "set_priority": { "priority": 0 } }
      },
      "delete": {
        "min_age": "10m",
        "actions": { "delete": {} }
      }
    }
  }
}
```

:warning: Two rules that get marked:
- `rollover` is legal in the **hot phase only**.
- A rollover action needs at least one `max_*` condition. In 8.15 the options are `max_age`, `max_docs`, `max_size`, `max_primary_shard_size`, `max_primary_shard_docs`, each with a `min_*` counterpart that must be satisfied before any `max_*` can fire.

Attach it to a rollover alias:
```json
PUT _index_template/totality-ts-tmpl
{
  "index_patterns": ["totality-ts-*"],
  "priority": 200,
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "index.lifecycle.name": "totality_policy",
      "index.lifecycle.rollover_alias": "totality-ts"
    }
  }
}
```

Bootstrap the first index — **this is required**, rollover on an alias will not work without a designated write index:
```json
PUT totality-ts-000001
{
  "aliases": { "totality-ts": { "is_write_index": true } }
}

POST totality-ts/_doc
{ "@timestamp": "2026-08-14T10:30:00.000Z", "message": "first doc" }
```

Watch it, and force a rollover instead of waiting:
```json
GET totality-ts-*/_ilm/explain?filter_path=*.*.age,*.*.phase
POST totality-ts/_rollover
GET _alias/totality-ts
```

:bulb: **ILM polls every 10 minutes by default**, so minute-scale policies never look instant. In a lab only:
```json
PUT _cluster/settings
{ "persistent": { "indices.lifecycle.poll_interval": "10s" } }
```

Debugging, in the order you should reach for them:
```json
GET _ilm/status                          // is ILM even RUNNING?
GET totality-ts-*/_ilm/explain           // where is each index, and what is step_info saying?
POST totality-ts-000001/_ilm/retry       // after fixing the cause of a stuck index
```
</details>
<hr>

## 1.4 Define an index template that creates a new data stream

:question: Create a data stream `totality-events` with an ILM policy, a component template for mappings, a component template for settings, and an index template that ties them together.

<details>
  <summary>View Solution (click to reveal)</summary>

### Step 1 — ILM policy

Nothing data-stream-specific about it:
```json
PUT _ilm/policy/totality_ds_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": { "max_primary_shard_size": "50gb", "max_age": "30d" }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

### Step 2 — component templates

```json
PUT _component_template/totality-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date", "format": "date_optional_time||epoch_millis" },
        "park_name":  { "type": "keyword" },
        "state":      { "type": "keyword" },
        "message":    { "type": "wildcard" }
      }
    }
  },
  "_meta": { "description": "Mappings for eclipse observation events" }
}

PUT _component_template/totality-settings
{
  "template": {
    "settings": {
      "index.lifecycle.name": "totality_ds_policy",
      "number_of_shards": 1,
      "number_of_replicas": 0
    }
  },
  "_meta": { "description": "ILM settings for eclipse events" }
}
```

:warning: The policy name here must match Step 1 **exactly**. A typo (`totality-ds-policy` vs `totality_ds_policy`) silently gives you a data stream with no lifecycle, and `_ilm/explain` returns nothing useful.

### Step 3 — the index template

`"data_stream": { }` is the thing that makes this a data stream. An empty object is correct.

```json
PUT _index_template/totality-events-tmpl
{
  "index_patterns": ["totality-events*"],
  "data_stream": { },
  "composed_of": ["totality-mappings", "totality-settings"],
  "priority": 500,
  "_meta": { "description": "Template for eclipse observation events" }
}
```

:warning: Do **not** set `index.lifecycle.rollover_alias` on a data stream template. Data streams roll over on their own name; the alias setting is for the classic index+alias pattern in 1.3, and setting it here is a common wrong answer.

### Step 4 — create the data stream

Implicitly, by writing a document:
```json
POST totality-events/_doc
{
  "@timestamp": "2024-04-08T15:29:32.000Z",
  "park_name": "Ahern State Park",
  "state": "New Hampshire",
  "message": "maximum eclipse observed, 97.47% coverage"
}
```
Or explicitly:
```json
PUT _data_stream/totality-events
```

:warning: Writes to a data stream must be `create` operations. `POST totality-events/_doc` is fine; `PUT totality-events/_doc/1` (indexing with an explicit ID) is **rejected**.

### Step 5 — inspect

```json
GET _data_stream/totality-events
GET _cat/indices/.ds-totality-events-*?v&expand_wildcards=all
POST totality-events/_rollover
```
Backing indices are named `.ds-totality-events-<yyyy.MM.dd>-<NNNNNN>` and are hidden.

### Also know: converting an existing alias to a data stream

```json
POST _data_stream/_migrate/totality-ts
```
The alias's indices become hidden backing indices and its write index becomes the stream's write index. A matching data-stream-enabled index template must already exist.

### Cleanup
```json
DELETE _data_stream/totality-events   // deletes the stream AND its backing indices
```
</details>
<hr>

---
---

# Part 2 — Searching Data

## 2.1 Write and execute a search query for terms and/or phrases in one or more fields of an index

:question: How many state parks are in New Hampshire?

<details>
  <summary>View Solution (click to reveal)</summary>

`state` is a `text` field with a `.keyword` sub-field under dynamic mapping, so there are two correct answers depending on which you use:

```json
// full-text match - "New Hampshire" is analysed into [new, hampshire], OR by default
GET totality-raw/_search?filter_path=hits.total.value
{
  "query": { "match": { "state": "New Hampshire" } }
}
```

:warning: This matches on *either* token, so it is not an exact-state query. The precise version uses the keyword sub-field:

```json
GET totality-raw/_search?filter_path=hits.total.value
{
  "query": { "term": { "state.keyword": "New Hampshire" } }
}

// Answer: 67
```

To force both words with the analysed field:
```json
GET totality-raw/_search?filter_path=hits.total.value
{
  "query": { "match": { "state": { "query": "New Hampshire", "operator": "and" } } }
}
```
</details>
<hr>

:question: How many parks are in Vermont, Maine, or Oklahoma?

<details>
  <summary>View Solution (click to reveal)</summary>

`terms` is `term` with multiple values — an implicit OR:

```json
GET totality-raw/_search?filter_path=hits.total.value
{
  "query": {
    "terms": {
      "state.keyword": ["Vermont", "Maine", "Oklahoma"]
    }
  }
}

// Answer: 51 + 33 + 39 = 123
```
</details>
<hr>

:question: How many parks have between 3 and 5 minutes of totality?

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search?filter_path=hits.total.value
{
  "query": {
    "range": {
      "totality_minutes": { "gte": 3, "lte": 5 }
    }
  }
}

// Answer: 25 (24 parks at 3 minutes, 1 at 4)
```
</details>
<hr>

:question: Find parks whose name contains the exact phrase "State Park", and separately, parks whose zip code starts with `05`.

<details>
  <summary>View Solution (click to reveal)</summary>

Phrase — word order matters, unlike `match`:
```json
GET totality-raw/_search?filter_path=hits.total.value
{
  "query": {
    "match_phrase": { "name": "State Park" }
  }
}
```

Prefix on a keyword field:
```json
GET totality-raw/_search?filter_path=hits.total.value
{
  "query": {
    "prefix": { "zip_code.keyword": "05" }
  }
}
```
Or with a wildcard:
```json
GET totality-raw/_search?filter_path=hits.total.value
{
  "query": {
    "wildcard": { "zip_code.keyword": "05*" }
  }
}
```
:bulb: This is exactly why `zip_code` must be a keyword and not a number.
</details>
<hr>

:question: Search **two fields at once** — find "Laconia" in either the city or the park name.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search
{
  "query": {
    "multi_match": {
      "query": "Laconia",
      "fields": ["city", "name"],
      "type": "best_fields"
    }
  }
}
```
Useful `type` values: `best_fields` (default), `most_fields`, `cross_fields`, `phrase`, `phrase_prefix`.
</details>
<hr>

:question: Find the one document that uses `coverage_percent` instead of `coverage`.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search
{
  "query": {
    "bool": {
      "must":     [ { "exists": { "field": "coverage_percent" } } ],
      "must_not": [ { "exists": { "field": "coverage" } } ]
    }
  }
}

// Answer: 1 document - Ahern State Park
```
`exists` is the tool for "which documents have / are missing this field" questions.
</details>
<hr>

## 2.2 Write and execute a search query that is a Boolean combination of multiple queries and filters

| Occur | Description |
| --- | --- |
| `must` | must match, **and contributes to the score** |
| `filter` | must match, score ignored, **cacheable** |
| `should` | should match — behaves like a logical OR |
| `must_not` | must not match, score ignored, cacheable |

:bulb: If you do not need relevance ranking, use `filter` rather than `must`. It is faster and cached. Exam questions phrased as "filter the results to…" want `filter`.

:question: Find all state parks in New Hampshire with the zip code `03579`.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search
{
  "size": 200,
  "query": {
    "bool": {
      "must": [
        { "term": { "state.keyword": "New Hampshire" } },
        { "term": { "zip_code.keyword": "03579" } }
      ]
    }
  }
}
```
</details>
<hr>

:question: Same query, but exclude parks with 0 seconds of totality.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search
{
  "size": 200,
  "query": {
    "bool": {
      "must": [
        { "term": { "state.keyword": "New Hampshire" } },
        { "term": { "zip_code.keyword": "03579" } }
      ],
      "must_not": [
        { "term": { "totality_seconds": 0 } }
      ]
    }
  }
}
```
</details>
<hr>

:question: Parks in **either** Vermont **or** Maine, that have 100% coverage, and more than 2 minutes of totality — with the coverage and totality conditions in filter context so they do not affect scoring.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search?filter_path=hits.total.value
{
  "track_total_hits": true,
  "query": {
    "bool": {
      "should": [
        { "term": { "state.keyword": "Vermont" } },
        { "term": { "state.keyword": "Maine" } }
      ],
      "minimum_should_match": 1,
      "filter": [
        { "term":  { "coverage.keyword": "100%" } },
        { "range": { "totality_minutes": { "gt": 2 } } }
      ]
    }
  }
}
```

:warning: **`minimum_should_match` is the trap.** When a `bool` query has a `must` or `filter` clause, `should` clauses become purely optional score boosters and match nothing on their own. Adding `"minimum_should_match": 1` forces at least one to match, which is what "either Vermont or Maine" actually means.
</details>
<hr>

## 2.3 Write an asynchronous search

The async search API submits a search, returns an `id`, and lets you poll for progress and partial results.

:warning: This dataset is far too small to produce a genuinely slow search, so a normal submit returns the full result inline and there is nothing to fetch by id. Force the async behaviour with the parameters below.

| Parameter | Default | Purpose |
| --- | --- | --- |
| `wait_for_completion_timeout` | `1s` | how long to block before returning an `id` — set it tiny to force an async response |
| `keep_on_completion` | `false` | store the results even if the search finished quickly — **you need this to fetch by id at all** |
| `keep_alive` | `5d` | how long results are retained |

:question: Submit an async search that buckets parks by state, in a way that actually returns an id on this small dataset.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
POST /totality-raw/_async_search?size=0&wait_for_completion_timeout=10ms&keep_on_completion=true&keep_alive=1d
{
  "aggs": {
    "by_state": {
      "terms": { "field": "state.keyword" }
    }
  }
}

// output
{
  "id": "FmRldE8zREVEUzA2ZVpUeGs2ejJFUFEaMkZ5QTVrSTZSaVN3WlNFVmtlWHJsdzoxMDc=",
  "is_partial": true,
  "is_running": true,
  "start_time_in_millis": 1583945890986,
  "expiration_time_in_millis": 1584377890986,
  "response": { ... }
}
```

`is_running` tells you whether it has finished; `is_partial` tells you whether the results you are looking at are complete.

Retrieve, check status, and clean up:
```json
GET    /_async_search/<id>
GET    /_async_search/status/<id>
DELETE /_async_search/<id>
```

:bulb: The status endpoint does not return results, only progress — it needs the `monitoring_user` role rather than access to the results themselves. `DELETE` cancels a running search or discards stored results.
</details>
<hr>

:question: A more realistic async candidate — a date histogram over the eclipse timeline, across all shards.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
POST /totality-raw/_async_search?size=0&wait_for_completion_timeout=10ms&keep_on_completion=true
{
  "aggs": {
    "eclipse_timeline": {
      "date_histogram": {
        "field": "max_time",
        "fixed_interval": "5m",
        "min_doc_count": 1
      }
    }
  }
}
```

:warning: `max_time` is a real timestamp with sub-hour spread, so `fixed_interval` gives a useful histogram. `eclipse_date` is the same value (`2024-04-08`) for all 190 documents, so a histogram on it produces exactly one bucket — technically valid, analytically useless.

:bulb: Use `calendar_interval` for units that vary in length (`1d`, `1M`, `1y`) and `fixed_interval` for exact ones (`5m`, `30s`, `12h`). Mixing them up is a common error.
</details>
<hr>

## 2.4 Write and execute metric and bucket aggregations

### Metric aggregations

Compute a value over the documents in scope: `avg`, `max`, `min`, `sum`, `stats`, `extended_stats`, `cardinality`, `percentiles`, `value_count`, `top_hits`.

:question: What is the longest totality in minutes across all parks?

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET /totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "most_minutes_totality": { "max": { "field": "totality_minutes" } }
  }
}

// Answer: 4
```
:bulb: `"size": 0` suppresses the hits so you only get the aggregation, and `filter_path=aggregations` trims the rest of the envelope. Use both.
</details>
<hr>

:question: Get count, min, max, average and sum of `totality_seconds` for parks with 100% coverage, in a single aggregation.

<details>
  <summary>View Solution (click to reveal)</summary>

`stats` gives you all five at once; `extended_stats` adds variance and standard deviation.

```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "query": {
    "term": { "coverage.keyword": "100%" }
  },
  "aggs": {
    "totality_stats": {
      "extended_stats": { "field": "totality_seconds" }
    }
  }
}
```
</details>
<hr>

:question: How many **distinct** states, cities and zip codes are in the dataset?

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "distinct_states":    { "cardinality": { "field": "state.keyword" } },
    "distinct_cities":    { "cardinality": { "field": "city.keyword" } },
    "distinct_zipcodes":  { "cardinality": { "field": "zip_code.keyword" } }
  }
}

// distinct_states: 4
```
:warning: `cardinality` is **approximate** — it uses HyperLogLog++. Exact for small sets like this one, but do not claim it is exact in general.
</details>
<hr>

### Bucket aggregations

Group documents into buckets: `terms`, `range`, `date_range`, `histogram`, `date_histogram`, `filter`, `filters`, `composite`, `missing`, `nested`, `significant_terms`.

:question: Show the number of parks per state.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "parks_per_state": {
      "terms": { "field": "state.keyword", "size": 10 }
    }
  }
}

// output
{
  "aggregations": {
    "parks_per_state": {
      "buckets": [
        { "key": "New Hampshire", "doc_count": 67 },
        { "key": "Vermont",       "doc_count": 51 },
        { "key": "Oklahoma",      "doc_count": 39 },
        { "key": "Maine",         "doc_count": 33 }
      ]
    }
  }
}
```
:warning: `terms` defaults to `"size": 10` buckets. If a question says "show all X", set `size` high enough — and check `sum_other_doc_count` in the response, which is non-zero when buckets were dropped.
</details>
<hr>

:question: Bucket the parks by minutes of totality, and separately into named ranges: none (0), short (1–2), long (3+).

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "by_minutes": {
      "terms": { "field": "totality_minutes" }
    },
    "by_duration_band": {
      "range": {
        "field": "totality_minutes",
        "keyed": true,
        "ranges": [
          { "key": "none",  "to": 1 },
          { "key": "short", "from": 1, "to": 3 },
          { "key": "long",  "from": 3 }
        ]
      }
    }
  }
}

// by_minutes: 0 -> 146, 3 -> 24, 2 -> 12, 1 -> 7, 4 -> 1
```
:warning: `range` bounds are `from` **inclusive**, `to` **exclusive**. The "short" bucket above is 1 and 2 minutes, not 1 to 3.
</details>
<hr>

:question: Count parks per timezone, and count how many documents are **missing** the `coverage` field.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "by_timezone": {
      "terms": { "field": "timezone.keyword" }
    },
    "no_coverage_field": {
      "missing": { "field": "coverage.keyword" }
    }
  }
}

// by_timezone: EDT -> 150, CDT -> 39  (the missing one is the coverage_percent doc)
// no_coverage_field: doc_count -> 1
```
The `missing` aggregation is the aggregation-side equivalent of the `exists` query, and it is how you find data-quality problems like this one.
</details>
<hr>

## 2.5 Write and execute aggregations that contain sub-aggregations

Sub-aggregations nest inside a bucket aggregation and run over the documents in each bucket. Build them **one layer at a time** and verify each layer before nesting the next — this is exactly how the exam question decomposes.

:question: For each state, show the average totality in seconds, the longest totality in minutes, and the number of parks with 100% coverage.

<details>
  <summary>View Solution (click to reveal)</summary>

**Layer 1** — the outer buckets. Confirm the state list and counts first:
```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "by_state": { "terms": { "field": "state.keyword", "size": 10 } }
  }
}
```

**Layer 2** — nest metrics inside. Note `aggs` is a **sibling** of `terms`, not inside it:
```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "by_state": {
      "terms": { "field": "state.keyword", "size": 10 },
      "aggs": {
        "avg_totality_seconds": {
          "avg": { "field": "totality_seconds" }
        },
        "max_totality_minutes": {
          "max": { "field": "totality_minutes" }
        },
        "full_coverage_parks": {
          "filter": { "term": { "coverage.keyword": "100%" } }
        }
      }
    }
  }
}

// full_coverage_parks doc_count: Vermont 29, Oklahoma 7, and the rest across NH/ME
```
:bulb: The `filter` **aggregation** is how you count a subset inside a bucket without filtering the whole query.
</details>
<hr>

:question: Which single park has the longest totality in each state?

<details>
  <summary>View Solution (click to reveal)</summary>

`top_hits` returns actual documents inside a bucket — the answer to every "which one is the X-est per group" question.

```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "by_state": {
      "terms": { "field": "state.keyword", "size": 10 },
      "aggs": {
        "longest_totality": {
          "top_hits": {
            "size": 1,
            "sort": [
              { "totality_minutes": { "order": "desc" } },
              { "totality_seconds": { "order": "desc" } }
            ],
            "_source": ["name", "city", "totality_minutes", "totality_seconds"]
          }
        }
      }
    }
  }
}
```
</details>
<hr>

:question: Three levels deep — for each state, for each coverage value, the average totality seconds.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "by_state": {
      "terms": { "field": "state.keyword", "size": 10 },
      "aggs": {
        "by_coverage": {
          "terms": { "field": "coverage.keyword", "size": 20 },
          "aggs": {
            "avg_seconds": {
              "avg": { "field": "totality_seconds" }
            }
          }
        }
      }
    }
  }
}
```
:warning: Bucket counts multiply. Four states × 20 coverage values = up to 80 buckets, and each extra level multiplies again. Keep `size` values deliberate.
</details>
<hr>

:question: Sort the state buckets by their average totality, rather than by document count.

<details>
  <summary>View Solution (click to reveal)</summary>

Reference the sub-aggregation by name in `order`:

```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "by_state": {
      "terms": {
        "field": "state.keyword",
        "size": 10,
        "order": { "avg_seconds": "desc" }
      },
      "aggs": {
        "avg_seconds": { "avg": { "field": "totality_seconds" } }
      }
    }
  }
}
```
Other `order` keys: `{ "_count": "desc" }` (the default) and `{ "_key": "asc" }` (alphabetical).
</details>
<hr>

## 2.6 Write and execute a query that searches across multiple clusters

:bulb: **Configuring** the remote cluster is objective 5.4. This objective is only about writing the query. If you have not set up a second cluster, read this now and come back after Part 5.

The scenario: each state's parks live on a different cluster, loaded from `example-date/unique-clusters.json`.

Syntax is `<remote_alias>:<index>`, comma separated, local indices with no prefix:

```json
// one remote
GET /cluster_one:totality-raw/_search
{
  "query": { "term": { "state.keyword": "Oklahoma" } }
}

// local + two remotes
GET /totality-raw,cluster_one:totality-raw,cluster_two:totality-raw/_search
{
  "query": { "match_all": {} }
}

// wildcards work on both sides of the colon
GET /*:totality-*,totality-*/_search
{
  "query": { "term": { "coverage.keyword": "100%" } }
}

// exclude a cluster
GET /*:totality-*,-cluster_three:totality-*/_search
{
  "query": { "match_all": {} }
}
```

:warning: Note the `/_search` on the end. Omitting it — as an earlier version of this file did — is a silent failure that returns index metadata instead of search results.

Verify it really went cross-cluster: the response contains a `_clusters` block with `total`, `successful` and `skipped`.

```json
// which clusters and indices does this pattern resolve to, and are they reachable? (8.13+)
GET _resolve/cluster/*:totality-*

// which remotes are configured and connected?
GET _remote/info
```

:bulb: Cross-cluster search combines well with async search — it is the recommended way to run long CCS queries:
```json
POST /totality-raw,cluster_one:totality-raw/_async_search?wait_for_completion_timeout=10ms&keep_on_completion=true
{ "size": 0, "aggs": { "by_state": { "terms": { "field": "state.keyword" } } } }
```

## 2.7 Write and execute a search that utilizes a runtime field

:bulb: **Defining** runtime fields with Painless is objective 4.5. This objective is about *searching* with one.

A runtime field is evaluated at query time, is not indexed, and never appears in `_source`.

:question: Return each park's total totality in seconds as a field called `total_time_seconds`, without changing the index.

<details>
  <summary>View Solution (click to reveal)</summary>

Define it inline under `runtime_mappings` — it exists for this one request only:

```json
GET totality-raw/_search
{
  "runtime_mappings": {
    "total_time_seconds": {
      "type": "long",
      "script": {
        "source": "emit((doc['totality_minutes'].value * 60) + doc['totality_seconds'].value)"
      }
    }
  },
  "size": 5,
  "fields": ["total_time_seconds", "name", "state"],
  "_source": false
}
```

:warning: **`fields` is how you get the value back.** A runtime field is never in `_source`, so without the `fields` parameter the query "works" and returns nothing visible. `"_source": false` makes that obvious in the output.
</details>
<hr>

:question: How many parks have more than 200 seconds of totality?

<details>
  <summary>View Solution (click to reveal)</summary>

You can query a runtime field exactly like a normal one:

```json
GET totality-raw/_search?filter_path=hits.total.value
{
  "track_total_hits": true,
  "runtime_mappings": {
    "total_time_seconds": {
      "type": "long",
      "script": {
        "source": "emit((doc['totality_minutes'].value * 60) + doc['totality_seconds'].value)"
      }
    }
  },
  "query": {
    "range": { "total_time_seconds": { "gt": 200 } }
  }
}
```
</details>
<hr>

:question: Aggregate parks into duration bands computed at query time, and sort the results by the runtime field.

<details>
  <summary>View Solution (click to reveal)</summary>

Aggregate:
```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "runtime_mappings": {
    "duration_band": {
      "type": "keyword",
      "script": {
        "source": """
          long total = (doc['totality_minutes'].value * 60) + doc['totality_seconds'].value;
          if (total == 0)        { emit('none');   }
          else if (total < 120)  { emit('short');  }
          else if (total < 210)  { emit('medium'); }
          else                   { emit('long');   }
        """
      }
    }
  },
  "aggs": {
    "bands": { "terms": { "field": "duration_band" } }
  }
}
```

Sort:
```json
GET totality-raw/_search
{
  "runtime_mappings": {
    "total_time_seconds": {
      "type": "long",
      "script": {
        "source": "emit((doc['totality_minutes'].value * 60) + doc['totality_seconds'].value)"
      }
    }
  },
  "size": 5,
  "sort": [ { "total_time_seconds": { "order": "desc" } } ],
  "fields": ["total_time_seconds", "name", "state"],
  "_source": false
}
```
</details>
<hr>

:bulb: If the runtime field is already in the index mapping (see 4.5), you search it with no `runtime_mappings` block at all — it behaves like any other field. A `runtime_mappings` definition in the search request **overrides** a same-named mapping field for that request only.

---
---

# Part 3 — Developing Search Applications

## 3.1 Sort the results of a query by a given set of requirements

:question: Return all Vermont parks sorted by minutes of totality, longest first.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search
{
  "query": {
    "term": { "state.keyword": "Vermont" }
  },
  "sort": [
    { "totality_minutes": { "order": "desc" } }
  ]
}
```

:warning: You cannot sort on an analysed `text` field — you get `Fielddata is disabled on text fields by default`. Sort on `state.keyword`, not `state`. The fix is always "use the keyword sub-field", not "enable fielddata".

:bulb: Once you add any `sort`, `_score` is no longer computed and comes back `null`. Add `"track_scores": true` if you need it.
</details>
<hr>

:question: Sort by minutes descending, then seconds descending, so parks with equal minutes are ordered correctly — and put parks missing the field last.

<details>
  <summary>View Solution (click to reveal)</summary>

Sort keys apply in array order; later keys break ties.

```json
GET totality-raw/_search
{
  "query": { "term": { "state.keyword": "Vermont" } },
  "size": 10,
  "sort": [
    { "totality_minutes": { "order": "desc", "missing": "_last" } },
    { "totality_seconds": { "order": "desc", "missing": "_last" } },
    { "name.keyword":     { "order": "asc" } }
  ],
  "_source": ["name", "totality_minutes", "totality_seconds"]
}
```

`missing` accepts `_first`, `_last`, or a literal value. `unmapped_type` stops the sort failing when one index in a pattern lacks the field.
</details>
<hr>

:question: Sort parks by their **total** totality time, which is not a field in the index.

<details>
  <summary>View Solution (click to reveal)</summary>

Compute it as a runtime field and sort on that — cleaner than a script sort in 8.x:

```json
GET totality-raw/_search
{
  "runtime_mappings": {
    "total_time_seconds": {
      "type": "long",
      "script": {
        "source": "emit((doc['totality_minutes'].value * 60) + doc['totality_seconds'].value)"
      }
    }
  },
  "size": 10,
  "sort": [ { "total_time_seconds": { "order": "desc" } } ],
  "fields": ["total_time_seconds", "name"],
  "_source": ["name", "state"]
}
```
</details>
<hr>

## 3.2 Implement pagination of the results of a search query

Three mechanisms, and the question wording tells you which one is wanted:

| Method | Use when | Limit |
| --- | --- | --- |
| `from` + `size` | shallow paging, page-numbered UI | `from + size` ≤ `index.max_result_window` (default 10000) |
| `search_after` + PIT | deep paging, exporting everything | none, needs a unique tiebreaker sort |
| `scroll` | legacy bulk export | deprecated in 8.x |

:question: Paginate the Vermont parks, 20 per page, showing page 3 (i.e. starting at result 40), sorted by coverage.

<details>
  <summary>View Solution (click to reveal)</summary>

Page N (1-based) with page size S is `"from": (N-1) * S, "size": S`. Page 3 at 20 per page is `from: 40`.

```json
GET totality-raw/_search
{
  "size": 20,
  "from": 40,
  "query": {
    "term": { "state.keyword": "Vermont" }
  },
  "sort": [
    { "coverage.keyword": { "order": "asc" } },
    { "name.keyword":     { "order": "asc" } }
  ],
  "_source": ["name", "city", "coverage"]
}
```

:warning: Always add a `sort`, and make it **deterministic**. Without a unique tiebreaker, two requests for the same page can return different documents, because ties are broken arbitrarily. Vermont has 51 parks, so page 3 returns the last 11.
</details>
<hr>

:question: Page through **all 190** parks in chunks of 50, consistently, in a way that would still work if there were 10 million.

<details>
  <summary>View Solution (click to reveal)</summary>

`search_after` with a point in time. Four steps.

**Step 1 — open a PIT.** This freezes the segment view so results do not shift while you page.
```json
POST /totality-raw/_pit?keep_alive=5m

// output
{ "id": "46ToAwMDaWR5BXV1aWQyKwZub2RlXzMAAAAAAAAAACoBYwADaWR4..." }
```

**Step 2 — first page.** With a PIT you put the id in the **body** and do **not** name the index in the URL. `_shard_doc` is the cheap built-in tiebreaker, available only with a PIT.
```json
GET /_search
{
  "size": 50,
  "query": { "match_all": {} },
  "pit": {
    "id": "46ToAwMDaWR5BXV1aWQyKwZub2RlXzMAAAAAAAAAACoBYwADaWR4...",
    "keep_alive": "5m"
  },
  "sort": [
    { "totality_minutes": "desc" },
    { "_shard_doc": "asc" }
  ],
  "_source": ["name", "state", "totality_minutes"]
}
```

**Step 3 — subsequent pages.** Take the `sort` array from the **last hit** of the previous page and pass it as `search_after`. Reuse the `id` from the response, which can change.
```json
GET /_search
{
  "size": 50,
  "query": { "match_all": {} },
  "pit": { "id": "46ToAwMDaWR5...", "keep_alive": "5m" },
  "sort": [
    { "totality_minutes": "desc" },
    { "_shard_doc": "asc" }
  ],
  "search_after": [ 3, 4294967298 ],
  "_source": ["name", "state", "totality_minutes"]
}
```
Repeat until a page returns fewer hits than `size`. With 190 documents that is four pages.

**Step 4 — close it.**
```json
DELETE /_pit
{ "id": "46ToAwMDaWR5..." }
```
</details>
<hr>

:question: A request for `"from": 10000` fails. What are the two possible answers, and which is correct?

<details>
  <summary>View Solution (click to reveal)</summary>

The error is `Result window is too large, from + size must be less than or equal to: [10000]`.

```json
// (a) raise the ceiling - blunt, costs heap, only right if the question explicitly asks for it
PUT totality-raw/_settings
{ "index.max_result_window": 50000 }

// (b) use search_after with a PIT - the correct answer to "page beyond 10,000 results"
```

If the question says "deep pagination" or "retrieve all results", it wants (b).
</details>
<hr>

:bulb: Paginating **aggregation buckets** is a different problem — use the composite aggregation and its `after_key`:
```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "parks": {
      "composite": {
        "size": 10,
        "sources": [
          { "state": { "terms": { "field": "state.keyword" } } },
          { "city":  { "terms": { "field": "city.keyword"  } } }
        ]
      }
    }
  }
}
```
Feed the returned `after_key` back in as `"after": { ... }` for the next page.

## 3.3 Define and use index aliases

:question: Define an alias `totality-all` pointing at `totality-raw`.

<details>
  <summary>View Solution (click to reveal)</summary>

The `_aliases` API is the one to learn — it is atomic and can add and remove in one call.

```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "totality-raw",
        "alias": "totality-all"
      }
    }
  ]
}

// output
{ "acknowledged": true }
```

Verify the document count matches the underlying index:
```json
GET totality-all/_count

// expect
{ "count": 190 }
```
</details>
<hr>

:question: Define a **filtered** alias `totality-full` that shows only the parks with 100% coverage.

<details>
  <summary>View Solution (click to reveal)</summary>

**1. Check the field type first** — this determines whether you filter on `coverage` or `coverage.keyword`:
```json
GET totality-raw/_mapping/field/coverage

// output - dynamically mapped, so text with a keyword sub-field
{
  "totality-raw": {
    "mappings": {
      "coverage": {
        "full_name": "coverage",
        "mapping": {
          "coverage": {
            "type": "text",
            "fields": {
              "keyword": { "type": "keyword", "ignore_above": 256 }
            }
          }
        }
      }
    }
  }
}
```

**2. Apply the alias with the filter.** Use the `.keyword` sub-field — a `term` query against the analysed `text` field returns zero results, because `"100%"` is analysed to the token `100`.
```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "totality-raw",
        "alias": "totality-full",
        "filter": {
          "term": { "coverage.keyword": "100%" }
        }
      }
    }
  ]
}
```

**3. Test it.**
```json
GET totality-full/_count

// Answer: 51
```

**4. Confirm the alias definition:**
```json
GET _alias/totality-full
GET totality-raw/_alias
GET _cat/aliases?v
```

**Bonus** — the same thing as a plain query:
```json
POST totality-raw/_search?filter_path=hits.total.value
{
  "query": { "term": { "coverage.keyword": "100%" } }
}

// { "hits": { "total": { "value": 51 } } }
```

Other useful filtered aliases on this data: one per state, one per timezone, one for parks with any totality at all.
</details>
<hr>

:question: You have reindexed `totality-raw` into an improved `totality-v2`. Switch the `totality-all` alias over with **zero downtime**.

<details>
  <summary>View Solution (click to reveal)</summary>

This is why the `_aliases` API is atomic — both actions land in a single cluster state update, so no request ever sees zero indices:

```json
POST /_aliases
{
  "actions": [
    { "remove": { "index": "totality-raw", "alias": "totality-all" } },
    { "add":    { "index": "totality-v2",  "alias": "totality-all" } }
  ]
}
```

Doing this as two separate calls leaves a window where `totality-all` resolves to nothing. That gap is the entire point of the question.
</details>
<hr>

:bulb: Other alias syntax worth knowing:
```json
// an alias over several indices needs exactly one write index
POST /_aliases
{
  "actions": [
    { "add": { "index": "totality-2024-vermont", "alias": "totality-2024" } },
    { "add": { "index": "totality-2024-maine",   "alias": "totality-2024", "is_write_index": true } }
  ]
}

// aliases declared in an index template
PUT _index_template/totality-2024-tmpl
{
  "index_patterns": ["totality-2024-*"],
  "priority": 200,
  "template": {
    "aliases": {
      "totality-all":  {},
      "totality-full": { "filter": { "term": { "coverage.keyword": "100%" } } }
    }
  }
}

// shorthand - fine, but _aliases is safer to memorise
PUT    totality-raw/_alias/totality-all
DELETE totality-raw/_alias/totality-all
```

:warning: A filtered alias filters **reads only**. Documents indexed through `totality-full` are not rejected for having 95% coverage — they just will not be visible through that alias afterwards.

---
---

# Part 4 — Data Processing

## 4.1 Define a mapping that satisfies a given set of requirements

### Dynamic vs explicit mapping

> **Dynamic mapping** lets you experiment without declaring anything — Elasticsearch adds fields automatically as documents arrive.
>
> **Explicit mapping** lets you choose precisely: which strings are full text, which fields are numbers/dates/geo, date formats, and custom rules for dynamically added fields.

The requirement wording maps almost one-to-one onto mapping parameters:

| Requirement wording | Parameter |
| --- | --- |
| "not aggregatable / not sortable" | `"doc_values": false` |
| "stored but not searchable" | `"index": false` |
| "ignore values longer than N" | `"ignore_above": N` |
| "do not reject malformed values" | `"ignore_malformed": true` |
| "accept these date formats" | `"format": "yyyy-MM-dd||epoch_millis"` |
| "use this value when missing" | `"null_value": "UNKNOWN"` |
| "searchable under a second name" | `"copy_to": "combined"` |
| "reject unknown fields" | `"dynamic": "strict"` |

:question: Create an index `oklahoma` with an explicit mapping for the eclipse data, where `zip_code` is a keyword that is **not** aggregatable, and `eclipse_date` accepts both `yyyy-MM-dd` and epoch millis.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT /oklahoma
{
  "settings": {
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "name":           { "type": "keyword" },
      "street_address": { "type": "keyword" },
      "city":           { "type": "keyword" },
      "state":          { "type": "keyword" },
      "zip_code": {
        "type": "keyword",
        "doc_values": false
      },
      "timezone":         { "type": "keyword" },
      "coverage":         { "type": "keyword" },
      "eclipse_date": {
        "type": "date",
        "format": "yyyy-MM-dd||epoch_millis"
      },
      "totality_minutes": { "type": "integer" },
      "totality_seconds": { "type": "integer" },
      "partial_start_time": { "type": "date" },
      "max_time":           { "type": "date" },
      "partial_end_time":   { "type": "date" }
    }
  }
}

// output
{ "acknowledged": true, "shards_acknowledged": true, "index": "oklahoma" }
```

**Why `zip_code` is a keyword and not a number:** identifiers like zip codes, ISBNs and product IDs are almost never used in range queries, but are very often fetched with term-level queries — which are faster on keyword fields. And `"03246"` as a number loses its leading zero. If you genuinely need both, use a multi-field (see 4.2).

**Why `doc_values: false` here:** doc values are what make a field sortable, aggregatable and readable from a script. Disabling them saves disk when you know you will only ever do term lookups. It is also the answer to "make this field searchable but not aggregatable".
</details>
<hr>

:question: Reindex only the Oklahoma parks from `totality-raw` into `oklahoma`, then verify only Oklahoma is present.

<details>
  <summary>View Solution (click to reveal)</summary>

Write the query first, confirm the count, then convert it into a `_reindex`:

```json
// 1. check the query
GET totality-raw/_count
{
  "query": { "term": { "state.keyword": "Oklahoma" } }
}
// { "count": 39 }

// 2. convert it
POST _reindex
{
  "source": {
    "index": "totality-raw",
    "query": { "term": { "state.keyword": "Oklahoma" } }
  },
  "dest": { "index": "oklahoma" }
}

// output
{
  "took": 405,
  "total": 39,
  "created": 39,
  "updated": 0,
  "failures": []
}
```

**3. Verify.** If only one state is present, a terms aggregation returns exactly one bucket:
```json
GET oklahoma/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": { "states": { "terms": { "field": "state" } } }
}
// one bucket: Oklahoma, doc_count 39
```

:warning: In `oklahoma`, `state` is an explicit `keyword`, so you aggregate on `state` directly — **not** `state.keyword`. In `totality-raw` it was dynamically mapped as `text` + `.keyword`. Getting this wrong is the single most common error when working across both indices, and the symptom is either a `Fielddata is disabled` error or zero results.

Or prove the negative:
```json
GET oklahoma/_count
{
  "query": {
    "bool": { "must_not": [ { "term": { "state": "Oklahoma" } } ] }
  }
}
// expect count: 0
```
</details>
<hr>

:bulb: **Changing a mapping later.** Adding a new field is allowed; changing an existing field's type is not. If a question asks you to change a field type, the answer is always *create a new index with the correct mapping and reindex*.
```json
PUT oklahoma/_mapping
{ "properties": { "state_abbrev": { "type": "keyword" } } }
```

## 4.2 Define and use multi-fields with different data types and/or analyzers

> :books: "Define and use a custom analyzer" is no longer a separate objective, but you cannot do this one without it, so it is covered inline.

Three things a question can ask for, often combined: a field indexed with **different types** (text for search, keyword for aggregation), with **different analyzers** (standard plus English stemming), or with a **normalizer** for case-insensitive exact matching.

:question: Create an index `totality-names` where the park `name` field is:
- full-text searchable with the standard analyzer
- also searchable with English stemming, under `name.english`
- also exactly aggregatable, under `name.raw`
- also matchable case-insensitively as an exact value, under `name.ci`

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT totality-names
{
  "settings": {
    "number_of_replicas": 0,
    "analysis": {
      "normalizer": {
        "lowercase_normalizer": {
          "type": "custom",
          "char_filter": [],
          "filter": ["lowercase", "asciifolding"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "english": { "type": "text",    "analyzer": "english" },
          "raw":     { "type": "keyword", "ignore_above": 256 },
          "ci":      { "type": "keyword", "normalizer": "lowercase_normalizer" }
        }
      },
      "state":            { "type": "keyword" },
      "totality_minutes": { "type": "integer" }
    }
  }
}
```

Populate and test each path:
```json
POST _reindex
{
  "source": { "index": "totality-raw" },
  "dest":   { "index": "totality-names" }
}

// full text
GET totality-names/_search?filter_path=hits.total.value
{ "query": { "match": { "name": "state park" } } }

// exact value, case sensitive
GET totality-names/_search?filter_path=hits.total.value
{ "query": { "term": { "name.raw": "Ahern State Park" } } }

// exact value, case INsensitive - this is what name.ci buys you
GET totality-names/_search?filter_path=hits.total.value
{ "query": { "term": { "name.ci": "ahern state park" } } }

// aggregate on the keyword sub-field
GET totality-names/_search?filter_path=aggregations
{ "size": 0, "aggs": { "names": { "terms": { "field": "name.raw", "size": 5 } } } }
```

Always verify an analyzer before committing to it:
```json
GET totality-names/_analyze
{ "field": "name.english", "text": "Ahern State Parks" }
```
</details>
<hr>

:question: Write a custom analyzer that rewrites `State Park` to `SP` in the park name, and apply it to a new index `totality-sp`.

<details>
  <summary>View Solution (click to reveal)</summary>

**Step 1 — test the char filter standalone**, before wiring it into an index:
```json
POST _analyze
{
  "char_filter": [
    {
      "type": "pattern_replace",
      "pattern": "State Park",
      "replacement": "SP"
    }
  ],
  "tokenizer": "keyword",
  "text": [
    "Ahern State Park",
    "Bear Brook State Park",
    "Androscoggin Wayside Park"
  ]
}

// output - tokens: "Ahern SP", "Bear Brook SP", "Androscoggin Wayside Park"
```
:bulb: `"tokenizer": "keyword"` keeps each input as a single token so you can see the replacement clearly. With `standard` you would get individual words.

**Step 2 — put it together.** The analyzer name in the mapping must match the name in `settings.analysis.analyzer`, and the char filter name must match the one in `settings.analysis.char_filter`:

```json
PUT totality-sp
{
  "settings": {
    "number_of_replicas": 0,
    "analysis": {
      "analyzer": {
        "sp_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "char_filter": ["state_park_filter"],
          "filter": ["lowercase"]
        }
      },
      "char_filter": {
        "state_park_filter": {
          "type": "pattern_replace",
          "pattern": "State Park",
          "replacement": "SP"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "sp_analyzer",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      },
      "state":            { "type": "keyword" },
      "totality_minutes": { "type": "integer" }
    }
  }
}
```

**Step 3 — reindex into it:**
```json
POST _reindex
{
  "source": { "index": "totality-raw" },
  "dest":   { "index": "totality-sp" }
}
```

**Step 4 — verify.** Searching for `sp` finds the parks:
```json
GET totality-sp/_search?filter_path=hits.total.value
{ "query": { "match": { "name": "sp" } } }
```

:warning: **You will get `Ahern State Park` back in `_source`, not `Ahern SP`** — and that is correct, not a bug. `_source` is the raw JSON you sent; analyzers only affect the inverted index. You can search for the transformed token but never retrieve it from `_source`.

Prove what was actually indexed:
```json
GET totality-sp/_termvectors/<doc_id>?fields=name
```
</details>
<hr>

:question: How would you index with an edge-ngram analyzer for autocomplete but search with the standard analyzer?

<details>
  <summary>View Solution (click to reveal)</summary>

`search_analyzer` overrides `analyzer` at query time. Without it, your search terms get ngrammed too and everything matches everything — the classic autocomplete bug:

```json
PUT totality-autocomplete
{
  "settings": {
    "analysis": {
      "analyzer": {
        "autocomplete_index": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "edge_ngrams"]
        }
      },
      "filter": {
        "edge_ngrams": { "type": "edge_ngram", "min_gram": 2, "max_gram": 15 }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "autocomplete_index",
        "search_analyzer": "standard"
      }
    }
  }
}
```
</details>
<hr>

:bulb: **Analyzer anatomy** — exactly three stages, in this order:

| Stage | Count | Examples |
| --- | --- | --- |
| `char_filter` | 0..n | `html_strip`, `mapping`, `pattern_replace` |
| `tokenizer` | exactly 1 | `standard`, `keyword`, `whitespace`, `pattern`, `edge_ngram`, `path_hierarchy` |
| `filter` | 0..n | `lowercase`, `stop`, `stemmer`, `synonym`, `asciifolding` |

## 4.3 Use the Reindex API and Update By Query API to reindex and/or update documents

| Question says | Use |
| --- | --- |
| "copy / migrate into a new index (with a new mapping)" | `_reindex` |
| "change documents **in place**" | `_update_by_query` |
| "apply a changed mapping to existing documents" | `_update_by_query` with no body |
| "run documents through a pipeline" | either, with `?pipeline=` or `dest.pipeline` |
| "remove documents matching X" | `_delete_by_query` |

:question: Reindex `totality-raw` into `totality-state-parks-raw`, then reindex **that** into `totality-state-parks-full` containing only the 100% coverage parks.

<details>
  <summary>View Solution (click to reveal)</summary>

:warning: `_reindex` requires `_source` to be enabled on the source index. Check your templates are not forcing `"_source": { "enabled": false }` — that breaks reindexing entirely.

```json
POST _reindex
{
  "source": { "index": "totality-raw" },
  "dest":   { "index": "totality-state-parks-raw" }
}

GET totality-state-parks-raw/_count?filter_path=count
// { "count": 190 }
```

Now the filtered one. Run the query first, confirm the count, then convert:
```json
POST _reindex
{
  "source": {
    "index": "totality-state-parks-raw",
    "query": { "term": { "coverage.keyword": "100%" } }
  },
  "dest": { "index": "totality-state-parks-full" }
}

GET totality-state-parks-full/_count?filter_path=count
// { "count": 51 }
```

Verify only 100% parks made it:
```json
GET totality-state-parks-full/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": { "coverages": { "terms": { "field": "coverage.keyword" } } }
}
// one bucket: "100%" -> 51
```

:warning: `_reindex` does **not** copy settings, mappings, aliases, or the ILM policy. Create the destination index (or a matching template) first if the mapping matters — otherwise you get dynamic mapping and, for example, `zip_code` becomes a `long`.
</details>
<hr>

:question: Set `totality_minutes` to 99 for every Vermont park, then verify one of them changed.

<details>
  <summary>View Solution (click to reveal)</summary>

Look at a document first so you have something to compare against:
```json
GET totality-raw/_search?filter_path=hits.hits._source
{
  "size": 1,
  "query": { "term": { "state.keyword": "Vermont" } }
}
```

Update in place. Note `ctx._source.field` — this is a reindex/UBQ script, not an ingest processor:
```json
POST totality-raw/_update_by_query
{
  "query": {
    "term": { "state.keyword": "Vermont" }
  },
  "script": {
    "lang": "painless",
    "source": "ctx._source.totality_minutes = 99"
  }
}

// output - check "updated" matches the expected count
{
  "took": 369,
  "total": 51,
  "updated": 51,
  "version_conflicts": 0,
  "failures": []
}
```

Verify:
```json
GET totality-raw/_search?filter_path=hits.hits._source
{
  "size": 1,
  "query": {
    "term": { "street_address.keyword": "151 Coon Point Rd" }
  }
}

// "totality_minutes": 99
```

:bonus: Undo it by reindexing from a clean copy, or reload the bulk file — that is the honest answer, since `_update_by_query` has no undo.
</details>
<hr>

:bulb: Options worth knowing on both APIs:
```json
POST _reindex?wait_for_completion=false&refresh=true
{
  "conflicts": "proceed",
  "max_docs": 1000,
  "source": {
    "index": "totality-raw",
    "query":  { "term": { "state.keyword": "Maine" } },
    "_source": ["name", "city", "coverage"],
    "size": 1000
  },
  "dest": {
    "index": "totality-maine",
    "op_type": "create",
    "pipeline": "totality-ingest"
  },
  "script": { "lang": "painless", "source": "ctx._source.migrated = true" }
}
```
`wait_for_completion=false` returns a **task id** immediately — the right answer for "reindex a large index without blocking":
```json
GET  _tasks/<task_id>
GET  _tasks?actions=*reindex&detailed
POST _tasks/<task_id>/_cancel
```

:warning: **`ctx` vs `ctx._source`.** In `_reindex` / `_update_by_query` / `_update` scripts you write `ctx._source.field`. In an ingest pipeline `script` processor you write `ctx.field`. Mixing them up is the most common silent failure in this whole section.

## 4.4 Define and use an ingest pipeline that satisfies a given set of requirements

:question: Build a pipeline `totality-ingest` that:
- adds a tag `pipeline_ingest` to show the document went through the pipeline
- normalises the `coverage_percent` field into `coverage` so the whole index is consistent
- combines the minutes and seconds into a `full_duration` display string
- adds a `full_totality_seconds` numeric field
- appends `State Park` to the name of any park that does not already have it

<details>
  <summary>View Solution (click to reveal)</summary>

**Step 1 — always simulate first.** Pass in two representative documents: one with `coverage`, one with the odd `coverage_percent`.

```json
POST _ingest/pipeline/_simulate?verbose
{
  "pipeline": {
    "processors": [
      {
        "append": {
          "tag": "add ingest marker",
          "field": "tags",
          "value": ["pipeline_ingest"]
        }
      },
      {
        "rename": {
          "tag": "normalise coverage_percent",
          "field": "coverage_percent",
          "target_field": "coverage",
          "ignore_missing": true
        }
      },
      {
        "set": {
          "tag": "human readable duration",
          "field": "full_duration",
          "value": "{{{totality_minutes}}}:{{{totality_seconds}}}"
        }
      },
      {
        "script": {
          "tag": "total seconds",
          "lang": "painless",
          "source": "ctx.full_totality_seconds = (ctx.totality_minutes * 60) + ctx.totality_seconds"
        }
      },
      {
        "script": {
          "tag": "ensure State Park suffix",
          "if": "ctx.name != null && !ctx.name.contains('State Park')",
          "lang": "painless",
          "source": "ctx.name = ctx.name + ' State Park'"
        }
      }
    ]
  },
  "docs": [
    {
      "_source": {
        "name": "Tenkiller",
        "street_address": "OK-100",
        "city": "Vian",
        "state": "Oklahoma",
        "zip_code": "74962",
        "timezone": "CDT",
        "coverage": "100%",
        "eclipse_date": "2024-04-08",
        "totality_minutes": 3,
        "totality_seconds": 54,
        "partial_start_time": "2024-04-08T12:28:11.000Z",
        "max_time": "2024-04-08T13:47:24.000Z",
        "partial_end_time": "2024-04-08T15:06:45.000Z"
      }
    },
    {
      "_source": {
        "name": "Ahern State Park",
        "city": "Laconia",
        "state": "New Hampshire",
        "zip_code": "03246",
        "timezone": "EDT",
        "coverage_percent": "97.47%",
        "eclipse_date": "2024-04-08",
        "totality_minutes": 0,
        "totality_seconds": 0
      }
    }
  ]
}
```

Expected for the first document: `name` becomes `Tenkiller State Park`, `full_duration` is `3:54`, `full_totality_seconds` is `234`, `tags` is `["pipeline_ingest"]`.
Expected for the second: `coverage_percent` is gone and `coverage` is `97.47%`, `name` is unchanged, `full_totality_seconds` is `0`.

:bulb: `?verbose` shows the document after **each** processor. When a five-processor pipeline produces the wrong answer, this is how you find which step broke it.

**Step 2 — save the working pipeline.** Copy the `processors` array verbatim:
```json
PUT _ingest/pipeline/totality-ingest
{
  "description": "normalise and enrich eclipse totality data",
  "processors": [
    { "append": { "tag": "add ingest marker", "field": "tags", "value": ["pipeline_ingest"] } },
    { "rename": { "tag": "normalise coverage_percent", "field": "coverage_percent", "target_field": "coverage", "ignore_missing": true } },
    { "set":    { "tag": "human readable duration", "field": "full_duration", "value": "{{{totality_minutes}}}:{{{totality_seconds}}}" } },
    {
      "script": {
        "tag": "total seconds",
        "lang": "painless",
        "source": "ctx.full_totality_seconds = (ctx.totality_minutes * 60) + ctx.totality_seconds"
      }
    },
    {
      "script": {
        "tag": "ensure State Park suffix",
        "if": "ctx.name != null && !ctx.name.contains('State Park')",
        "lang": "painless",
        "source": "ctx.name = ctx.name + ' State Park'"
      }
    }
  ],
  "on_failure": [
    { "set": { "field": "ingest_error", "value": "{{{_ingest.on_failure_message}}}" } }
  ]
}
```

**Step 3 — apply it to the existing data:**
```json
POST totality-raw/_update_by_query?pipeline=totality-ingest&refresh

// output - check "updated"
{
  "took": 188,
  "total": 190,
  "updated": 190,
  "version_conflicts": 0,
  "failures": []
}
```

**Step 4 — verify.** The `coverage_percent` document should be gone:
```json
GET totality-raw/_count
{ "query": { "exists": { "field": "coverage_percent" } } }
// expect count: 0

GET totality-raw/_count
{ "query": { "exists": { "field": "coverage" } } }
// expect count: 190
```

And spot-check a document:
```json
GET totality-raw/_search?filter_path=hits.hits._source
{
  "size": 1,
  "query": { "term": { "name.keyword": "Ahern State Park" } }
}
```
</details>
<hr>

:warning: The value of `ctx` is **read-only inside `if` conditions** — you can test it, but you cannot assign to it. Use a `script` processor when you need to modify a document conditionally.

### Processors worth recognising on sight

| Processor | Does |
| --- | --- |
| `set` / `append` / `remove` / `rename` | the basics |
| `convert` | change type: `integer`, `long`, `float`, `double`, `string`, `boolean`, `ip`, `auto` |
| `gsub` | regex replace inside a string |
| `split` / `join` | string ↔ array |
| `trim`, `lowercase`, `uppercase` | string cleanup |
| `grok` / `dissect` | parse unstructured text into fields |
| `date` | parse a string into a timestamp |
| `date_index_name` | route a document to a date-based index |
| `json` / `kv` / `dot_expander` | structure parsing |
| `script` | anything the above cannot do |
| `fail` / `drop` | reject or silently discard |
| `pipeline` | call another pipeline |
| `foreach` | run a processor over every array element |
| `enrich` | join in data from another index |

### The four ways to use a pipeline

```json
// 1. per request
POST totality-raw/_doc?pipeline=totality-ingest
{ "name": "Test", "totality_minutes": 1, "totality_seconds": 0 }

// 2. in place on an existing index
POST totality-raw/_update_by_query?pipeline=totality-ingest

// 3. as part of a reindex
POST _reindex
{ "source": { "index": "a" }, "dest": { "index": "b", "pipeline": "totality-ingest" } }

// 4. as an index default, so every write uses it automatically
PUT totality-raw/_settings
{ "index.default_pipeline": "totality-ingest" }
```
:bulb: `index.final_pipeline` runs *after* the default pipeline and after any `?pipeline=` parameter — use it for "always stamp this field no matter what".

Housekeeping:
```json
GET    _ingest/pipeline
GET    _ingest/pipeline/totality-ingest
DELETE _ingest/pipeline/totality-ingest
```

## 4.5 Define runtime fields to retrieve custom values using Painless scripting

:sparkles: **New objective since 8.1.** Objective 2.7 was about *searching* with a runtime field; this one is about *defining* it correctly.

### The Painless rules for runtime fields

1. You **must** call `emit(...)`. A script that `return`s a value instead of emitting one produces nothing.
2. Read values with `doc['field']` (doc values, fast) or `params._source['field']` (raw JSON, slower but works on fields without doc values).
3. `doc['field'].value` **throws** when the field is missing. Guard with `doc['field'].size() != 0` or `return;` early.
4. You may `emit()` more than once — runtime fields can be multi-valued.
5. `date` runtime fields emit **epoch milliseconds**.
6. Use `"""..."""` for multi-line scripts so you do not have to escape quotes.

Supported types: `boolean`, `composite`, `date`, `double`, `geo_point`, `ip`, `keyword`, `long`, `lookup`.

### Three places to define one

:question: 1. Define `total_time_seconds` in the mapping **when the index is created**.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT totality-raw-runtime
{
  "mappings": {
    "runtime": {
      "total_time_seconds": {
        "type": "long",
        "script": {
          "source": "emit((doc['totality_minutes'].value * 60) + doc['totality_seconds'].value)"
        }
      }
    },
    "properties": {
      "name":             { "type": "keyword" },
      "state":            { "type": "keyword" },
      "totality_minutes": { "type": "long" },
      "totality_seconds": { "type": "long" }
    }
  }
}

// output
{ "acknowledged": true, "shards_acknowledged": true, "index": "totality-raw-runtime" }
```

Load data into it. Use `example-date/put-runtime-fields.json`, whose action lines have no `_index`, so the index comes from the URL:
```json
POST /totality-raw-runtime/_bulk?refresh
{ "index": {}}
{"name":"Ahern State Park","street_address":"43 Great Bay Lane","city":"Laconia","state":"New Hampshire","zip_code":"03246","timezone":"EDT","coverage_percent":"97.47%","eclipse_date":"2024-04-08","totality_minutes":0,"totality_seconds":0,"partial_start_time":"2024-04-08T14:16:03.000Z","max_time":"2024-04-08T15:29:32.000Z","partial_end_time":"2024-04-08T16:38:48.000Z"}
{ "index": {}}
{"name":"Androscoggin Wayside Park","street_address":"Route 16","city":"Milan","state":"New Hampshire","zip_code":"03588","timezone":"EDT","coverage":"100%","eclipse_date":"2024-04-08","totality_minutes":2,"totality_seconds":5,"partial_start_time":"2024-04-08T14:17:11.000Z","max_time":"2024-04-08T15:30:07.000Z","partial_end_time":"2024-04-08T16:38:57.000Z"}
```
</details>
<hr>

:question: 2. Add the same field to the **existing** `totality-raw` index, without reindexing.

<details>
  <summary>View Solution (click to reveal)</summary>

This is the headline feature — it applies to every document already in the index and takes effect immediately:

```json
PUT totality-raw/_mapping
{
  "runtime": {
    "total_time_seconds": {
      "type": "long",
      "script": {
        "source": """
          if (doc['totality_minutes'].size() == 0 || doc['totality_seconds'].size() == 0) { return; }
          emit((doc['totality_minutes'].value * 60) + doc['totality_seconds'].value);
        """
      }
    }
  }
}

// output
{ "acknowledged": true }
```

:warning: There is **no** `params` block of field *types*. `params` in a runtime script is for constants you pass in, and it is not needed here at all.

Use it — remembering that runtime fields are never in `_source`:
```json
GET totality-raw/_search
{
  "fields": ["total_time_seconds", "name"],
  "_source": false,
  "size": 2
}
```

Remove it just as cheaply — `null` deletes it:
```json
PUT totality-raw/_mapping
{ "runtime": { "total_time_seconds": null } }
```
</details>
<hr>

:question: 3. Define it in a **single search request** only.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "runtime_mappings": {
    "total_time_seconds": {
      "type": "long",
      "script": {
        "source": "emit((doc['totality_minutes'].value * 60) + doc['totality_seconds'].value)"
      }
    }
  },
  "aggs": {
    "durations": {
      "terms": { "field": "total_time_seconds", "size": 20 }
    }
  }
}
```
Nothing is persisted; the field exists for this request only.
</details>
<hr>

### More runtime field exercises on this data

:question: Define a `keyword` runtime field `state_code` that abbreviates the state name — NH, VT, ME, OK.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT totality-raw/_mapping
{
  "runtime": {
    "state_code": {
      "type": "keyword",
      "script": {
        "source": """
          if (doc['state.keyword'].size() == 0) { return; }
          String s = doc['state.keyword'].value;
          if (s == 'New Hampshire') { emit('NH'); }
          else if (s == 'Vermont')  { emit('VT'); }
          else if (s == 'Maine')    { emit('ME'); }
          else if (s == 'Oklahoma') { emit('OK'); }
          else                      { emit('??'); }
        """
      }
    }
  }
}
```

Test:
```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": { "codes": { "terms": { "field": "state_code" } } }
}
// NH 67, VT 51, OK 39, ME 33
```

:warning: `doc['state']` would fail — `state` is analysed `text` with no doc values. Always use the `.keyword` sub-field in `doc[]`.
</details>
<hr>

:question: Define a `boolean` runtime field `in_totality` — true when the park has any totality at all.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET totality-raw/_search?filter_path=aggregations
{
  "size": 0,
  "runtime_mappings": {
    "in_totality": {
      "type": "boolean",
      "script": {
        "source": """
          long total = (doc['totality_minutes'].value * 60) + doc['totality_seconds'].value;
          emit(total > 0);
        """
      }
    }
  },
  "aggs": {
    "totality": { "terms": { "field": "in_totality" } }
  }
}
```
</details>
<hr>

:question: Define a `date` runtime field `totality_start_time` — the maximum eclipse time minus half the totality duration.

<details>
  <summary>View Solution (click to reveal)</summary>

Date runtime fields emit epoch milliseconds:

```json
GET totality-raw/_search
{
  "runtime_mappings": {
    "totality_start_time": {
      "type": "date",
      "script": {
        "source": """
          if (doc['max_time'].size() == 0) { return; }
          long halfMillis = (((doc['totality_minutes'].value * 60) + doc['totality_seconds'].value) * 1000L) / 2;
          emit(doc['max_time'].value.toInstant().toEpochMilli() - halfMillis);
        """
      }
    }
  },
  "size": 3,
  "query": { "range": { "totality_minutes": { "gt": 0 } } },
  "fields": ["totality_start_time", "max_time", "name"],
  "_source": false
}
```
:bulb: Note the `1000L` — integer arithmetic overflows silently in Painless if you are not careful with types.
</details>
<hr>

:bulb: **`"dynamic": "runtime"`** maps unknown fields as runtime fields rather than indexing them — searchable at zero index cost:
```json
PUT totality-runtime-dyn
{
  "mappings": {
    "dynamic": "runtime",
    "properties": { "eclipse_date": { "type": "date" } }
  }
}
```

:bulb: **Promoting a runtime field.** If it turns out to be queried constantly, add it as a real indexed field in the mapping and reindex. Queries do not change — the indexed field simply shadows the runtime one.

---
---

# Part 5 — Cluster Management

## 5.1 Diagnose shard issues and repair a cluster's health

### Set up a broken index to practise on

```json
DELETE broken_index

PUT broken_index
{
  "settings": {
    "number_of_replicas": 2,
    "number_of_shards": 2
  }
}

PUT broken_index/_doc/1
{ "field1": "data" }
```
On a single-node cluster this is permanently yellow — two replicas with nowhere to go.

### :sparkles: Start with the Health API

Added in 8.7 and the fastest route to a diagnosis. It does not just report a colour; it names the cause and gives you the remediation and a doc link.

```json
GET _health_report

// or one indicator at a time
GET _health_report/shards_availability
GET _health_report/disk
GET _health_report/ilm
GET _health_report/slm
```

Then the classics:
```json
GET _cluster/health
GET _cat/health?v

GET /_cat/indices?v
GET _cat/indices?v&health=red
GET _cat/indices?v&health=yellow
GET _cat/shards/broken_index?v&s=index
```

| Status | Meaning |
| --- | --- |
| **green** | all primaries and replicas assigned |
| **yellow** | all primaries assigned, some **replica** unassigned — data intact, availability reduced |
| **red** | some **primary** unassigned — data unavailable |

### Explain the allocation

```json
GET _cluster/allocation/explain
{
  "index": "broken_index",
  "shard": 0,
  "primary": false
}
```

:warning: You must supply `index`, `shard` **and** `primary`, or you get
`Validation Failed: 1: shard must be specified;2: primary must be specified;`

:bulb: With an **empty body** Elasticsearch picks an arbitrary unassigned shard for you, which is usually what you want:
```json
GET _cluster/allocation/explain
```

Read `unassigned_info.reason`, `can_allocate`, `allocate_explanation`, and — most importantly — `node_allocation_decisions[].deciders[]`.

| Decider says | Real cause | Fix |
| --- | --- | --- |
| `a copy of this shard is already allocated to this node` | more replicas than nodes | reduce `number_of_replicas` or add a node |
| `the node is above the high watermark...disk.watermark.high` | disk full | free space or raise the watermark |
| `node does not match index setting [index.routing.allocation...]` | tier/attribute filter unsatisfiable | fix the routing setting or add a matching node |
| `shard has exceeded the maximum number of retries [5]` | repeated failure | fix the cause, then `POST _cluster/reroute?retry_failed=true` |

### Repair

```json
// the single-node yellow fix
PUT /broken_index/_settings
{ "number_of_replicas": 0 }

GET _cat/indices/broken_index?v
// health should now be green
```

The rest of the toolbox:
```json
// apply to every index at once
PUT /*/_settings
{ "index.number_of_replicas": 0 }

// a shard failed 5 times and gave up
POST _cluster/reroute?retry_failed=true

// disk watermarks - the #1 real-world cause of unassigned shards
GET _cat/allocation?v
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low":  "85%",
    "cluster.routing.allocation.disk.watermark.high": "90%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
  }
}

// flood stage sets indices read-only-allow-delete; clear the block after freeing space
PUT /*/_settings
{ "index.blocks.read_only_allow_delete": null }

// is allocation switched off entirely?
PUT _cluster/settings
{ "persistent": { "cluster.routing.allocation.enable": "all" } }

// when the cluster is just slow
GET _cluster/pending_tasks
GET _nodes/hot_threads
GET _cat/nodes?v&h=name,node.role,heap.percent,ram.percent,cpu,disk.avail
```

:warning: Last resort only, and it **loses data**: `POST _cluster/reroute` with `allocate_empty_primary` / `allocate_stale_primary` and `"accept_data_loss": true`. Know it exists; never reach for it before the health API and allocation explain.

## 5.2 Backup and restore a cluster and/or specific indices

> You cannot back up an Elasticsearch cluster by copying its data directories — the only reliable way is snapshot and restore.

A complete backup covers the data, the cluster configuration, and the security configuration.

### Register a repository

`path.repo` must be set in `elasticsearch.yml` on **every** node, to an identical path.

```json
GET /_nodes?filter_path=nodes.*.settings.path.repo

PUT /_snapshot/totality_repo
{
  "type": "fs",
  "settings": {
    "location": "/tmp/totality_repo",
    "compress": true
  }
}

POST /_snapshot/totality_repo/_verify
```

| Type | Notes |
| --- | --- |
| `fs` | shared filesystem — needs `path.repo` on every node |
| `s3`, `gcs`, `azure` | plugins bundled in 8.x, credentials go in the keystore |
| `url` | read-only |
| `source` | `_source` only — smaller, needs a reindex to restore |

:warning: `/tmp` is fine for a lab and wrong for production — use NFS, S3, GCS or Azure.

### Take a snapshot

Date math in a URL path must be URI-encoded: `<` → `%3C`, `>` → `%3E`, `/` → `%2F`, `{` → `%7B`, `}` → `%7D`.

```json
PUT /_snapshot/totality_repo/%3Ctotality-snapshot-%7Bnow%2Fd%7D%3E?wait_for_completion=true
{
  "indices": "totality-*",
  "ignore_unavailable": true,
  "include_global_state": false
}

// list them
GET _cat/snapshots/totality_repo?v
GET /_snapshot/totality_repo/_all
GET /_snapshot/totality_repo/_current
GET /_snapshot/totality_repo/_status
```

### Restore

```json
POST /_snapshot/totality_repo/totality-snapshot-2026.08.14/_restore
{
  "indices": "totality-raw",
  "ignore_unavailable": true,
  "include_global_state": false,
  "rename_pattern": "(.+)",
  "rename_replacement": "restored_$1",
  "index_settings": {
    "index.number_of_replicas": 0
  }
}

GET _cat/indices/*totality*?v
```

:warning: **You cannot restore over an open index of the same name.** Either close it, delete it, or restore under a new name as above — renaming is usually the safer exam answer.
```json
POST totality-raw/_close    // then restore, then _open
// or
DELETE totality-raw         // then restore
```

Monitor a restore:
```json
GET _cat/recovery?v&active_only=true
GET _cluster/health?wait_for_status=green&timeout=60s
```

### Version compatibility

A snapshot restores to the same major version, or to the **next** major version — one hop only.
- 7.x index → 8.x :white_check_mark:
- 6.x index → 8.x :x: (reindex in 7.x first)
- Newer Elasticsearch → older :x:

### Cluster and security configuration

Take regular backups of the `$ES_PATH_CONF` directory (usually `/etc/elasticsearch`) with your normal file backup tooling, and capture the persistent cluster settings:
```json
GET _cluster/settings?pretty&flat_settings&filter_path=persistent
```

Security configuration lives partly in files (`xpack.security.*` in `elasticsearch.yml`, the keystore, role and role-mapping files) and partly in the `.security` index — which holds native-realm users and hashed passwords, roles created via the API, role mappings, application privileges, and API keys. Snapshot it like any other index, into a repository with strictly restricted access:
```json
PUT /_snapshot/secure_repo/security-snapshot
{
  "indices": ".security",
  "include_global_state": true,
  "feature_states": ["security"]
}
```

## 5.3 Configure a snapshot to be searchable

> Searchable snapshots let you search infrequently accessed, read-only data cost-effectively. The cold and frozen tiers use them to cut storage and operating costs. They eliminate the need for replica shards, potentially halving local storage.

Two storage modes:

| Mode | Tier | Behaviour |
| --- | --- | --- |
| `full_copy` (default) | cold | full local copy, no replicas needed |
| `shared_cache` | frozen | only a partial local cache; data fetched from the repository on demand |

Set the frozen-tier cache size in `elasticsearch.yml`:
```
xpack.searchable.snapshot.shared_cache.size=100mb
```
> Defaults to 90% of total disk space on dedicated frozen nodes, and `0b` otherwise — so you must set it if your node is not a dedicated frozen node.

:question: Mount the eclipse snapshot as a searchable index called `mounted-totality`.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
POST _snapshot/totality_repo/totality-snapshot-2026.08.14/_mount?wait_for_completion=true&storage=shared_cache
{
  "index": "totality-raw",
  "renamed_index": "mounted-totality"
}
```

Test it — it queries exactly like a normal index:
```json
GET _cat/indices/mounted-totality?v
GET mounted-totality/_count
GET mounted-totality/_search
{
  "size": 5,
  "query": { "term": { "state.keyword": "Vermont" } }
}
```

Cleanup:
```json
POST /mounted-totality/_searchable_snapshots/cache/clear
DELETE mounted-totality
```
</details>
<hr>

:bulb: ILM can do this for you with the `searchable_snapshot` action, which is legal in the hot, cold and frozen phases:
```json
"cold": {
  "min_age": "30d",
  "actions": {
    "searchable_snapshot": { "snapshot_repository": "totality_repo" }
  }
}
```

## 5.4 Configure a cluster for cross-cluster search

:bulb: Writing the *query* is objective 2.6.

### Two security models in 8.15

| Model | Notes |
| --- | --- |
| **API key based** (8.14+, recommended) | Create a cross-cluster API key on the remote, store it in the local keystore under `cluster.remote.<alias>.credentials`. Needs `remote_cluster_server.enabled: true` on the remote (port **9443**). Fine-grained access control. |
| **Certificate based** (classic) | Mutual TLS on the transport port (**9300**). The local user's *role names* are sent to the remote, which must have identically-named roles. Simpler, but a superuser on one side is a superuser on the other. |

### Two connection modes

| Mode | Setting | Use when |
| --- | --- | --- |
| `sniff` (default) | `cluster.remote.<alias>.seeds` | you can reach the remote nodes' publish addresses |
| `proxy` | `.mode: proxy` + `.proxy_address` | the remote sits behind a load balancer (this is what Elastic Cloud does) |

### Register the remotes

```json
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_one": {
          "seeds": ["10.0.0.1:9300"],
          "skip_unavailable": true
        },
        "cluster_two": {
          "seeds": ["10.0.0.2:9300"]
        }
      }
    }
  }
}
```

Proxy mode instead:
```json
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.cluster_two.mode": "proxy",
    "cluster.remote.cluster_two.proxy_address": "proxy.example.com:9400"
  }
}
```

Remove a remote by setting it to `null`:
```json
PUT _cluster/settings
{ "persistent": { "cluster.remote.cluster_one.seeds": null } }
```

### Verify

```json
GET _remote/info
GET _resolve/cluster/*:totality-*
GET _cluster/settings?filter_path=persistent.cluster.remote

// output
{
  "persistent": {
    "cluster.remote.cluster_one.seeds": ["10.0.0.1:9300"],
    "cluster.remote.cluster_one.skip_unavailable": "true"
  }
}
```

Then run a real cross-cluster count:
```json
GET cluster_one:totality-raw/_count
```

:warning: The coordinating node needs the **`remote_cluster_client`** role. On a default all-roles node this is already true, but on a cluster with dedicated node roles it is a very common cause of "remote cluster is not connected".

:bulb: `skip_unavailable: true` means a search succeeds (with `_clusters.skipped` incremented) when that cluster is down. At the 8.15 default of `false`, the whole search fails.

:bulb: This is also easy to do in Kibana: **Stack Management → Remote Clusters → Add a remote cluster**.

## 5.5 Implement cross-cluster replication

> CCR uses an active-passive model. You index into a **leader** index, and the data is replicated to one or more read-only **follower** indices. The remote cluster containing the leader must be configured first.

Uses: surviving a datacenter outage, keeping search volume off the indexing cluster, and reducing search latency by serving requests closer to users.

### Prerequisites

- A **platinum or enterprise licence**. Start a 30-day trial in a lab:
  ```json
  POST /_license/start_trial?acknowledge=true
  ```
  Or in Kibana: **Stack Management → License management → Start a 30-day trial**.
- The leader registered as a remote cluster **on the follower** (see 5.4).
- `remote_cluster_client` on the coordinating nodes of both clusters.

### The lab setup

| Cluster | Kibana | Remote name | Seed node | Role |
| --- | --- | --- | --- | --- |
| East | http://\<host\>:5601 | west-cluster | esnode-west:9300 | Leader |
| West | http://\<host\>:5602 | east-cluster | esnode-east:9300 | Follower |

:warning: Use one incognito window per Kibana. The two instances overwrite each other's session cookies and log you out constantly.

### :warning: Do this with the API, not just the GUI

The exam is done in Dev Tools. This is the section people most often practise only in the UI.

```json
// 1. on the FOLLOWER - register the leader
PUT _cluster/settings
{ "persistent": { "cluster.remote.east-cluster.seeds": ["esnode-east:9300"] } }

GET _remote/info

// 2. on the FOLLOWER - follow a single index
PUT /follower-totality-raw/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster": "east-cluster",
  "leader_index": "totality-raw"
}

// 3. check it
GET /follower-totality-raw/_ccr/info      // status: active | paused
GET /follower-totality-raw/_ccr/stats     // checkpoints, lag, read exceptions
GET /_ccr/stats                           // cluster-wide, plus auto-follow stats

GET _cat/indices/follow*?v
// green open follower-totality-raw ... 190
```

### Auto-follow pattern

Replicates any **new** index matching a pattern — the answer to "replicate our time-series indices automatically".

```json
// on the FOLLOWER
PUT /_ccr/auto_follow/east-auto-follow
{
  "remote_cluster": "east-cluster",
  "leader_index_patterns": ["totality-ts-*"],
  "leader_index_exclusion_patterns": ["totality-ts-ignore-*"],
  "follow_index_pattern": "follower-{{leader_index}}"
}

GET    /_ccr/auto_follow/east-auto-follow
DELETE /_ccr/auto_follow/east-auto-follow
```

Test it — create a matching index on the leader and add documents:
```json
// on the LEADER
PUT totality-ts-000001
{
  "settings": { "number_of_replicas": 0, "number_of_shards": 1 },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "park_name":  { "type": "keyword" }
    }
  }
}

POST totality-ts-000001/_doc
{ "@timestamp": "2024-04-08T15:29:32.000Z", "park_name": "Ahern State Park" }

// on the FOLLOWER, a moment later
GET follower-totality-ts-000001/_count
```

### Follower lifecycle

```json
POST /follower-totality-raw/_ccr/pause_follow
POST /follower-totality-raw/_ccr/resume_follow
{ "max_read_request_operation_count": 5120 }

// converting a follower into a normal writeable index is a three step dance
POST /follower-totality-raw/_ccr/pause_follow
POST /follower-totality-raw/_close
POST /follower-totality-raw/_ccr/unfollow
POST /follower-totality-raw/_open
```

:warning: Things that catch people out:
- A follower index is **read-only**. Writes must go to the leader.
- `follow_index_pattern` uses `{{leader_index}}` — double braces, mustache style.
- The index status shows **Paused** during remote recovery, then flips to **Active** once following begins. That is normal, not a fault.

## 5.6 Automate snapshots with Snapshot Lifecycle Management

:sparkles: **New objective since 8.1** — make sure you can do this cold.

> SLM automatically takes snapshots on a schedule and deletes them when they age out.

### Prerequisites
1. A **registered repository** — SLM cannot create one for you.
2. The `manage_slm` cluster privilege plus `manage` on the indices being snapshotted.

### Policy anatomy

| Field | Notes |
| --- | --- |
| `schedule` | **Cron with a seconds field**: `<sec> <min> <hour> <day-of-month> <month> <day-of-week> [year]`. 8.15 also accepts a plain interval like `"1d"`. |
| `name` | Date math, **not** URI-encoded here because it is in the body: `<nightly-snap-{now/d}>`. A UUID is appended so names never clash. |
| `repository` | must already exist |
| `config` | the same body you would send to the snapshot API: `indices`, `include_global_state`, `feature_states`, `partial` |
| `retention` | `expire_after`, `min_count`, `max_count` |

:question: Create an SLM policy `totality-nightly` that snapshots `totality-*` to `totality_repo` every day at 02:00 UTC, keeping snapshots for 14 days, never fewer than 3 and never more than 20.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT _slm/policy/totality-nightly
{
  "schedule": "0 0 2 * * ?",
  "name": "<totality-nightly-{now/d}>",
  "repository": "totality_repo",
  "config": {
    "indices": ["totality-*"],
    "include_global_state": false
  },
  "retention": {
    "expire_after": "14d",
    "min_count": 3,
    "max_count": 20
  }
}
```

Do not wait until 02:00 — run it now:
```json
POST _slm/policy/totality-nightly/_execute

// output
{ "snapshot_name": "totality-nightly-2026.08.14-abc123def456ghi789jkl" }
```

Verify:
```json
GET _slm/policy/totality-nightly

// look for:
//   "next_execution"             - when it fires next
//   "last_success.snapshot_name"
//   "last_failure"
//   "stats.snapshots_taken"

GET _cat/snapshots/totality_repo?v
GET _slm/stats
```
</details>
<hr>

### Cron quick reference

The seconds field first is what trips people up.

| Expression | Fires |
| --- | --- |
| `0 30 1 * * ?` | daily at 01:30:00 |
| `0 0 2 * * ?` | daily at 02:00:00 |
| `0 0 */6 * * ?` | every 6 hours on the hour |
| `0 15 2 ? * MON` | Mondays at 02:15 |
| `0 0 0 1 * ?` | first of the month at midnight |
| `0 0/30 * * * ?` | every 30 minutes |

:bulb: `?` means "no specific value" and is required in **exactly one** of day-of-month / day-of-week. You cannot put `*` in both.

### The rest of the API

```json
GET    _slm/policy
GET    _slm/policy/totality-nightly
DELETE _slm/policy/totality-nightly

POST _slm/policy/totality-nightly/_execute
POST _slm/_execute_retention

GET  _slm/stats
GET  _slm/status
POST _slm/stop
POST _slm/start
```

:warning: Traps:
- `retention` is applied by a separate periodic task (`slm.retention_schedule`, default 01:30 UTC daily), **not** at snapshot time.
- Deleting a policy does **not** delete the snapshots it already took.
- `last_failure` showing `repository_missing_exception` means you created the policy before the repository.
- SLM policies are cluster state, so a snapshot with `include_global_state: true` backs them up too.
- `GET _health_report/slm` is the fastest answer to "SLM is not working, fix it".

---
---

# Bonus — no longer on the objective list

These were objectives on older versions of the exam. Still useful; do not spend your last revision hours on them.

## (Bonus) Highlight the search terms in the response of a query

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/highlighting.html

:question: Search for parks in New Hampshire and highlight the matched terms in the `name` field, wrapping them in `#aaa#` and `#bbb#`.

<details>
  <summary>View Solution (click to reveal)</summary>

:warning: You must highlight a field that the **query actually matched on**. Highlighting `name` while querying `state` produces an empty `highlight` block — a mistake the earlier version of this file made.

```json
GET totality-raw/_search
{
  "size": 3,
  "query": {
    "match": { "name": "State Park" }
  },
  "highlight": {
    "fields": {
      "name": {
        "pre_tags": "#aaa#",
        "post_tags": "#bbb#"
      }
    }
  }
}
```

Highlight across two fields, with a query that touches both:
```json
GET totality-raw/_search
{
  "size": 3,
  "query": {
    "multi_match": {
      "query": "Laconia",
      "fields": ["city", "name"]
    }
  },
  "highlight": {
    "pre_tags": ["<em>"],
    "post_tags": ["</em>"],
    "fields": {
      "city": {},
      "name": {}
    }
  }
}
```
</details>
<hr>

## (Bonus) Define and use a search template

A search template is a stored search you can run with different variables — a bit like a SQL user-defined function, though with no per-template security; read access to the underlying index is all that is required.

:question: Create a search template that returns the parks in a given state at a given coverage, then use it to find the Vermont parks in totality.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
POST _scripts/get_by_state
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "bool": {
          "must": [
            { "term": { "coverage.keyword": "{{coverage}}" } },
            { "term": { "state.keyword":    "{{state}}"    } }
          ]
        }
      }
    }
  }
}

// output
{ "acknowledged": true }
```

Pull it back — note it is not formatted nicely :rage:
```json
GET _scripts/get_by_state

// output
{
  "_id": "get_by_state",
  "found": true,
  "script": {
    "lang": "mustache",
    "source": """{"query":{"bool":{"must":[{"term":{"coverage.keyword":"{{coverage}}"}},{"term":{"state.keyword":"{{state}}"}}]}}}""",
    "options": { "content_type": "application/json;charset=utf-8" }
  }
}
```

Run it:
```json
GET totality-raw/_search/template?filter_path=hits.total.value
{
  "id": "get_by_state",
  "params": {
    "coverage": "100%",
    "state": "Vermont"
  }
}

// Answer: 29
```

Render it without running it — the fastest way to debug a template:
```json
POST _render/template
{
  "id": "get_by_state",
  "params": { "coverage": "100%", "state": "Vermont" }
}
```

**Bonus templates to try:** count parks in totality for a given zip code; parks in a given city; parks above a parameterised minimum totality.
</details>
<hr>

## (Bonus) Configure an index so that it properly maintains the relationships of nested arrays of objects

:question: 1. Create an index that lets relationships between parks be queried correctly.

```json
POST totality_r/_bulk?refresh
{"index":{"_id":"0"}}
{"name":"X State Park","relationship":[{"camping":"Yes","neighbor":"Y State Park"}]}
{"index":{"_id":"1"}}
{"name":"Y State Park","relationship":[{"camping":"No","neighbor":"Z State Park"}]}
{"index":{"_id":"2"}}
{"name":"Z State Park","relationship":[{"camping":"Yes","neighbor":"A State Park"}]}
{"index":{"_id":"3"}}
{"name":"A State Park","relationship":[{"camping":"Yes","neighbor":"X State Park"}]}
{"index":{"_id":"4"}}
{"name":"B State Park","relationship":[{"camping":"No","neighbor":"X State Park"}]}
{"index":{"_id":"5"}}
{"name":"C State Park","relationship":[{"camping":"No","neighbor":"Y State Park"}]}
{"index":{"_id":"6"}}
{"name":"D State Park","relationship":[{"camping":"Yes","neighbor":"Z State Park"},{"camping":"No","neighbor":"A State Park"}]}
```

:warning: The bulk syntax is `POST <index>/_bulk`, with `{"index":{"_id":"N"}}` action lines. `PUT totality_r/_doc/_bulk` — as an earlier version of this file had — is not valid.

<details>
  <summary>View Solution (click to reveal)</summary>

Create the mapping **before** loading the data. The important part is `"type": "nested"`, so the array of objects is not flattened on ingest.

```json
PUT totality_r
{
  "mappings": {
    "properties": {
      "name": { "type": "keyword" },
      "relationship": {
        "type": "nested",
        "properties": {
          "camping":  { "type": "keyword" },
          "neighbor": { "type": "keyword" }
        }
      }
    }
  }
}
```

**Why this matters:** without `nested`, Elasticsearch flattens `relationship` into two parallel arrays — `relationship.camping: ["Yes","No"]` and `relationship.neighbor: ["Z State Park","A State Park"]` — and loses which value went with which. Document 6 would then wrongly match a search for "camping=No AND neighbor=Z State Park". `nested` indexes each object as its own hidden document, preserving the pairing.
</details>
<hr>

:question: 2. Query all parks that have camping set to `Yes`.

<details>
  <summary>View Solution (click to reveal)</summary>

You need a `nested` query with the matching `path`, or it will not work:

```json
GET totality_r/_search?filter_path=hits.total.value,hits.hits._source.name
{
  "query": {
    "nested": {
      "path": "relationship",
      "query": {
        "term": { "relationship.camping": "Yes" }
      }
    }
  }
}

// X, Z, A, D State Park
```
</details>
<hr>

:question: 3. Show parks where camping is `Yes` **and** the neighbor is `Y State Park` — in the same relationship object.

<details>
  <summary>View Solution (click to reveal)</summary>

This is the whole point of `nested`. Both conditions must hold **within one object**, not merely somewhere in the array:

```json
GET totality_r/_search?filter_path=hits.total.value,hits.hits._source.name
{
  "query": {
    "nested": {
      "path": "relationship",
      "query": {
        "bool": {
          "must": [
            { "term": { "relationship.camping":  "Yes" } },
            { "term": { "relationship.neighbor": "Y State Park" } }
          ]
        }
      }
    }
  }
}

// Answer: 1 - X State Park
```

Document 6 has both `camping: Yes` and a `neighbor: A State Park` object, plus a `camping: No` with `neighbor: Z State Park` — but no single object with both `Yes` and `Y State Park`, so it correctly does not match. Remove `"type": "nested"` from the mapping and re-run this to watch it return the wrong answer.

:bulb: Add `"inner_hits": {}` to the nested query to see *which* nested object matched.
</details>
<hr>

## (Bonus) Role-based access control

See the bonus section of [Cluster_Management.md](Cluster_Management.md) for roles, users, and field/document level security.
