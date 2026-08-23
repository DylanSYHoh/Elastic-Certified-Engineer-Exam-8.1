# Developing Search Applications (8.15)

**8.15 objectives covered here:**
1. Sort the results of a query by a given set of requirements
2. Implement pagination of the results of a search query
3. Define and use index aliases

> :books: The 8.1-era objectives "Highlight the search terms in the response of a query" and "Define and use a search template" are **no longer listed for 8.15**. They are kept at the bottom of this file as bonus material.

## :floppy_disk: Prerequisite data

This file uses the **bank accounts** dataset (`accounts.json`, 1000 docs) and the **Shakespeare** dataset. Load them before starting, or every exercise below returns zero hits:

```bash
curl -k -u "elastic:Password01" -H "Content-Type: application/x-ndjson" \
  -XPOST "localhost:9200/accounts-raw/_bulk?refresh" --data-binary "@accounts.json"
```
```json
GET accounts-raw/_count     // expect 1000
```

:warning: `accounts.json` action lines are `{"index":{"_id":"1"}}` with **no `_index`**, so the index name must go in the URL. Let `accounts-raw` be **dynamically mapped** — do not create the `accounts-*` index template from [Data_Management.md](Data_Management.md) first, or `gender` becomes a plain `keyword` and every `gender.keyword` query here silently returns nothing.

See the [Datasets section of the README](README.md) for all three datasets and how to load them.

---

# Sort the results of a query by a given set of requirements

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/sort-search-results.html

## What you can sort on

| You can sort on | You cannot sort on |
| --- | --- |
| `keyword`, numeric, date, boolean, `ip` | analysed `text` fields (no doc values) — use the `.keyword` sub-field |
| `_score`, `_doc` | fields with `"doc_values": false` |
| runtime fields | |
| nested fields (with a `nested` sort clause) | |

:warning: Sorting on a `text` field gives you
`Fielddata is disabled on text fields by default` — the fix in the exam is almost always "use `field.keyword`", not "enable fielddata".

<hr>

:question: Return all of `Othello`'s lines in reverse order.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET shakespeare/_search
{
  "query": {
    "term": { "speaker": "OTHELLO" }
  },
  "sort": [
    { "speech_number": { "order": "desc" } }
  ]
}
```
</details>
<hr>

:question: Sort the `shakespeare` index by `play_name` ascending, then by `speech_number` descending, and finally by relevance score.

<details>
  <summary>View Solution (click to reveal)</summary>

Sort keys are applied in array order — first key wins, later keys break ties.

```json
GET shakespeare/_search
{
  "query": { "match": { "text_entry": "love" } },
  "sort": [
    { "play_name":     { "order": "asc"  } },
    { "speech_number": { "order": "desc" } },
    { "_score":        { "order": "desc" } }
  ]
}
```

:bulb: As soon as you add any `sort`, `_score` is no longer computed (you get `null`) unless you explicitly sort on `_score` or set `"track_scores": true`.
</details>
<hr>

## Options that turn up in "given a set of requirements" questions

```json
GET my-index/_search
{
  "track_total_hits": true,          // exact hit count beyond the 10,000 default cap
  "track_scores": true,              // keep _score even when sorting on a field
  "sort": [
    {
      "price": {
        "order": "asc",
        "missing": "_last",          // "_first", "_last", or a literal value like 0
        "unmapped_type": "long",     // don't fail if an index in the pattern lacks the field
        "mode": "min"                // for array/multi-value fields: min|max|sum|avg|median
      }
    }
  ]
}
```

### Sort by distance from a point (geo)
```json
GET kibana_sample_data_ecommerce/_search
{
  "sort": [
    {
      "_geo_distance": {
        "geoip.location": { "lat": 40.71, "lon": -74.00 },
        "order": "asc",
        "unit": "km"
      }
    }
  ]
}
```

### Sort by a script (or, better in 8.x, a runtime field)
```json
GET shakespeare/_search
{
  "runtime_mappings": {
    "line_length": {
      "type": "long",
      "script": { "source": "emit(doc['text_entry.keyword'].value.length())" }
    }
  },
  "sort": [ { "line_length": { "order": "desc" } } ],
  "fields": ["line_length", "text_entry"],
  "_source": false
}
```

### Sort on a nested field
```json
GET my-index/_search
{
  "sort": [
    {
      "offers.price": {
        "order": "asc",
        "mode": "min",
        "nested": { "path": "offers", "filter": { "term": { "offers.color": "blue" } } }
      }
    }
  ]
}
```

---

# Implement pagination of the results of a search query

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/paginate-search-results.html

There are **three** pagination mechanisms in 8.15 and the exam may ask for a specific one. Know when each is correct:

| Method | Use when | Limit |
| --- | --- | --- |
| `from` + `size` | shallow paging, a UI with page numbers | `from + size` must be ≤ `index.max_result_window` (default **10000**) |
| `search_after` + PIT | deep paging, "scroll through all results" | none, but requires a tiebreaker sort |
| `scroll` | legacy bulk export | **deprecated** — the docs tell you to use `search_after` with a PIT instead |

## 1. `from` / `size`

:question: Paginate the `Othello` play, `20` speech lines per page, starting from line `40`. What is the first line on this page?

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET shakespeare/_search
{
  "size": 20,
  "from": 40,
  "query": {
    "term": { "play_name": "Othello" }
  },
  "sort": [
    { "speech_number": { "order": "asc" } }
  ]
}

// Output
      {
        "_source" : {
          "text_entry" : "Will you think so?"
        }
      }
```

:bulb: Page N (1-based) with page size S is `"from": (N-1) * S, "size": S`. Always add a `sort` — without a deterministic sort, paging results are not stable between requests.
</details>
<hr>

:warning: **The 10,000 wall.** `"from": 10000` throws
`Result window is too large, from + size must be less than or equal to: [10000]`.

Two answers, and the exam question wording tells you which one it wants:
```json
// (a) "raise the limit on this index"  - blunt instrument, costs heap
PUT shakespeare/_settings
{ "index.max_result_window": 50000 }

// (b) "page beyond 10,000 results"     - the correct answer, use search_after
```

## 2. `search_after` with a point in time (PIT)

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/point-in-time-api.html

This is the modern deep-paging answer and a very plausible exam task. Four steps:

### Step 1 — open a PIT

The PIT freezes the set of segments so results do not shift while you page.

```json
POST /shakespeare/_pit?keep_alive=5m

// output
{ "id": "46ToAwMDaWR5BXV1aWQyKwZub2RlXzMAAAAAAAAAACoBYwADaWR4..." }
```

### Step 2 — first page

:warning: When you use a PIT you put the `id` in the **body** and you do **not** name the index in the URL.

:bulb: Add a **tiebreaker** so the sort is unique — `_shard_doc` is the cheap built-in one, and it is only available with a PIT.

```json
GET /_search
{
  "size": 20,
  "query": { "term": { "play_name": "Othello" } },
  "pit": {
    "id": "46ToAwMDaWR5BXV1aWQyKwZub2RlXzMAAAAAAAAAACoBYwADaWR4...",
    "keep_alive": "5m"
  },
  "sort": [
    { "speech_number": "asc" },
    { "_shard_doc": "asc" }
  ]
}
```

### Step 3 — subsequent pages

Take the `sort` array from the **last hit** of the previous page and feed it back in as `search_after`. Also reuse the `id` returned in the response (it can change).

```json
GET /_search
{
  "size": 20,
  "query": { "term": { "play_name": "Othello" } },
  "pit": { "id": "46ToAwMDaWR5...", "keep_alive": "5m" },
  "sort": [
    { "speech_number": "asc" },
    { "_shard_doc": "asc" }
  ],
  "search_after": [ 12, 4294967298 ]
}
```

Repeat until a page comes back with fewer hits than `size`.

### Step 4 — close the PIT

```json
DELETE /_pit
{
  "id": "46ToAwMDaWR5..."
}
```

<hr>

:bulb: `search_after` works **without** a PIT too — you just lose the consistency guarantee and cannot use `_shard_doc`, so you must supply your own unique tiebreaker:

```json
GET shakespeare/_search
{
  "size": 20,
  "query": { "term": { "play_name": "Othello" } },
  "sort": [ { "speech_number": "asc" }, { "line_id": "asc" } ],
  "search_after": [ 12, 51234 ]
}
```

## 3. `scroll` (legacy — know it exists)

```json
POST /shakespeare/_search?scroll=1m
{ "size": 100, "query": { "match_all": {} } }

POST /_search/scroll
{ "scroll": "1m", "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAD4WYm9laVYtZndUQlNsdDcwakFMNjU1QQ==" }

DELETE /_search/scroll
{ "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gB..." }
```
:warning: Deprecated for real-time paging in 8.x. If a question says "deep pagination" the expected answer is `search_after` + PIT.

## Paginating aggregation results

Not the same thing — for buckets use the **composite** aggregation and its `after_key`:

```json
GET kibana_sample_data_ecommerce/_search?filter_path=aggregations
{
  "size": 0,
  "aggs": {
    "my_buckets": {
      "composite": {
        "size": 10,
        "sources": [ { "category": { "terms": { "field": "category.keyword" } } } ],
        "after": { "category": "Men's Shoes" }
      }
    }
  }
}
```

---

# Define and use index aliases

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/aliases.html <br>
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/indices-aliases.html

## part 1

:question: Define an index alias for `accounts-raw` called `accounts-all`

<details>
  <summary>View Solution (click to reveal)</summary>

The `_aliases` API is the one to learn — it is atomic and can do adds and removes in one call.

```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "accounts-raw",
        "alias": "accounts-all"
      }
    }
  ]
}
```

Check that the document count matches
```json
GET accounts-all/_count
```
</details>
<hr>

## part 2

:question: Define an index alias for `accounts-raw` called `accounts-male`

:question: Apply a filter to only show the male account owners.

<details>
  <summary>View Solution (click to reveal)</summary>

1. check that the field you want to filter is a keyword

```json
GET accounts-raw/_mapping/field/gender

// Output
{
  "accounts-raw" : {
    "mappings" : {
      "gender" : {
        "full_name" : "gender",
        "mapping" : {
          "gender" : {
            "type" : "text",
            "fields" : {
              "keyword" : { "type" : "keyword", "ignore_above" : 256 }
            }
          }
        }
      }
    }
  }
}
```

2. apply the alias with the filter

**Hint**: you need to use the `.keyword` field here, or you will get zero results.

```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "accounts-raw",
        "alias": "accounts-male",
        "filter": {
          "term": { "gender.keyword": "M" }
        }
      }
    }
  ]
}
```

3. test

```json
GET accounts-male/_count

// Output
{ "count" : 507 }
```

4. :question: BONUS: Run a query to do the same on the `accounts-raw` index, printing only the total hits

```json
POST accounts-raw/_search?filter_path=hits.total.value
{
  "query": { "match": { "gender": "M" } }
}

// Output
{ "hits" : { "total" : { "value" : 507 } } }
```
</details>
<hr>

## The rest of the alias syntax you need

### Zero-downtime swap (the classic "reindex with no downtime" question)
This is why the atomic `_aliases` API exists — both actions apply in a single cluster state update, so no request ever sees zero indices:

```json
POST /_aliases
{
  "actions": [
    { "remove": { "index": "accounts-v1", "alias": "accounts" } },
    { "add":    { "index": "accounts-v2", "alias": "accounts" } }
  ]
}
```

### One alias over many indices, with a designated write index
An alias pointing at more than one index is **read-only for writes** unless exactly one of them is flagged `is_write_index`:

```json
POST /_aliases
{
  "actions": [
    { "add": { "index": "logs-2026.08.01", "alias": "logs" } },
    { "add": { "index": "logs-2026.08.14", "alias": "logs", "is_write_index": true } }
  ]
}
```

### Wildcards, and removing an index entirely
```json
POST /_aliases
{
  "actions": [
    { "add":    { "index": "logs-2026.*", "alias": "logs-august" } },
    { "remove": { "index": "logs-2026.07.*", "alias": "logs" } },
    { "remove_index": { "index": "logs-2026.06.30" } }
  ]
}
```

### Routing and field-level control
```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "accounts-raw",
        "alias": "accounts-uk",
        "filter": { "term": { "state.keyword": "UK" } },
        "routing": "uk",
        "index_routing": "uk",
        "search_routing": "uk"
      }
    }
  ]
}
```

### Aliases defined in an index template
```json
PUT _index_template/accounts-tmpl
{
  "index_patterns": ["accounts-*"],
  "priority": 200,
  "template": {
    "aliases": {
      "accounts-all": {},
      "accounts-male": { "filter": { "term": { "gender.keyword": "M" } } }
    }
  }
}
```

### Inspecting aliases
```json
GET _alias/accounts-all              // which indices does this alias point at?
GET accounts-raw/_alias              // which aliases does this index have?
GET _cat/aliases?v
GET _alias                           // everything
```

### Shorthand (fine, but the `_aliases` API is safer to memorise)
```json
PUT accounts-raw/_alias/accounts-all
DELETE accounts-raw/_alias/accounts-all
```

:warning: Gotchas the exam likes:
- A **filtered alias** does not restrict writes — documents indexed through it can fall outside the filter. Only the query side is filtered.
- You cannot delete documents through an alias that points at multiple indices without a write index.
- Data streams have aliases too, but you add them with `"add": { "index": "my-data-stream", "alias": "..." }` and they cannot have a filter with `is_write_index` semantics identical to indices — check the docs if a question goes there.

---
---

# :books: BONUS — no longer listed in the 8.15 objectives

Everything below was an objective on older versions of the exam. Worth a read, not worth your last revision hour.

# (Bonus) Highlight the search terms in the response of a query
https://www.elastic.co/guide/en/elasticsearch/reference/8.15/highlighting.html

:question: In the spoken lines of the play, highlight the word `Hamlet` starting the highlight with `#aaa#` and ending it with `#bbb#`

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET shakespeare/_search
{
    "query": {
        "match": {
            "text_entry":  "Hamlet"
        }
    }, 
    "highlight": {
        "fields":  { 
          "text_entry": {
            "pre_tags": "#aaa#", 
            "post_tags": "#bbb#"
        }
      }
    }
}
```
</details>
<hr>

# (Bonus) Define and use a search template

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-template.html

:question: Create and use a search template that returns the lines of a person in a play.

:question: Show all lines that belong to `Attendant` in the play `Cymbeline`

<details>
  <summary>View Solution (click to reveal)</summary>

```json
POST _scripts/get_lines
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "bool": {
          "must": [
            {
              "term": {
                "play_name": "{{play_name}}"
              }
            },
            {
              "term": {
                "speaker": "{{ speaker }}"
              }
            }
          ]
        }
      }
    }
  }
}
```

## pull the template back

:warning: Note that it is not formatted nicely 😤

```json
GET _scripts/get_lines

// output

{
  "_id" : "get_lines",
  "found" : true,
  "script" : {
    "lang" : "mustache",
    "source" : """{"query":{"bool":{"must":[{"term":{"play_name":"{{play_name}}"}},{"term":{"speaker":"{{ speaker }}"}}]}}}""",
    "options" : {
      "content_type" : "application/json; charset=UTF-8"
    }
  }
}
```

## Test

Only read access to the underlying index is required.

```json
GET shakespeare/_search/template?filter_path=hits.hits.*.text_entry
{
    "id": "get_lines", 
    "params": {
        "play_name": "Cymbeline",
        "speaker" : "Attendant"
    }
}

// output

{
  "hits" : {
    "hits" : [
      {
        "_source" : {
          "text_entry" : "Please you, sir,"
        }
      },
      {
        "_source" : {
          "text_entry" : "Her chambers are all lockd; and theres no answer"
        }
      },
      {
        "_source" : {
          "text_entry" : "That will be given to the loudest noise we make."
        }
      }
    ]
  }
}
```

Render a template without running it (good for debugging):
```json
POST _render/template
{
  "id": "get_lines",
  "params": { "play_name": "Cymbeline", "speaker": "Attendant" }
}
```
</details>
<hr>
