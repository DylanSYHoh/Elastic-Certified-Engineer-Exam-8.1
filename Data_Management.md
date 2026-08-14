# Data Management (8.15)

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/data-management.html

**8.15 objectives covered here:**
1. Define an index that satisfies a given set of requirements
2. Define and use a dynamic template that satisfies a given set of requirements
3. Define an Index Lifecycle Management policy for a time-series index
4. Define an index template that creates a new data stream

> :books: "Define and use an index template for a given pattern" is **no longer a listed objective**, but it is a hard prerequisite for the data stream objective, so it is kept below as background.

---

# Define an index that satisfies a given set of requirements

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/indices-create-index.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/index-modules.html (the full settings list)

```json
PUT accounts-raw
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  }
}
```

Most "define an index" questions bundle settings **and** mappings. The settings that actually turn up:

| Setting | Notes |
| --- | --- |
| `number_of_shards` | static — cannot be changed after creation |
| `number_of_replicas` | dynamic — `PUT idx/_settings` any time |
| `refresh_interval` | dynamic, e.g. `"30s"` or `-1` to disable |
| `index.lifecycle.name` | attach an ILM policy |
| `index.lifecycle.rollover_alias` | required for ILM rollover on an **alias** (not needed for data streams) |
| `index.default_pipeline` / `index.final_pipeline` | attach an ingest pipeline to every write |
| `index.max_result_window` | default `10000`; raise it (carefully) for deep `from`/`size` paging |
| `index.blocks.read_only` / `index.blocks.write` | for the "make this index read only" flavour of question |
| `index.mapping.total_fields.limit` | default `1000` |
| `index.routing.allocation.include._tier_preference` | data tier placement |

:question: Create an index `accounts-raw` with 1 primary shard, no replicas, a 30s refresh interval, and a mapping where `email` is a `keyword` and `address` is full-text searchable.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT accounts-raw
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "refresh_interval": "30s"
  },
  "mappings": {
    "properties": {
      "email":   { "type": "keyword" },
      "address": { "type": "text" }
    }
  }
}
```

Verify:
```json
GET accounts-raw/_settings
GET accounts-raw/_mapping
```
</details>
<hr>

:bulb: **Changing settings later** — static settings (`number_of_shards`, most `analysis.*`) require a close/reopen or a reindex; dynamic ones do not:

```json
PUT accounts-raw/_settings
{
  "number_of_replicas": 0,
  "refresh_interval": "1s"
}
```

Adding an analyzer to an existing index needs the index closed first:
```json
POST accounts-raw/_close
PUT  accounts-raw/_settings
{ "analysis": { "analyzer": { "my_analyzer": { "type": "custom", "tokenizer": "standard" } } } }
POST accounts-raw/_open
```

---

# Background: index templates (prerequisite, not a listed 8.15 objective)

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/index-templates.html

> :warning: **Legacy `PUT _template/<name>` is deprecated.** Older versions of these notes used it. On 8.x always use **composable** templates: `PUT _index_template/<name>` and `PUT _component_template/<name>`. Composable templates take precedence over legacy ones, and legacy templates are removed in 9.x.

```json
PUT _index_template/accounts-tmpl
{
  "index_patterns": ["accounts-*"],
  "priority": 200,
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0
    },
    "mappings": {
      "properties": {
        "account_number": { "type": "integer" },
        "balance":        { "type": "integer" },
        "firstname":      { "type": "keyword" },
        "lastname":       { "type": "keyword" },
        "age":            { "type": "integer" },
        "gender":         { "type": "keyword" },
        "address":        { "type": "text" },
        "employer":       { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
        "email":          { "type": "keyword" },
        "city":           { "type": "keyword" },
        "state":          { "type": "keyword" }
      }
    }
  }
}
```

:warning: Do **not** set `"_source": { "enabled": false }` in a template you intend to reindex from later — `_reindex`, `_update_by_query` and `_source` filtering all need it.

## Test a template **without** creating anything

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/indices-simulate-index.html

This is the single most useful template-debugging API in the exam — it shows exactly what settings/mappings a new index called `accounts-new` would get, including everything merged in from component templates:

```json
POST _index_template/_simulate_index/accounts-new
```

Simulate a template body before you even save it:
```json
POST _index_template/_simulate
{
  "index_patterns": ["accounts-*"],
  "template": { "settings": { "number_of_shards": 3 } }
}
```

Check which templates exist and which one wins:
```json
GET _index_template/accounts-tmpl
GET _cat/templates?v
```

:bulb: **Priority rules** — when several composable templates match an index pattern, the one with the highest `priority` wins outright (they are *not* merged). Component templates listed in `composed_of` *are* merged, in order, with later ones overriding earlier ones, and the inline `template` block overriding all of them.

Verify a created index behaves:
```json
POST accounts-new/_search?filter_path=hits.total.value
{
  "query": { "match": { "employer": "Zork" } }
}

// Output
{ "hits": { "total": { "value": 1 } } }
```

---

# Define and use a dynamic template that satisfies a given set of requirements

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/dynamic-templates.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/dynamic-mapping.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/dynamic-field-mapping.html

Dynamic templates let you control how **fields you did not explicitly map** get mapped when they first appear in a document.

## The matching options

| Key | Matches on |
| --- | --- |
| `match_mapping_type` | the JSON type Elasticsearch detected: `string`, `long`, `double`, `boolean`, `date`, `object`, or `*` |
| `match` / `unmatch` | the **field name**, glob patterns (`ip_*`, `*_count`) |
| `match_pattern: "regex"` | switches `match` to regex instead of glob |
| `path_match` / `path_unmatch` | the **full dotted path** (`user.address.*`) |

Inside `mapping` you can use two placeholders: `{name}` (the field name) and `{dynamic_type}` (the detected type).

:warning: **Order matters.** `dynamic_templates` is an *array*, and the first template that matches wins. Put your most specific rules first.

<hr>

:question: 1. Create an index `logs-dyn` where **every** string field is mapped as `keyword` only (no `text`, no `.keyword` sub-field), to save space.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT logs-dyn
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_keywords": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "keyword"
          }
        }
      }
    ]
  }
}
```

Test it — index a doc with a field you never declared, then look at the resulting mapping:
```json
POST logs-dyn/_doc
{ "hostname": "web-01", "message": "connection refused" }

GET logs-dyn/_mapping
```
Both fields should be `"type": "keyword"`.
</details>
<hr>

:question: 2. Create an index `logs-dyn2` where:
- any field whose name ends in `_ip` is mapped as `ip`
- any field whose name starts with `num_` is mapped as `long`
- every other string becomes `text` **with** a `raw` keyword sub-field
- any detected `date` keeps the format `yyyy-MM-dd HH:mm:ss||epoch_millis`

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Build this one rule at a time and re-index a test document after each — this is exactly how the exam question decomposes.

```json
PUT logs-dyn2
{
  "mappings": {
    "dynamic_templates": [
      {
        "ip_fields": {
          "match": "*_ip",
          "mapping": { "type": "ip" }
        }
      },
      {
        "numeric_prefixed": {
          "match": "num_*",
          "mapping": { "type": "long" }
        }
      },
      {
        "dates_with_format": {
          "match_mapping_type": "date",
          "mapping": {
            "type": "date",
            "format": "yyyy-MM-dd HH:mm:ss||epoch_millis"
          }
        }
      },
      {
        "strings_text_and_raw": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "text",
            "fields": {
              "raw": { "type": "keyword", "ignore_above": 256 }
            }
          }
        }
      }
    ]
  }
}
```

Test:
```json
POST logs-dyn2/_doc
{
  "client_ip": "192.168.1.10",
  "num_bytes": 4096,
  "message": "GET /index.html",
  "event_time": "2024-04-08 15:29:32"
}

GET logs-dyn2/_mapping
```
</details>
<hr>

:question: 3. Map everything under the object path `user.metrics.*` as `float`, but leave `user.metrics.label` as a keyword.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT metrics-dyn
{
  "mappings": {
    "dynamic_templates": [
      {
        "metrics_label": {
          "path_match": "user.metrics.label",
          "mapping": { "type": "keyword" }
        }
      },
      {
        "metrics_as_float": {
          "path_match": "user.metrics.*",
          "mapping": { "type": "float" }
        }
      }
    ]
  }
}
```
The more specific `path_match` must come **first** — otherwise `user.metrics.label` is swallowed by the wildcard rule and mapped as `float`, which will then reject the string value.
</details>
<hr>

:question: 4. Create an index that maps unknown string fields as `keyword`, but **rejects** any document containing a field that is not already in the mapping.

<details>
  <summary>View Solution (click to reveal)</summary>

That is the `dynamic` parameter, not a dynamic template. Know all four values:

| `dynamic` | Behaviour |
| --- | --- |
| `true` (default) | new fields are added to the mapping and indexed |
| `runtime` | new fields are added as **runtime** fields — queryable, not indexed |
| `false` | new fields are ignored, not indexed, but still stored in `_source` |
| `strict` | indexing a document with an unknown field **throws an exception** |

```json
PUT strict-idx
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "known_field": { "type": "keyword" }
    }
  }
}

POST strict-idx/_doc
{ "known_field": "a", "surprise": "b" }
// -> strict_dynamic_mapping_exception
```

:bulb: `"dynamic": "runtime"` is worth remembering — it links the Data Management and Data Processing sections of the exam together.
</details>
<hr>

## Dynamic templates in a component template

Combining the two is a very likely exam shape — "define a template for `logs-*` where all new string fields become keywords":

```json
PUT _component_template/dyn-mappings
{
  "template": {
    "mappings": {
      "dynamic_templates": [
        {
          "strings_as_keywords": {
            "match_mapping_type": "string",
            "mapping": { "type": "keyword" }
          }
        }
      ]
    }
  }
}

PUT _index_template/logs-tmpl
{
  "index_patterns": ["logs-*"],
  "composed_of": ["dyn-mappings"],
  "priority": 200
}

// prove it, without creating anything:
POST _index_template/_simulate_index/logs-2026.08.14
```

---

# Define an Index Lifecycle Management policy for a time-series index

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/index-lifecycle-management.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/set-up-lifecycle-policy.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ilm-actions.html

## The phases, in order

`hot` → `warm` → `cold` → `frozen` → `delete`

## Which actions are legal in which phase (memorise this table)

| Action | hot | warm | cold | frozen | delete |
| --- | :---: | :---: | :---: | :---: | :---: |
| `set_priority` | ✔ | ✔ | ✔ | | |
| `unfollow` | ✔ | ✔ | ✔ | ✔ | |
| `rollover` | ✔ | | | | |
| `readonly` | ✔ | ✔ | ✔ | | |
| `shrink` | ✔ | ✔ | | | |
| `forcemerge` | ✔ | ✔ | | | |
| `downsample` | ✔ | ✔ | ✔ | | |
| `searchable_snapshot` | ✔ | | ✔ | ✔ | |
| `allocate` | | ✔ | ✔ | | |
| `migrate` | | ✔ | ✔ | | |
| `delete` | | | | | ✔ |

:warning: `rollover` is **hot only**. `searchable_snapshot` cannot be in `warm`. `delete` is the only action in the delete phase.

## Rollover conditions (8.15)

At least one `max_*` condition is required. `min_*` conditions must *all* be satisfied before any `max_*` can trigger.

| `max_*` | `min_*` |
| --- | --- |
| `max_age` | `min_age` |
| `max_docs` | `min_docs` |
| `max_size` | `min_size` |
| `max_primary_shard_size` | `min_primary_shard_size` |
| `max_primary_shard_docs` | `min_primary_shard_docs` |

:bulb: Elastic recommends `max_primary_shard_size` (target ~50gb) over the deprecated-in-spirit `max_size`, because it is independent of shard count.

## Example: Rich Raposa's exam-video style policy

Requirements: the corresponding index template is `task3`; data is hot for 3 minutes then rolls to warm; warm for 5 minutes then to cold; deleted 10 minutes after rollover.

:warning: `min_age` in a phase is measured **from the rollover time** of the index (not from the previous phase), so the numbers below are cumulative from rollover.

```json
PUT _ilm/policy/task3
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
        "actions": { "delete": { "delete_searchable_snapshot": true } }
      }
    }
  }
}
```

## Example: a fuller time-series policy

:question: Create `timeseries_policy` that rolls over at 50GB primary shard size **or** 30 days, force-merges to 1 segment and makes the index read-only in warm after 7 days, moves to a searchable snapshot in cold at 30 days, and deletes at 90 days.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT _ilm/policy/timeseries_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "30d"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": { "max_num_segments": 1 },
          "readonly": {},
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "searchable_snapshot": { "snapshot_repository": "my_test_backup" },
          "set_priority": { "priority": 0 }
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
`searchable_snapshot` requires a **registered repository** — see [Cluster_Management.md](Cluster_Management.md).
</details>
<hr>

## Attaching the policy — alias style (classic index + rollover alias)

```json
PUT _index_template/timeseries_template
{
  "index_patterns": ["timeseries-*"],
  "priority": 200,
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "index.lifecycle.name": "timeseries_policy",
      "index.lifecycle.rollover_alias": "timeseries"
    }
  }
}
```

### Bootstrap the initial index
:bulb: Do this **before** ingesting. Non-data-stream rollover will not work without the write alias.

```json
PUT timeseries-000001
{
  "aliases": {
    "timeseries": { "is_write_index": true }
  }
}
```

### Write to the alias, not the index
```json
POST timeseries/_doc
{
  "message": "logged the request",
  "@timestamp": "2026-08-14T10:30:00.000Z"
}
```

### Force a rollover (useful for testing without waiting for the ILM poll)
```json
POST timeseries/_rollover
```

### Check which index the alias now writes to
```json
GET _alias/timeseries

// output
{
  "timeseries-000002": { "aliases": { "timeseries": { "is_write_index": true  } } },
  "timeseries-000001": { "aliases": { "timeseries": { "is_write_index": false } } }
}
```

## Debugging ILM — the four commands you will actually need

```json
GET timeseries-*/_ilm/explain              // where is each index in the lifecycle, and why is it stuck?
GET _ilm/policy/timeseries_policy          // read the policy back
GET _ilm/status                            // is ILM even RUNNING?
POST _ilm/start                            // ...if it was STOPPED
```

Trimmed view:
```json
GET test-index/_ilm/explain?filter_path=*.*.age,*.*.phase
```

:bulb: **ILM runs on a poll interval, by default `10m`.** Nothing you do with minute-scale `min_age` values will look instant. In a lab you can shorten it:
```json
PUT _cluster/settings
{ "persistent": { "indices.lifecycle.poll_interval": "10s" } }
```
In production use days/hours, not minutes. Every phase transition is timed from the **rollover** of that index, and no time is exact.

```json
// example output after letting it run
{
  "indices": {
    "test-index-000003": { "age": "39.65m", "phase": "delete" },
    "test-index-000004": { "age": "29.64m", "phase": "cold"   },
    "test-index-000005": { "age": "19.65m", "phase": "warm"   },
    "test-index-000006": { "age": "9.65m",  "phase": "hot"    }
  }
}
```

If an index is stuck, `_ilm/explain` shows `step_info` with the error; fix the cause then:
```json
POST timeseries-000001/_ilm/retry
```

---

# Define an index template that creates a new data stream

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/data-streams.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/set-up-a-data-stream.html

## What is a data stream

> A data stream lets you store append-only time series data across multiple indices while giving you a single named resource for requests. _Data streams are well-suited for logs, events, metrics, and other continuously generated data._

:warning: __Use for data that doesn't need updating__ — so __not__ for alarms that open/close or support tickets. Use an index + alias for that.

> ### Data stream write index
>
> The most recently created backing index is the data stream's write index. The stream adds new documents to this index _only_.
>
> You cannot add new documents to other backing indices, even by sending requests directly to the index.
>
> You also cannot perform operations on a write index that may hinder indexing, such as: Clone, Delete, Freeze, Shrink, Split.

> ### Data streams are append-only (mainly)
> You cannot send update or deletion requests for existing documents directly to a data stream. Instead, use the update by query and delete by query APIs, or send the request directly to the document's backing index.

### Hard requirements — these are what the exam marks
1. The index template **must** contain a `"data_stream": { }` block (empty object is fine).
2. The mapping **must** have an `@timestamp` field of type `date` or `date_nanos`. (If you omit it entirely, Elasticsearch adds it for you — but declare it explicitly.)
3. The template `priority` should be set (built-in Elastic templates use 100–200; use a higher number to win).
4. Backing indices are named `.ds-<name>-<yyyy.MM.dd>-<NNNNNN>` and are hidden.
5. **No `rollover_alias` setting** — data streams roll over on their own name. Setting `index.lifecycle.rollover_alias` on a data stream template is a common wrong answer.

## 1. Create an ILM policy

Nothing data-stream-specific here:

```json
PUT _ilm/policy/ds_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "30d"
          }
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

## 2. Create the component templates

### Mappings

```json
PUT _component_template/my-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date",
          "format": "date_optional_time||epoch_millis"
        },
        "message": {
          "type": "wildcard"
        }
      }
    }
  },
  "_meta": {
    "description": "Mappings for @timestamp and message fields"
  }
}
```

### Settings

:warning: The policy name must match exactly what you created above — `ds_policy`, not `ds-policy`. A typo here silently gives you a data stream with no lifecycle, and `_ilm/explain` will return nothing.

```json
PUT _component_template/my-settings
{
  "template": {
    "settings": {
      "index.lifecycle.name": "ds_policy",
      "number_of_shards": 1,
      "number_of_replicas": 0
    }
  },
  "_meta": {
    "description": "Settings for ILM"
  }
}
```

## 3. Create the combined index template

:bulb: `"data_stream": { }` is what makes this a data stream template.

```json
PUT _index_template/my-component-index-template
{
  "index_patterns": ["my-data-stream*"],
  "data_stream": { },
  "composed_of": ["my-mappings", "my-settings"],
  "priority": 500,
  "_meta": {
    "description": "Template for my time series data"
  }
}
```

## 4. Create the data stream

Either implicitly, by indexing a document:

```json
POST my-data-stream/_doc
{
  "@timestamp": "2099-05-06T16:21:15.000Z",
  "message": "192.0.2.42 - - [06/May/2099:16:21:15 +0000] \"GET /images/bg.jpg HTTP/1.0\" 200 24736"
}

// output
{
  "_index": ".ds-my-data-stream-2026.08.14-000001",
  "_id": "fMjrTnoBw1F50El3ljBq",
  "result": "created",
  ...
}
```

:bulb: There is no bootstrap index to create, unlike the alias approach above.

Or explicitly:
```json
PUT _data_stream/my-data-stream
```

:warning: You must use `op_type=create` (or `_bulk` `create`, or plain `POST .../_doc`) when writing to a data stream. `PUT my-data-stream/_doc/1` — indexing with an explicit ID — is **rejected**.

## 5. Inspect and manage it

```json
GET _data_stream/my-data-stream            // generation, backing indices, ILM policy, timestamp field
GET _data_stream/my-data-stream/_stats
GET _cat/indices/.ds-my-data-stream-*?v&expand_wildcards=all

POST my-data-stream/_rollover              // manual rollover
DELETE _data_stream/my-data-stream         // deletes the stream AND its backing indices
```

Update the mapping or settings of a data stream — this applies to **future** backing indices; existing ones keep what they had:
```json
PUT _index_template/my-component-index-template   // edit the template
POST my-data-stream/_rollover                     // then roll over to pick it up
```

Adding a *new* field is different: `PUT _data_stream/.../_mapping` does not exist, you update the **template**, but you can add a compatible field to all current backing indices with:
```json
PUT my-data-stream/_mapping
{ "properties": { "new_field": { "type": "keyword" } } }
```

## 6. Secure a data stream

Usual privileges apply; grant index privileges on the data stream name, not on `.ds-*`.
