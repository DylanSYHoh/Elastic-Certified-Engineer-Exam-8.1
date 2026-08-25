# Practice Questions — High-Frequency Exam Topics (8.15)

A drill file weighted towards **the topics that actually eat your three hours**, rather than one
question per objective. Every query in here was executed against a real Elasticsearch **8.15.0**
cluster with the Kibana sample data loaded, so the syntax and the stated answers are verified —
not copied out of a doc page.

Answers are collapsed. **Write your attempt in Dev Tools first, then reveal.**

---

## How these topics were chosen

Elastic does **not** publish percentage weightings for the Elastic Certified Engineer exam. What
is published is the objective list on the
[exam page](https://www.elastic.co/training/elastic-certified-engineer-exam) — 25 objectives for
8.15 — and what is consistently reported by people who have sat it is that the exam is roughly
**10 tasks in 3 hours**, with tasks chained together (task 6 searches the index task 3 built).

10 tasks against 25 objectives means **every task bundles several objectives**. That is what drives
the ranking below: a topic is "high frequency" not because it has its own objective, but because it
keeps turning up *inside* other people's tasks. Mapping, for example, is not one question — you
cannot finish an aggregation task if the field you need to group on was mapped as `text`.

| Tier | Topic | Why it ranks here |
| --- | --- | --- |
| **1** | **Aggregations + sub-aggregations** | Two explicit objectives, and the natural way to phrase "report on this data". Nearly always nested 2–3 levels with an `order` on a sub-aggregation. |
| **1** | **Mappings, multi-fields, dynamic templates** | Four objectives across Data Management + Data Processing. Also the silent dependency of every other task. |
| **1** | **Ingest pipelines** | One objective, but a big one — grok/dissect/script/conditionals/`on_failure`, plus `_simulate` to prove it. Usually paired with reindex. |
| **1** | **Query DSL — terms/phrases + bool** | Two objectives, and the entry point to most search tasks. The `term` vs `match` distinction is the single most common self-inflicted wound. |
| **1** | **Runtime fields (Painless)** | Two objectives (define one; search with one). Poorly covered by the docs, so it disproportionately costs people time. |
| **2** | **Reindex / update by query** | One objective, but it is the standard "now fix the index you just discovered is wrong" task. |
| **2** | **Index templates → data streams → ILM** | Two objectives that in practice arrive as one multi-step task: component templates, index template with `data_stream`, ILM policy, then prove it rolled over. |
| **2** | **Shard / cluster health troubleshooting** | One objective, almost always its own standalone task. |
| **2** | **Aliases, sorting, pagination** | Three objectives, small individually, usually bundled into one "search application" task. |
| **2** | **Snapshot, restore, SLM** | Three objectives in Cluster Management. Mechanical marks — do not lose them. |
| **3** | **Cross-cluster search / replication, searchable snapshots** | Three objectives, but they need a second cluster and a non-basic licence, so they are usually one task at most. |
| **3** | **Async search** | One objective, and the smallest one on the list. Learn the three endpoints and move on. |

:bulb: **The practical read:** Parts 1–5 below are where the marks are. If you only have one evening,
do Parts 1, 2 and 4.

### Full 8.15 objective list, for reference

<details>
  <summary>View the 25 objectives (click to reveal)</summary>

**Data Management**
1. Define an index that satisfies a given set of requirements
2. Define and use a dynamic template that satisfies a given set of requirements
3. Define an Index Lifecycle Management policy for a time-series index
4. Define an index template that creates a new data stream

**Data Processing**

5. Define a mapping that satisfies a given set of requirements
6. Define and use multi-fields with different data types and/or analyzers
7. Use the Reindex API and Update By Query API to reindex and/or update documents
8. Define and use an ingest pipeline that satisfies a given set of requirements
9. Define runtime fields to retrieve custom values using Painless scripting

**Searching Data**

10. Write and execute a search query for terms and/or phrases in one or more fields of an index
11. Write and execute a search query that is a Boolean combination of multiple queries and filters
12. Write an asynchronous search
13. Write and execute metric and bucket aggregations
14. Write and execute aggregations that contain sub-aggregations
15. Write and execute a query that searches across multiple clusters
16. Write and execute a search that utilizes a runtime field

**Developing Search Applications**

17. Sort the results of a query by a given set of requirements
18. Implement pagination of the results of a search query
19. Define and use index aliases

**Cluster Management**

20. Diagnose shard issues and repair a cluster's health
21. Backup and restore a cluster and/or specific indices
22. Configure a snapshot to be searchable
23. Configure a cluster for cross-cluster search
24. Implement cross-cluster replication
25. Automate snapshots with Snapshot Lifecycle Management

:warning: The exam moves to **Elastic 9.3 on 1 September 2026**. In 9.3, objectives 6, 9, 12, 14, 16,
17, 18 and all of Cluster Management as worded above are **removed**, and ES|QL, semantic search,
Streams, and RBAC are **added**. Everything in this file targets **8.15**.

</details>

---

## Prerequisites

| Dataset | Index | How to load |
| --- | --- | --- |
| eCommerce | `kibana_sample_data_ecommerce` (4,675 docs) | Kibana → Home → Try sample data |
| Flights | `kibana_sample_data_flights` (13,014 docs) | Kibana → Home → Try sample data |
| Web logs | `kibana_sample_data_logs` (14,074 docs) | Kibana → Home → Try sample data |
| Shakespeare | `shakespeare` (111,396 docs) | See [README.md](README.md) |

Any other data a question needs is supplied inline in that question.

### Two things that will confuse you if nobody warns you

1. **The sample data is generated relative to your install date.** `order_date`, `timestamp` and
   `@timestamp` are shifted so the data always looks "recent". Any question here that buckets by
   date will give **you** different numbers than it gave me. Where that happens the answer says
   *shape only* — the query is what is being graded, not the count. Non-date facets
   (`manufacturer`, `day_of_week`, `Carrier`, `category`) are stable, so those answers are exact.

2. **`kibana_sample_data_logs` is a data stream in 8.15**, not a plain index. Its real backing index
   is `.ds-kibana_sample_data_logs-<date>-000001`. That is a feature for these drills — it means you
   can practise data-stream behaviour on it — but it also means `PUT kibana_sample_data_logs/_doc/1`
   will be rejected. See Q55.

:warning: A few questions in Part 9 (searchable snapshots, CCR) need a **trial or platinum licence**.
On the default Docker `basic` licence they fail with
`current license is non-compliant`. Start a 30-day trial with:

```bash
curl -X POST "localhost:9200/_license/start_trial?acknowledge=true"
```

---

# Part 1 — Aggregations and sub-aggregations

> **Objectives 13 & 14.** The highest-value part of this file. Exam aggregation tasks are almost
> never a bare `terms` — they are "group by X, compute Y, keep only the top N by Y".

:bulb: Three habits that save you every time:
1. Always add `"size": 0` — you are not being asked for hits, and 10 hits of noise hide the aggs.
2. Group on a `keyword` field. Aggregating on `manufacturer` (a `text` field) fails with
   `Fielddata is disabled on [manufacturer] … Please use a keyword field instead` —
   `manufacturer.keyword` is what you want.
3. To sort buckets by a metric, the metric must be a **named sub-aggregation** and you reference
   that name in `order`.

<hr>

:question: **Q1.** In `kibana_sample_data_ecommerce`, find the **5 manufacturers with the most orders**, and for each one the **average order value** (`taxful_total_price`).

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: A `terms` bucket aggregation with an `avg` metric aggregation nested inside it. `manufacturer` is a `text` field with a `keyword` multi-field — you must aggregate on `manufacturer.keyword`.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-bucket-terms-aggregation.html

```json
GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "by_manufacturer": {
      "terms": {
        "field": "manufacturer.keyword",
        "size": 5
      },
      "aggs": {
        "avg_order_value": {
          "avg": { "field": "taxful_total_price" }
        }
      }
    }
  }
}
```

> Answer:
> | manufacturer | orders | avg order value |
> | --- | --- | --- |
> | Low Tide Media | 1553 | 84.40 |
> | Elitelligence | 1370 | 68.44 |
> | Oceanavigations | 1218 | 90.63 |
> | Tigress Enterprises | 1055 | 72.00 |
> | Pyramidustries | 947 | 67.80 |
>
> :warning: Note `sum_other_doc_count: 2789` in the response. Orders contain products from several
> manufacturers, so the bucket counts sum to more than 4,675. That is expected, not a bug.

</details>
<hr>

:question: **Q2.** Which **day of the week** generates the most revenue in the eCommerce data? Return all seven days, **ordered by revenue, highest first**.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: This is the core sub-aggregation skill: `order` points at the **name** of the sub-aggregation, not at a field.

```json
GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "by_day": {
      "terms": {
        "field": "day_of_week",
        "size": 7,
        "order": { "revenue": "desc" }
      },
      "aggs": {
        "revenue": { "sum": { "field": "taxful_total_price" } }
      }
    }
  }
}
```

> Answer: **Friday**, 58,215.59 (770 orders). Then Thursday 57,807.38, Saturday 53,841.04,
> Sunday 45,850.05, Monday 45,410.29, Wednesday 45,080.91, Tuesday 44,678.88.
>
> :bulb: `day_of_week` is already a `keyword` — no `.keyword` suffix. Check the mapping before you
> assume; guessing wrong costs you a round trip.

</details>
<hr>

:question: **Q3.** For the whole eCommerce index, return in **one request**: the number of distinct customers, the number of distinct SKUs, and the min / max / avg / sum of `taxful_total_price`.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Sibling metric aggregations at the top level. `stats` gives you four metrics for the price of one.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-metrics-stats-aggregation.html

```json
GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "unique_customers": { "cardinality": { "field": "customer_id" } },
    "unique_skus":      { "cardinality": { "field": "sku" } },
    "order_value":      { "stats": { "field": "taxful_total_price" } }
  }
}
```

> Answer: 46 customers, 7,186 SKUs; order value min 6.99, max 2250.00, avg 75.06, sum 350,884.13.
>
> :bulb: `cardinality` is **approximate** (HyperLogLog++). Exact below `precision_threshold`
> (default 3000). If a question says "exactly how many distinct…", say so or raise the threshold.

</details>
<hr>

:question: **Q4.** Split the eCommerce orders into **male** and **female** buckets and report, for each, the average basket value and the total number of items sold. Do it **without** a `terms` aggregation.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `filters` — named buckets, each defined by its own query. Use it when the buckets are not simply "every value of a field", or when you want to name them.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-bucket-filters-aggregation.html

```json
GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "by_gender": {
      "filters": {
        "filters": {
          "male":   { "term": { "customer_gender": "MALE" } },
          "female": { "term": { "customer_gender": "FEMALE" } }
        }
      },
      "aggs": {
        "avg_basket": { "avg": { "field": "taxful_total_price" } },
        "items":      { "sum": { "field": "total_quantity" } }
      }
    }
  }
}
```

> Answer: female — 2,433 orders, avg 73.52, 5,174 items. male — 2,242 orders, avg 76.72, 4,917 items.

</details>
<hr>

:question: **Q5.** Produce a distribution of eCommerce order values in **fixed £50 bands**, including bands with no orders.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `histogram` for numeric fields with a fixed `interval`. `min_doc_count: 0` keeps the empty bands so the distribution has no holes.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-bucket-histogram-aggregation.html

```json
GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "value_bands": {
      "histogram": {
        "field": "taxful_total_price",
        "interval": 50,
        "min_doc_count": 0
      }
    }
  }
}
```

> Answer: 0–50 → 1633, 50–100 → 2036, 100–150 → 724, 150–200 → 205, 200–250 → 53, 250–300 → 14,
> 300–350 → 7, 350–400 → 2, then empty bands all the way to a single order at 2250.

</details>
<hr>

:question: **Q6.** Bucket the flights into **short (< 2,000 km)**, **medium (2,000–8,000 km)** and **long (≥ 8,000 km)** and give the average delay in minutes for each. The bucket keys must be those three words.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `range` aggregation. `from` is **inclusive**, `to` is **exclusive**. `keyed: true` turns the array of buckets into an object keyed by name — much easier to read.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-bucket-range-aggregation.html

```json
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "distance_bands": {
      "range": {
        "field": "DistanceKilometers",
        "keyed": true,
        "ranges": [
          { "key": "short",  "to": 2000 },
          { "key": "medium", "from": 2000, "to": 8000 },
          { "key": "long",   "from": 8000 }
        ]
      },
      "aggs": {
        "avg_delay": { "avg": { "field": "FlightDelayMin" } }
      }
    }
  }
}
```

> Answer: short 2,947 flights / 48.12 min; medium 4,283 / 46.91 min; long 5,784 / 47.28 min.

</details>
<hr>

:question: **Q7.** For the **3 busiest origin countries** in the flights data, show the **3 most common destination countries** from each.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `terms` inside `terms` — the textbook sub-aggregation. Keep the inner `size` small; buckets multiply.

```json
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "by_origin": {
      "terms": { "field": "OriginCountry", "size": 3 },
      "aggs": {
        "top_dest": {
          "terms": { "field": "DestCountry", "size": 3 }
        }
      }
    }
  }
}
```

> Answer: IT (2269) → IT 458, US 327, CN 194 · US (1995) → IT 379, US 321, CN 185 ·
> JP (797) → IT 192, US 117, JP 63.

</details>
<hr>

:question: **Q8.** For the **2 largest eCommerce categories**, show the **2 manufacturers generating the most revenue** in each, with that revenue. Three levels deep.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `terms` → `terms` ordered by → `sum`. The inner `order` references the name of the metric agg that is nested inside the *inner* terms agg.

```json
GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category.keyword", "size": 2 },
      "aggs": {
        "by_manufacturer": {
          "terms": {
            "field": "manufacturer.keyword",
            "size": 2,
            "order": { "revenue": "desc" }
          },
          "aggs": {
            "revenue": { "sum": { "field": "taxful_total_price" } }
          }
        }
      }
    }
  }
}
```

> Answer: **Men's Clothing** (2024) → Low Tide Media 89,114.43, Elitelligence 83,531.21 ·
> **Women's Clothing** (1903) → Tigress Enterprises 58,768.50, Pyramidustries 50,627.29.
>
> :bulb: Note Low Tide Media has *fewer* docs than Elitelligence in Men's Clothing but more revenue —
> which is exactly why `order` by a metric is not the same as `order` by `_count`.

</details>
<hr>

:question: **Q9.** For each of the top 3 eCommerce categories, return the **single most expensive order** — its `order_id` and price, not just the max value.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `max` gives you the number; `top_hits` gives you the **document**. Whenever a question says "which order / which flight / which line", reach for `top_hits`.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-metrics-top-hits-aggregation.html

```json
GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category.keyword", "size": 3 },
      "aggs": {
        "priciest": {
          "top_hits": {
            "size": 1,
            "sort": [ { "taxful_total_price": "desc" } ],
            "_source": [ "order_id", "taxful_total_price", "customer_full_name" ]
          }
        }
      }
    }
  }
}
```

> Answer: Men's Clothing → order 739290 at 2250.00. Women's Clothing and Women's Shoes both →
> order 731279 at 344.00 (one order can carry several categories, so it appears in both buckets).

</details>
<hr>

:question: **Q10.** In the `shakespeare` index, find the **5 speakers with the most lines**, and for each, **how many different plays** they appear in.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET shakespeare/_search
{
  "size": 0,
  "aggs": {
    "top_speakers": {
      "terms": { "field": "speaker", "size": 5 },
      "aggs": {
        "plays": { "cardinality": { "field": "play_name" } }
      }
    }
  }
}
```

> Answer: GLOUCESTER 1920 lines / **6 plays**, HAMLET 1582 / 1, IAGO 1161 / 1, FALSTAFF 1117 / 2,
> KING HENRY V 1086 / 1.
>
> :bulb: GLOUCESTER across 6 plays is the interesting one — a good reminder that a `terms` bucket
> groups by *value*, not by *entity*.

</details>
<hr>

:question: **Q11.** :fire: In the web logs, find the **5 destination countries with the highest average response size** (`bytes`) — but only consider countries with **at least 100 requests**.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: **This is the trap.** The obvious answer — `order` by the sub-agg plus `min_doc_count: 100` — returns an **empty bucket list**. `terms` selects its top-N candidates *on each shard first*, using `shard_min_doc_count` (default 0), and only applies `min_doc_count` at the coordinating node. Ordering by a metric makes the shard picks tiny one-doc countries, all of which are then filtered out, leaving nothing.

Two correct fixes — both give the same answer:

**(a) Push the threshold down to the shards:**

```json
GET kibana_sample_data_logs/_search
{
  "size": 0,
  "aggs": {
    "by_dest": {
      "terms": {
        "field": "geo.dest",
        "size": 5,
        "min_doc_count": 100,
        "shard_min_doc_count": 100,
        "order": { "avg_bytes": "desc" }
      },
      "aggs": {
        "avg_bytes": { "avg": { "field": "bytes" } }
      }
    }
  }
}
```

**(b) Collect wide, then filter and trim with pipeline aggregations:**

```json
GET kibana_sample_data_logs/_search
{
  "size": 0,
  "aggs": {
    "by_dest": {
      "terms": { "field": "geo.dest", "size": 250, "order": { "avg_bytes": "desc" } },
      "aggs": {
        "avg_bytes":   { "avg": { "field": "bytes" } },
        "big_enough":  { "bucket_selector": { "buckets_path": { "c": "_count" }, "script": "params.c >= 100" } },
        "top5":        { "bucket_sort": { "sort": [ { "avg_bytes": "desc" } ], "size": 5 } }
      }
    }
  }
}
```

> Answer: TH 6403.47 (130 reqs), IT 6269.40 (116), DE 6142.54 (177), ET 5980.72 (172), GB 5871.18 (116).
>
> :bulb: Without the threshold the "top 5" is BZ, SB and GQ — countries with **one request each**.
> If an aggregation answer looks absurd, check `doc_count` before you trust it.

</details>
<hr>

:question: **Q12.** List only the airlines whose **average delay exceeds 47 minutes**, with that average. Do the filtering server-side.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `bucket_selector` is a **pipeline aggregation**: it runs after the buckets are built and drops the ones whose script returns `false`. `buckets_path` maps a variable name to a sibling metric.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-pipeline-bucket-selector-aggregation.html

```json
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "by_carrier": {
      "terms": { "field": "Carrier", "size": 10 },
      "aggs": {
        "avg_delay": { "avg": { "field": "FlightDelayMin" } },
        "delayed_only": {
          "bucket_selector": {
            "buckets_path": { "d": "avg_delay" },
            "script": "params.d > 47"
          }
        }
      }
    }
  }
}
```

> Answer: Logstash Airways 49.60 (3,323 flights) and ES-Air 47.45 (3,211). Kibana Airlines (46.38)
> and JetBeats (45.91) are excluded.

</details>
<hr>

:question: **Q13.** Build a weekly revenue report for the eCommerce data showing, per week: revenue, **running total to date**, and **change vs the previous week**. Also return the **average weekly revenue** as a single number.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Three pipeline aggregations at once. `cumulative_sum` and `derivative` are **siblings of the metric inside each bucket**; `avg_bucket` sits **outside** the histogram and points into it with the `>` path separator.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-pipeline.html

```json
GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "per_week": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "week",
        "min_doc_count": 0
      },
      "aggs": {
        "revenue":       { "sum": { "field": "taxful_total_price" } },
        "running_total": { "cumulative_sum": { "buckets_path": "revenue" } },
        "wow_change":    { "derivative": { "buckets_path": "revenue" } }
      }
    },
    "avg_weekly_revenue": {
      "avg_bucket": { "buckets_path": "per_week>revenue" }
    }
  }
}
```

> Answer: *shape only* — the sample data is generated relative to your install date, so your weeks
> will differ. Check that: the **first** bucket has a `running_total` equal to its own `revenue` and
> **no** `wow_change` (nothing to subtract from), and that `avg_weekly_revenue` sits outside `per_week`.
>
> :bulb: `calendar_interval` (`week`, `month`, `quarter`, `year`) respects calendar boundaries;
> `fixed_interval` (`7d`, `30m`) does not. A question saying "per calendar month" means
> `calendar_interval: month`.

</details>
<hr>

:question: **Q14.** Which single week had the **highest** revenue, and what was it?

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `max_bucket` — a sibling pipeline agg that returns both the value and the key(s) that produced it.

```json
GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "per_week": {
      "date_histogram": { "field": "order_date", "calendar_interval": "week" },
      "aggs": {
        "revenue": { "sum": { "field": "taxful_total_price" } }
      }
    },
    "best_week": {
      "max_bucket": { "buckets_path": "per_week>revenue" }
    }
  }
}
```

> Answer: *shape only* (date-relative data). The response gives you `value` plus a `keys` **array** —
> `max_bucket` returns every key that ties for the maximum.

</details>
<hr>

:question: **Q15.** Find the 3 busiest **city-pair routes** (origin city → destination city) in the flights data, with the average ticket price on each. One aggregation, not two nested ones.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `multi_terms` builds one bucket per **combination** of field values. Nested `terms` would give you "top origins, then top dests within each" — a different question.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-bucket-multi-terms-aggregation.html

```json
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "routes": {
      "multi_terms": {
        "size": 3,
        "terms": [
          { "field": "OriginCityName" },
          { "field": "DestCityName" }
        ]
      },
      "aggs": {
        "avg_price": { "avg": { "field": "AvgTicketPrice" } }
      }
    }
  }
}
```

> Answer: Zurich→Zurich 144 flights / 215.14, Xi'an→Xi'an 58 / 221.03, Xi'an→Zurich 54 / 627.92.
>
> :bulb: `multi_terms` is slower than `terms` and cannot be ordered by a sub-agg on all shards
> reliably at large cardinality — fine at exam scale, worth knowing the caveat.

</details>
<hr>

:question: **Q16.** Return the average basket value **for female customers**, alongside the average basket value **across all customers**, in one request whose `hits` are restricted to female customers.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `global` escapes the query scope. Every other aggregation in the request sees only the query's matches; a `global` agg sees the whole index. This is the standard "compare my slice to the baseline" pattern.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-bucket-global-aggregation.html

```json
GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "query": { "term": { "customer_gender": "FEMALE" } },
  "aggs": {
    "female_avg": { "avg": { "field": "taxful_total_price" } },
    "everyone": {
      "global": {},
      "aggs": {
        "overall_avg": { "avg": { "field": "taxful_total_price" } }
      }
    }
  }
}
```

> Answer: female 73.52, overall 75.06 (`doc_count` on the global bucket is the full 4,675).

</details>
<hr>

:question: **Q17.** Report the **median, 95th and 99th percentile** response size in the web logs, and what **percentage** of responses are under 1,000 and under 5,000 bytes.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `percentiles` goes value → rank; `percentile_ranks` goes rank → value. Exam wording "what proportion is below X" means `percentile_ranks`.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/search-aggregations-metrics-percentile-aggregation.html

```json
GET kibana_sample_data_logs/_search
{
  "size": 0,
  "aggs": {
    "sizes": {
      "percentiles": { "field": "bytes", "percents": [ 50, 95, 99 ] }
    },
    "under": {
      "percentile_ranks": { "field": "bytes", "values": [ 1000, 5000 ] }
    }
  }
}
```

> Answer: p50 ≈ 5,479 · p95 ≈ 10,483 · p99 ≈ 17,901 bytes. About **8.5 %** of responses are under
> 1,000 bytes and **44.9 %** under 5,000.
>
> :bulb: Percentiles are approximate (TDigest) — small variations between runs are normal.

</details>
<hr>

# Part 2 — Runtime fields and Painless

> **Objectives 9 & 16.** Two objectives, thin documentation, and a scripting language most people
> only use here. Disproportionately expensive if you have not drilled it.

:bulb: The three places a runtime field can live, and when to use each:

| Where | Syntax | Use when |
| --- | --- | --- |
| In the **search request** | `"runtime_mappings": { ... }` at the top of `_search` | One-off. The question says "write a search that…". |
| In the **mapping** | `PUT idx/_mapping` with a `"runtime"` block | Reusable. The question says "define a runtime field on the index". No reindex needed. |
| In the **index template** | `"runtime"` inside `template.mappings` | Every future index gets it. |

:bulb: Painless survival kit:
- `emit(value)` is how a runtime field produces its value. **No `return` of a value** — `return;` bare is how you emit *nothing*.
- `doc['field'].value` reads a doc value. **The field must have doc values** — that means `keyword`, not `text`. Use `field.keyword`.
- Guard missing fields with `doc['f'].size() == 0`, and guard nulls **explicitly** — `?.` alone will still NPE in a comparison.
- Numeric doc values on `integer`/`short`/`long` fields all come back as **`long`**. `int x = doc['n'].value;` throws `cannot convert MethodHandle(Longs)long to (Object)int`. Use `long` or `def`.

<hr>

:question: **Q18.** Without changing the mapping, run a search on the flights data that reports the **average flight duration in hours** per airline. The index only stores minutes.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `runtime_mappings` in the search body. The field exists only for this request, and you can aggregate on it exactly like a real field.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/runtime-search-request.html

```json
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "runtime_mappings": {
    "FlightDurationHours": {
      "type": "double",
      "script": {
        "source": "emit(doc['FlightTimeMin'].value / 60)"
      }
    }
  },
  "aggs": {
    "by_carrier": {
      "terms": { "field": "Carrier" },
      "aggs": {
        "avg_hours": { "avg": { "field": "FlightDurationHours" } }
      }
    }
  }
}
```

> Answer: Logstash Airways 8.60 h, JetBeats 8.54 h, ES-Air 8.53 h, Kibana Airlines 8.40 h.

</details>
<hr>

:question: **Q19.** :fire: Classify every flight as `on time` (0 min delay), `minor` (1–60 min) or `major` (> 60 min) using a runtime field, and count each class.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: A `keyword` runtime field with branching. **This is where most people lose 10 minutes**: `FlightDelayMin` is mapped `integer`, but its doc value comes back as a **`long`**. Writing `int d = doc['FlightDelayMin'].value;` fails with:
>
> `cannot convert MethodHandle(Longs)long to (Object)int`

```json
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "runtime_mappings": {
    "delay_band": {
      "type": "keyword",
      "script": {
        "source": """
          long d = doc['FlightDelayMin'].value;
          if (d == 0)       { emit('on time'); }
          else if (d <= 60) { emit('minor'); }
          else              { emit('major'); }
        """
      }
    }
  },
  "aggs": {
    "bands": { "terms": { "field": "delay_band" } }
  }
}
```

> Answer: on time 9,744 · major 2,721 · minor 549.
>
> :bulb: The `"""triple quotes"""` form is Kibana Dev Tools only — it lets you write multi-line
> Painless without escaping. If you paste into `curl` you must use `\n` in a normal JSON string.

</details>
<hr>

:question: **Q20.** Add a **permanent** runtime field to `kibana_sample_data_ecommerce` called `order_discount`, holding the total discount across all products in the order. Then report the total discount given, and how many orders were discounted at all.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: A mapping-level runtime field — `PUT _mapping` with a `runtime` block. It is added **without a reindex** because the value is computed at query time. `products.discount_amount` is an array, so iterate the doc values.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/runtime-mapping-fields.html

```json
PUT kibana_sample_data_ecommerce/_mapping
{
  "runtime": {
    "order_discount": {
      "type": "double",
      "script": {
        "source": """
          double t = 0;
          for (d in doc['products.discount_amount']) { t += d; }
          emit(t);
        """
      }
    }
  }
}
```

```json
GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "total_discount": { "sum": { "field": "order_discount" } },
    "orders": {
      "range": {
        "field": "order_discount",
        "ranges": [
          { "key": "no discount", "to": 0.01 },
          { "key": "discounted",  "from": 0.01 }
        ]
      }
    }
  }
}
```

> Answer: 1,018.00 total discount, across **81** discounted orders (4,594 with none).
>
> :bulb: To remove it again: `PUT kibana_sample_data_ecommerce/_mapping` with
> `{"runtime": {"order_discount": null}}`. Setting the field to `null` is the documented delete.
> (The sample index already ships with three runtime fields — `order_day`, `price_after_discount`,
> `weekend` — so do not wipe the whole `runtime` block.)

</details>
<hr>

:question: **Q21.** The web logs store a full `referer` URL. Without reindexing, report the **top referring domains**.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: String surgery in Painless. `referer` is already a `keyword`, so it has doc values. Bare `return;` skips a document that has no usable value.

```json
GET kibana_sample_data_logs/_search
{
  "size": 0,
  "runtime_mappings": {
    "referer_domain": {
      "type": "keyword",
      "script": {
        "source": """
          if (doc['referer'].size() == 0) { return; }
          String r = doc['referer'].value;
          int s = r.indexOf('://');
          if (s < 0) { return; }
          int e = r.indexOf('/', s + 3);
          emit(e < 0 ? r.substring(s + 3) : r.substring(s + 3, e));
        """
      }
    }
  },
  "aggs": {
    "domains": { "terms": { "field": "referer_domain", "size": 5 } }
  }
}
```

> Answer: www.elastic-elastic-elastic.com 6,119 · twitter.com 4,261 · facebook.com 2,474 ·
> nytimes.com 1,220.

</details>
<hr>

:question: **Q22.** Show the **hourly traffic profile** of the web logs — request count for each hour of the day 0–23 — using a runtime field.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `doc['@timestamp'].value` is a `ZonedDateTime`, so all the Java date methods are available: `getHour()`, `getDayOfWeek()`, `getMonthValue()`, `getYear()`. `min_doc_count: 0` keeps quiet hours in the output.

```json
GET kibana_sample_data_logs/_search
{
  "size": 0,
  "runtime_mappings": {
    "hour_of_day": {
      "type": "long",
      "script": { "source": "emit(doc['@timestamp'].value.getHour())" }
    }
  },
  "aggs": {
    "by_hour": {
      "histogram": { "field": "hour_of_day", "interval": 1, "min_doc_count": 0 }
    }
  }
}
```

> Answer: a clean daytime curve — trough at 23:00 (10 requests), peak at 12:00 (1,656).
> Your absolute counts may differ slightly if your sample data was generated on a different day,
> but the shape holds.

</details>
<hr>

:question: **Q23.** Return the **10 flights with the worst delay relative to their flight time** — flights delayed by at least as long as they spend in the air. Sort by that ratio.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: A runtime field can be **filtered on and sorted on**, not just aggregated. Note the two guards: missing fields, and division by zero (some flights have `FlightTimeMin: 0`).

```json
GET kibana_sample_data_flights/_search
{
  "size": 10,
  "runtime_mappings": {
    "delay_ratio": {
      "type": "double",
      "script": {
        "source": """
          if (doc['FlightDelayMin'].size() == 0 || doc['FlightTimeMin'].size() == 0) { return; }
          double t = doc['FlightTimeMin'].value;
          if (t <= 0) { return; }
          emit(doc['FlightDelayMin'].value / t);
        """
      }
    }
  },
  "query": {
    "bool": { "filter": [ { "range": { "delay_ratio": { "gte": 1 } } } ] }
  },
  "sort": [ { "delay_ratio": "desc" } ],
  "fields": [ "delay_ratio", "FlightNum", "FlightDelayMin", "FlightTimeMin" ],
  "_source": false
}
```

> Answer: **176** flights match, all with `delay_ratio` exactly 1.0 (the generator sets the delay
> equal to the flight time in those cases).
>
> :bulb: `"fields": [...]` with `"_source": false` is the **only** way to see a runtime field's value
> in the hits — it is not in `_source`, because it does not exist on disk.

</details>
<hr>

:question: **Q24.** Group web log requests by **HTTP status class** — `2xx`, `4xx`, `5xx` — without touching the mapping.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `response` is `text` with a `.keyword` multi-field. `doc['response']` would fail ("Fielddata is disabled"); you must use `doc['response.keyword']`.

```json
GET kibana_sample_data_logs/_search
{
  "size": 0,
  "runtime_mappings": {
    "status_class": {
      "type": "keyword",
      "script": {
        "source": """
          if (doc['response.keyword'].size() == 0) { return; }
          emit(doc['response.keyword'].value.substring(0, 1) + 'xx');
        """
      }
    }
  },
  "aggs": {
    "classes": { "terms": { "field": "status_class" } }
  }
}
```

> Answer: 2xx 12,832 · 4xx 801 · 5xx 441.

</details>
<hr>

:question: **Q25.** :fire: Report the number of **failed** web requests (status ≥ 400), broken down by file extension, by defining a **boolean** runtime field and querying on it.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Runtime fields can be `boolean`, `long`, `double`, `date`, `keyword`, `ip` or `geo_point`. A `boolean` one can be used directly in a `term` query — that is the point of the objective's word "utilizes".

```json
GET kibana_sample_data_logs/_search
{
  "size": 0,
  "runtime_mappings": {
    "is_error": {
      "type": "boolean",
      "script": {
        "source": """
          emit(doc['response.keyword'].size() > 0
               && doc['response.keyword'].value.compareTo('400') >= 0);
        """
      }
    }
  },
  "query": { "term": { "is_error": true } },
  "aggs": {
    "by_extension": { "terms": { "field": "extension.keyword", "size": 3 } }
  }
}
```

> Answer: **1,242** failed requests (801 + 441 — consistent with Q24). Top extensions: `""` 531,
> `gz` 211, `css` 173.

</details>
<hr>

:question: **Q26.** :fire: `DistanceKilometers` in the flights index is unreliable. Override it — for this search only — with a value recalculated from `DistanceMiles`, and prove the override took effect.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: **Shadowing.** A runtime field defined with the *same name* as a mapped field takes precedence at query time. The indexed values are untouched and `_source` still shows the old value — which is exactly how you prove it.

```json
GET kibana_sample_data_flights/_search
{
  "size": 2,
  "runtime_mappings": {
    "DistanceKilometers": {
      "type": "double",
      "script": { "source": "emit(Math.round(doc['DistanceMiles'].value * 1.60934))" }
    }
  },
  "query": { "range": { "DistanceMiles": { "gt": 5000 } } },
  "fields": [ "DistanceKilometers", "DistanceMiles" ],
  "_source": false
}
```

> Answer: e.g. `DistanceMiles: 10247.856` now returns `DistanceKilometers: 16492.0` — rounded, and
> derived, rather than the stored float.
>
> :bulb: Classic exam framing: *"a field was populated incorrectly, correct it without reindexing."*
> Shadowing is the answer. If they say "correct it **permanently on disk**", it is `_update_by_query`
> instead (Part 5).

</details>
<hr>

:question: **Q27.** Extract the **file extension** from the `url` field in the web logs with a `grok` pattern **inside a runtime field**, and return it alongside the URL.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Painless in a runtime field can call `grok('PATTERN').extract(value)` and `dissect('PATTERN').extract(value)`. It returns a `Map`, or `null` on no match — always null-check it.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/runtime-examples.html

```json
GET kibana_sample_data_logs/_search
{
  "size": 1,
  "runtime_mappings": {
    "url_ext": {
      "type": "keyword",
      "script": {
        "source": """
          if (doc['url.keyword'].size() == 0) { return; }
          def m = grok('%{GREEDYDATA}\\.%{WORD:ext}').extract(doc['url.keyword'].value);
          if (m != null) { emit(m.ext); }
        """
      }
    }
  },
  "query": { "exists": { "field": "url.keyword" } },
  "fields": [ "url_ext", "url" ],
  "_source": false
}
```

> Answer: e.g. `https://artifacts.elastic.co/downloads/.../elasticsearch-6.3.2.deb_1` → `deb_1`.
>
> :warning: **Backslashes bite here.** Painless single-quoted strings accept only two escape
> sequences, `\\` and `\'`. So the Painless source for a regex dot must read `\\.` — and how you
> type that depends on where you are typing it:
>
> | Where you type it | What you type |
> | --- | --- |
> | Dev Tools triple quotes (as above) — passed through literally | `\\.` |
> | A normal JSON `"string"` (curl, or a one-line `"source"`) | `\\\\.` |
>
> Typing a single `\.` inside triple quotes will not compile — you get
> `unexpected character ['%{GREEDYDATA}\.]. The only valid escape sequences in strings starting with ['] are [\\] and [\'].`

</details>
<hr>

# Part 3 — Mappings, multi-fields, analyzers, dynamic templates

> **Objectives 1, 2, 5 & 6.** Four of the 25 objectives, and the silent prerequisite for everything
> else. A task that says "index this data and then report on it" is really a mapping task with an
> aggregation attached.

:bulb: Before writing any mapping, ask three questions of every field:
1. **Do I need to match parts of it?** → `text`. **Whole value only?** → `keyword`. **Both?** → multi-field.
2. **Do I need to aggregate, sort or script on it?** → it needs **doc values**, so not `text`.
3. **Do I need it back but never search it?** → `"index": false` (still aggregatable) or `_source` excludes.

<hr>

:question: **Q28.** Create an index `products` for the documents below with these requirements: `sku` matched exactly only; `title` and `description` full-text searchable, **and searchable together through one field**; `price` stored to 2 decimal places with a missing price treated as `0`; `released` accepts `yyyy-MM-dd` or epoch millis and **must not reject a document with a bad date**; `stock` returned but never searched; unexpected fields must be **rejected**.

```json
{ "sku": "AB-1", "title": "Blue Widget", "description": "a sturdy widget",
  "price": null, "released": "not-a-date", "stock": 5 }
```

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Six mapping parameters in one question — this is very close to the real exam's phrasing style. Map the requirements one at a time:

| Requirement | Parameter |
| --- | --- |
| exact match only | `keyword` |
| searchable together | `copy_to` into a shared `text` field |
| 2 decimal places | `scaled_float` with `scaling_factor: 100` |
| missing price → 0 | `null_value: 0` |
| two date formats | `format: "yyyy-MM-dd||epoch_millis"` |
| bad date must not reject the doc | `ignore_malformed: true` |
| returned but not searched | `"index": false` |
| reject unknown fields | `"dynamic": "strict"` |

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/mapping-params.html

```json
PUT products
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 0 },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "sku":         { "type": "keyword" },
      "title":       { "type": "text", "copy_to": "all_text" },
      "description": { "type": "text", "copy_to": "all_text" },
      "all_text":    { "type": "text" },
      "price":       { "type": "scaled_float", "scaling_factor": 100, "null_value": 0 },
      "released":    { "type": "date", "format": "yyyy-MM-dd||epoch_millis", "ignore_malformed": true },
      "stock":       { "type": "integer", "index": false }
    }
  }
}
```

> Answer / how to prove each one:
> - `GET products/_search {"query":{"match":{"all_text":"sturdy"}}}` → 1 hit (`copy_to` works; note
>   `all_text` is **not** in `_source` — it is built at index time only).
> - `{"query":{"term":{"price":0}}}` → 1 hit (`null_value` substituted).
> - The document indexes successfully and comes back with `"_ignored": ["released"]`, with the raw
>   `"not-a-date"` still visible in `_source`.
> - Indexing `{"sku":"AB-2","colour":"red"}` is rejected:
>   `mapping set to strict, dynamic introduction of [colour] within [_doc] is not allowed`.
>
> :warning: Two traps in this one:
> - `doc_values` is **not** a valid parameter on `text` — `unknown parameter [doc_values] on mapper
>   [...] of type [text]`. Text fields use `fielddata`, not doc values.
> - `"index": false` does **not** always make a field unsearchable. On `stock` (an `integer`, which
>   keeps doc values) a `term` query still returns the hit via a doc-values fallback. On a `text`
>   field it genuinely fails: `Cannot search on field [...] since it is not indexed.`

</details>
<hr>

:question: **Q29.** Define a multi-field on `title` giving you: full-text search, **exact** sorting/aggregating, and **stemmed** English search — all from one input field. Show a query that only the stemmed sub-field can satisfy.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Multi-fields index the **same source value** several ways. No extra `_source` cost, one extra inverted index each.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/multi-fields.html

```json
PUT catalog
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 },
          "english": { "type": "text", "analyzer": "english" }
        }
      }
    }
  }
}
```

```json
POST catalog/_doc/1?refresh
{ "title": "The Running Shoes" }
```

> Answer, indexing `"The Running Shoes"`:
>
> | Query | Hits | Why |
> | --- | --- | --- |
> | `match` on `title` for `running` | 1 | `standard` analyzer, lowercased token `running` |
> | `match` on `title` for `run` | **0** | no stemming on the `standard` analyzer |
> | `match` on `title.english` for `run` | **1** | `english` analyzer stems `running` → `run` |
> | `term` on `title.keyword` for `The Running Shoes` | 1 | keyword keeps the exact string |
> | `term` on `title.keyword` for `the running shoes` | **0** | keyword is **not** lowercased |
>
> :bulb: You can add a new sub-field to an existing mapping at any time — but only **new documents**
> get it. To populate it for existing docs, `_update_by_query` (Q45).

</details>
<hr>

:question: **Q30.** Build a custom analyzer `product_analyzer` that: strips HTML, converts `:)` to `_happy_` and `:(` to `_sad_`, lowercases, folds accents, removes the stopwords `the`, `a`, `and`, and stems English. Prove it with `_analyze` on `<b>The</b> Amazing Résumé Runners :) and running`.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: The order is fixed and matters: **char_filter → tokenizer → filter**. Char filters see raw characters (so `html_strip` and the `mapping` filter go there); token filters see tokens.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/analysis-custom-analyzer.html

```json
PUT catalog-analyzed
{
  "settings": {
    "analysis": {
      "char_filter": {
        "emoticons": {
          "type": "mapping",
          "mappings": [ ":) => _happy_", ":( => _sad_" ]
        }
      },
      "filter": {
        "my_stop": { "type": "stop", "stopwords": [ "the", "a", "and" ] },
        "my_stemmer": { "type": "snowball", "language": "English" }
      },
      "analyzer": {
        "product_analyzer": {
          "type": "custom",
          "char_filter": [ "html_strip", "emoticons" ],
          "tokenizer": "standard",
          "filter": [ "lowercase", "asciifolding", "my_stop", "my_stemmer" ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "product_analyzer" }
    }
  }
}
```

```json
GET catalog-analyzed/_analyze
{
  "analyzer": "product_analyzer",
  "text": "<b>The</b> Amazing Résumé Runners :) and running"
}
```

> Answer: `amaz`, `resum`, `runner`, `_happy_`, `run`.
>
> Note what each stage did: `<b>` gone (html_strip), `The`/`and` gone (my_stop), `Résumé` → `resum`
> (asciifolding then snowball), `:)` survived tokenization as `_happy_` (mapping char filter — the
> standard tokenizer would otherwise have thrown the punctuation away).
>
> :warning: Analysis settings are **static**. To change an analyzer on an existing index you must
> `POST /idx/_close`, `PUT /idx/_settings`, `POST /idx/_open` — or, more usually in the exam,
> create a new index and reindex.

</details>
<hr>

:question: **Q31.** Add search-as-you-type behaviour to a `name` field: typing `ela` must match `Elastic`, but searching for the full word must not explode into every prefix.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: The `analyzer` / `search_analyzer` split. Index-time you generate every prefix; **search-time you must not**, or `Elastic` would be tokenised into `el, ela, elas…` and match far too much.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/analysis-edgengram-tokenizer.html

```json
PUT autocomplete-demo
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "autocomplete_tok": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 10,
          "token_chars": [ "letter", "digit" ]
        }
      },
      "analyzer": {
        "autocomplete": { "tokenizer": "autocomplete_tok", "filter": [ "lowercase" ] }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "autocomplete",
        "search_analyzer": "standard"
      }
    }
  }
}
```

```json
GET autocomplete-demo/_analyze
{ "field": "name", "text": "Elastic" }
```

> Answer: the **field** analyzer produces `el, ela, elas, elast, elasti, elastic` — six tokens.
> Because `search_analyzer` is `standard`, a query for `ela` is a single token `ela`, which matches
> the indexed `ela` gram.
>
> :bulb: `GET idx/_analyze {"field": "name", ...}` runs the **index** analyzer for that field. There
> is no `_analyze` switch for the search analyzer — reason about it separately.

</details>
<hr>

:question: **Q32.** Create an index where, **without listing any fields by name**: every string becomes a `keyword` capped at 256 chars, except fields whose name ends in `_text`, which must be full-text; every whole number becomes an `integer` rather than a `long`; and anything under a `debug` object is stored but never indexed.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Dynamic templates, and **order matters** — they are evaluated top to bottom and the first match wins. Put the specific `match` rules **above** the catch-all `match_mapping_type` rules.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/dynamic-templates.html

```json
PUT events
{
  "mappings": {
    "dynamic_templates": [
      {
        "text_suffix": {
          "match": "*_text",
          "match_mapping_type": "string",
          "mapping": { "type": "text" }
        }
      },
      {
        "strings_as_keyword": {
          "match_mapping_type": "string",
          "mapping": { "type": "keyword", "ignore_above": 256 }
        }
      },
      {
        "longs_as_integer": {
          "match_mapping_type": "long",
          "mapping": { "type": "integer" }
        }
      },
      {
        "debug_not_indexed": {
          "path_match": "debug.*",
          "mapping": { "type": "object", "enabled": false }
        }
      }
    ]
  }
}
```

```json
POST events/_doc/1?refresh
{ "order_id": "A-1", "note_text": "hello world", "qty": 7, "debug": { "trace": { "a": 1 } } }
```

> Answer, from `GET events/_mapping`: `order_id` → `keyword` (ignore_above 256),
> `note_text` → `text`, `qty` → `integer`, `debug.trace` → `object` with `"enabled": false`.
>
> :bulb: The matchers, in decreasing order of how often the exam uses them:
> `match` / `unmatch` (field **name**, wildcard) · `match_mapping_type` (JSON type ES detected) ·
> `path_match` / `path_unmatch` (full dotted **path**) · `match_pattern: "regex"` to make `match`
> a regex. Swap in `{{name}}` inside the mapping to reuse the field's name.

</details>
<hr>

:question: **Q33.** :fire: The eCommerce `products` field is an **array of objects**. Show that this makes the query "orders containing an Oceanavigations item costing £100 or more" wrong, then build an index where it is right.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: The single most testable mapping concept. A plain `object` array is **flattened** at index time: `products.manufacturer: [A, B]` and `products.price: [10, 200]` lose all knowledge of which price went with which manufacturer. `nested` indexes each object as a hidden separate document, preserving the pairing.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/nested.html

**The wrong answer, on the sample index as shipped:**

```json
GET kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        { "term":  { "products.manufacturer.keyword": "Oceanavigations" } },
        { "range": { "products.price": { "gte": 100 } } }
      ]
    }
  }
}
```

**Build a nested copy and ask again:**

```json
PUT ecommerce-nested
{
  "mappings": {
    "properties": {
      "order_id":           { "type": "keyword" },
      "customer_gender":    { "type": "keyword" },
      "taxful_total_price": { "type": "double" },
      "products": {
        "type": "nested",
        "properties": {
          "product_name": { "type": "text", "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } } },
          "price":        { "type": "double" },
          "quantity":     { "type": "integer" },
          "manufacturer": { "type": "keyword" },
          "category":     { "type": "keyword" }
        }
      }
    }
  }
}
```

```json
POST _reindex?refresh
{
  "source": {
    "index": "kibana_sample_data_ecommerce",
    "_source": [ "order_id", "customer_gender", "taxful_total_price",
                 "products.product_name", "products.price",
                 "products.quantity", "products.manufacturer", "products.category" ]
  },
  "dest": { "index": "ecommerce-nested" }
}
```

```json
GET ecommerce-nested/_search
{
  "size": 0,
  "query": {
    "nested": {
      "path": "products",
      "query": {
        "bool": {
          "must": [
            { "term":  { "products.manufacturer": "Oceanavigations" } },
            { "range": { "products.price": { "gte": 100 } } }
          ]
        }
      }
    }
  }
}
```

> Answer: the flat query returns **93** orders. The nested query returns **40**. The other 53 are
> false positives — orders that contain *an* Oceanavigations item **and** *a* £100+ item, but not
> the same item.
>
> :bulb: Aggregating nested data needs the matching `nested` agg wrapper:
> ```json
> "aggs": { "items": { "nested": { "path": "products" },
>           "aggs": { "by_mfr": { "terms": { "field": "products.manufacturer" } } } } }
> ```
> Its `doc_count` is **10,087 items**, not 4,675 orders — nested aggs count sub-documents.

</details>
<hr>

:question: **Q34.** You need `attrs` to hold arbitrary user-supplied key/value pairs without creating a new mapping entry for every key. What type do you use, and how do you then query one key?

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `flattened` — the whole object becomes **one** field. It stops mapping explosion dead, at the cost of: everything is treated as a `keyword`, no numeric ranges, no analysis.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/flattened.html

```json
PUT attrs-demo
{ "mappings": { "properties": { "attrs": { "type": "flattened" } } } }
```

```json
POST attrs-demo/_doc/1?refresh
{ "attrs": { "colour": "blue", "size": "L" } }
```

```json
GET attrs-demo/_search
{ "query": { "term": { "attrs.colour": "blue" } } }
```

> Answer: 1 hit. `GET attrs-demo/_mapping` shows **only** `attrs` — `colour` and `size` never became
> mapping entries, which is the whole point.
>
> :bulb: Contrast with the other "many fields" answers you might be asked for:
> `nested` (preserve object relationships) · `object` with `"enabled": false` (store in `_source`,
> index nothing) · `"dynamic": false` (store and keep in `_source`, do not add to the mapping).

</details>
<hr>

:question: **Q35.** An application still sends `item_code`, but the index now calls the field `sku`. Make both names work **without reindexing and without duplicating data**.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: The `alias` field type. Purely a mapping-level pointer — zero storage, resolves at query time.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/field-alias.html

```json
PUT products/_mapping
{
  "properties": {
    "item_code": { "type": "alias", "path": "sku" }
  }
}
```

> Answer: `{"query":{"term":{"item_code":"AB-1"}}}` now returns the same hit as querying `sku`.
>
> :warning: Do not confuse the three "alias" things the exam can ask for:
> - **field alias** (`"type": "alias"`) — another name for a field. This question.
> - **index alias** (`POST _aliases`) — another name for one or more indices. Q68–Q70.
> - **`copy_to`** — actually duplicates the value into a second field at index time. Q28.
>
> A field alias can be used in queries, aggregations and sorting, but **not** in `_source` (docs
> still come back with `sku`), and you cannot index into it.

</details>
<hr>

# Part 4 — Ingest pipelines

> **Objective 8.** One objective, but a wide one, and it is nearly always paired with a reindex
> (Part 5) or a data stream (Part 6). Learn `_simulate` first — it is how you get these right
> without burning attempts on real data.

:bulb: `POST /_ingest/pipeline/_simulate` takes the **pipeline definition inline** and never touches
an index. Add `?verbose` to see the document after **every** processor — the fastest debugging tool
in the exam.

:bulb: Field access, by context:

| In a… | Use | Example |
| --- | --- | --- |
| `script` processor | `ctx.field` | `ctx.total = ctx.price * ctx.qty` |
| processor `if` condition | `ctx.field` (Painless) | `"if": "ctx.status >= 400"` |
| processor **value** string | Mustache `{{{field}}}` | `"value": "{{{host}}}-prod"` |
| `foreach` inner processor | `_ingest._value` | `{"trim": {"field": "_ingest._value"}}` |

<hr>

:question: **Q36.** Write a pipeline that parses these raw web-log lines into `client.ip`, `http.request.method`, `url.path`, `http.response.status_code` (a number) and `http.response.bytes`; tags any 4xx/5xx with `event.outcome: failure`; drops the raw `message`; and, for lines it cannot parse, tags them `event.kind: parse_failure` **instead of failing the document**. Prove it with `_simulate`.

```
192.168.1.10 GET /cart/checkout 503 4021
this line is not a log line
```

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Grok is the default answer whenever the input is an unstructured string. `on_failure` **on the grok processor** contains the fallback; without it a bad line aborts the whole document.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/grok-processor.html

```json
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "description": "parse raw web log lines",
    "processors": [
      {
        "grok": {
          "field": "message",
          "patterns": [
            "%{IP:client.ip} %{WORD:http.request.method} %{URIPATHPARAM:url.path} %{NUMBER:http.response.status_code:int} %{NUMBER:http.response.bytes:long}"
          ],
          "on_failure": [
            { "set": { "field": "event.kind", "value": "parse_failure" } }
          ]
        }
      },
      {
        "set": {
          "field": "event.outcome",
          "value": "failure",
          "if": "ctx.http?.response?.status_code != null && ctx.http.response.status_code >= 400"
        }
      },
      { "remove": { "field": "message", "ignore_missing": true } }
    ]
  },
  "docs": [
    { "_source": { "message": "192.168.1.10 GET /cart/checkout 503 4021" } },
    { "_source": { "message": "this line is not a log line" } }
  ]
}
```

> Answer: doc 1 →
> `{"client":{"ip":"192.168.1.10"},"http":{"request":{"method":"GET"},"response":{"bytes":4021,"status_code":503}},"url":{"path":"/cart/checkout"},"event":{"outcome":"failure"}}`
> · doc 2 → `{"event":{"kind":"parse_failure"}}`.
>
> :warning: :fire: **The `?.` trap.** Writing the condition as
> `"if": "ctx.http?.response?.status_code >= 400"` looks safe but **throws** on doc 2:
> `Cannot invoke "Object.getClass()" because "leftObject" is null`.
> Safe navigation stops the *traversal* from NPE-ing; it still hands `null` to the `>=` comparison.
> You must null-check explicitly, as above.
>
> :bulb: Type conversion inside grok — `%{NUMBER:field:int}` and `:long`, `:float`, `:double`, `:boolean` —
> saves you a separate `convert` processor. `%{IP:client.ip}` with a dot in the name creates the
> nested object automatically.

</details>
<hr>

:question: **Q37.** The same data now arrives in a strictly fixed format. Parse it **without a regex**, promote the timestamp to `@timestamp`, lowercase the level, add the message length, and remove the working fields.

```
2026-03-01T10:15:00+0000 ERROR [checkout] payment gateway timeout
```

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `dissect` when the delimiters are fixed — it is much faster than grok because there is no backtracking. Grok when the format varies.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/dissect-processor.html

```json
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      { "dissect": { "field": "raw", "pattern": "%{ts} %{level} [%{service}] %{msg}" } },
      { "date": { "field": "ts", "target_field": "@timestamp", "formats": [ "yyyy-MM-dd'T'HH:mm:ssZ" ] } },
      { "lowercase": { "field": "level" } },
      { "script": { "source": "ctx.msg_length = ctx.msg.length()" } },
      { "remove": { "field": [ "raw", "ts" ] } }
    ]
  },
  "docs": [
    { "_source": { "raw": "2026-03-01T10:15:00+0000 ERROR [checkout] payment gateway timeout" } }
  ]
}
```

> Answer:
> `{"@timestamp":"2026-03-01T10:15:00.000Z","level":"error","service":"checkout","msg":"payment gateway timeout","msg_length":23}`
>
> :bulb: Useful dissect modifiers: `%{+field}` appends to a previous capture, `%{?skip}` discards,
> `%{field->}` allows repeated padding delimiters (a common fix for column-aligned logs).

</details>
<hr>

:question: **Q38.** Instead of tagging bad documents inline, capture **any** failure anywhere in the pipeline into `error.message` and `error.processor`, keeping the original document.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: **Pipeline-level** `on_failure` — a sibling of `processors`, not inside one. It catches failures from every processor, and gives you three metadata variables.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/handling-failure-in-pipelines.html

```json
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      { "grok": { "field": "message", "patterns": [ "%{IP:client_ip} %{WORD:method}" ] } }
    ],
    "on_failure": [
      { "set": { "field": "error.message",   "value": "{{{ _ingest.on_failure_message }}}" } },
      { "set": { "field": "error.processor", "value": "{{{ _ingest.on_failure_processor_type }}}" } }
    ]
  },
  "docs": [ { "_source": { "message": "garbage" } } ]
}
```

> Answer:
> `{"message":"garbage","error":{"message":"Provided Grok expressions do not match field value: [garbage]","processor":"grok"}}`
>
> :bulb: The three variables: `_ingest.on_failure_message`, `_ingest.on_failure_processor_type`,
> `_ingest.on_failure_processor_tag` (the `tag` you gave the processor — set one if you have several
> of the same type).
>
> :bulb: The three levels of error handling, in order of how targeted they are:
> `ignore_missing` / `ignore_failure` on one processor → `on_failure` on one processor →
> `on_failure` on the whole pipeline.

</details>
<hr>

:question: **Q39.** A CSV feed arrives as one string per document, with a `meta` field of `key=value` pairs. Split it into fields, coerce `amount` to a number, flatten `meta` into `meta_*` fields, tag every document `imported` and `csv`, parse the user agent, and **discard** any row whose country is `XX`.

```json
{ "row": "1001, Ada Lovelace , 249.99, GB", "meta": "channel=web;device=mobile", "ua": "Mozilla/5.0 … Chrome/120.0 …" }
{ "row": "1002, Test User, 0, XX" }
```

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Six processors that come up constantly. `drop` is the one people forget exists — it removes the document silently rather than failing it.

```json
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      { "csv": { "field": "row", "target_fields": [ "order_id", "customer", "amount", "country" ], "separator": ",", "trim": true } },
      { "convert": { "field": "amount", "type": "double" } },
      { "kv": { "field": "meta", "field_split": ";", "value_split": "=", "prefix": "meta_", "ignore_missing": true } },
      { "append": { "field": "tags", "value": [ "imported", "csv" ] } },
      { "drop": { "if": "ctx.country == 'XX'" } },
      { "user_agent": { "field": "ua", "ignore_missing": true } },
      { "remove": { "field": [ "row", "meta" ], "ignore_missing": true } }
    ]
  },
  "docs": [
    { "_source": { "row": "1001, Ada Lovelace , 249.99, GB", "meta": "channel=web;device=mobile", "ua": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36" } },
    { "_source": { "row": "1002, Test User, 0, XX" } }
  ]
}
```

> Answer: doc 1 gains `order_id: "1001"`, `customer: "Ada Lovelace"` (note `trim`), `amount: 249.99`,
> `country: "GB"`, `meta_channel: "web"`, `meta_device: "mobile"`, `tags: ["imported","csv"]` and a
> full `user_agent` object (`name: Chrome`, `version: 120.0`, `os.name: Mac OS X`, `device.name: Mac`).
> Doc 2 comes back as **`null`** in the `docs` array — that is what a dropped document looks like.

</details>
<hr>

:question: **Q40.** Normalise a messy tags string `" Alpha , BETA ,Gamma"` into the array `["alpha","beta","gamma"]`, and strip everything but digits out of `"ZO-0299-XT"`.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `foreach` applies an inner processor to every element of an array. Inside it, the current element is `_ingest._value`.

```json
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      { "split": { "field": "tags", "separator": ",", "target_field": "tag_list" } },
      { "foreach": { "field": "tag_list", "processor": { "trim": { "field": "_ingest._value" } } } },
      { "foreach": { "field": "tag_list", "processor": { "lowercase": { "field": "_ingest._value" } } } },
      { "gsub": { "field": "sku", "pattern": "[^0-9]", "replacement": "" } }
    ]
  },
  "docs": [ { "_source": { "tags": " Alpha , BETA ,Gamma", "sku": "ZO-0299-XT" } } ]
}
```

> Answer: `tag_list: ["alpha","beta","gamma"]`, `sku: "0299"`.
>
> :bulb: `split` takes a **regex** as its separator, so `"separator": "."` splits on every character.
> Escape it as `"\\."` if you mean a literal dot.

</details>
<hr>

:question: **Q41.** Add an `alliance` field to incoming flight documents by looking the carrier up in a reference index. Unknown carriers must pass through untouched.

```json
{"Carrier":"Kibana Airlines","alliance":"Visual","hq_country":"NL"}
{"Carrier":"Logstash Airways","alliance":"Pipeline","hq_country":"US"}
{"Carrier":"ES-Air","alliance":"Core","hq_country":"US"}
{"Carrier":"JetBeats","alliance":"Pipeline","hq_country":"JP"}
```

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: The `enrich` processor. Three steps, and **people forget step 3**: a policy does nothing until it is `_execute`d, which materialises a read-only system index. Change the source data → re-execute the policy.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ingest-enriching-data.html

```json
PUT carriers
{ "mappings": { "properties": {
    "Carrier":    { "type": "keyword" },
    "alliance":   { "type": "keyword" },
    "hq_country": { "type": "keyword" } } } }
```

```json
POST carriers/_bulk?refresh
{"index":{}}
{"Carrier":"Kibana Airlines","alliance":"Visual","hq_country":"NL"}
{"index":{}}
{"Carrier":"Logstash Airways","alliance":"Pipeline","hq_country":"US"}
{"index":{}}
{"Carrier":"ES-Air","alliance":"Core","hq_country":"US"}
{"index":{}}
{"Carrier":"JetBeats","alliance":"Pipeline","hq_country":"JP"}
```

```json
PUT _enrich/policy/carrier-policy
{
  "match": {
    "indices": "carriers",
    "match_field": "Carrier",
    "enrich_fields": [ "alliance", "hq_country" ]
  }
}
```

```json
POST _enrich/policy/carrier-policy/_execute
```

```json
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "processors": [
      { "enrich": { "policy_name": "carrier-policy", "field": "Carrier", "target_field": "carrier_info", "max_matches": 1 } },
      { "set": { "field": "alliance", "copy_from": "carrier_info.alliance", "ignore_empty_value": true } },
      { "remove": { "field": "carrier_info", "ignore_missing": true } }
    ]
  },
  "docs": [
    { "_source": { "Carrier": "JetBeats", "FlightNum": "ABC123" } },
    { "_source": { "Carrier": "Unknown Air" } }
  ]
}
```

> Answer: doc 1 → `{"Carrier":"JetBeats","FlightNum":"ABC123","alliance":"Pipeline"}`.
> doc 2 → `{"Carrier":"Unknown Air"}`, unchanged — no match, and `ignore_empty_value` stopped the
> `set` from writing a null.
>
> :bulb: `POST _enrich/policy/<name>/_execute` returns `{"status":{"phase":"COMPLETE"}}`. The three
> policy types are `match` (exact term), `range` (numeric/date/IP range lookup — think GeoIP ranges)
> and `geo_match`.

</details>
<hr>

:question: **Q42.** Every index in your cluster must stamp `ecs.version` and `event.ingested` on ingest, without repeating those two processors in each pipeline. Then make a `logs-app` index run that shared work **plus** its own, and guarantee an audit field is written even when a caller passes their own pipeline.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Three separate mechanisms, and the exam likes to test that you know which is which:
- **`pipeline` processor** — call another pipeline from inside this one.
- **`index.default_pipeline`** — used when the request does not name a pipeline. An explicit `?pipeline=` **overrides** it.
- **`index.final_pipeline`** — always runs last, and **cannot** be overridden. This is the audit hook.

```json
PUT _ingest/pipeline/common-fields
{
  "processors": [
    { "set": { "field": "ecs.version", "value": "8.11.0" } },
    { "set": { "field": "event.ingested", "value": "{{{_ingest.timestamp}}}" } }
  ]
}
```

```json
PUT _ingest/pipeline/app-pipeline
{
  "processors": [
    { "pipeline": { "name": "common-fields" } },
    { "set": { "field": "host.name", "copy_from": "hostname", "ignore_empty_value": true } },
    { "remove": { "field": "hostname", "ignore_missing": true } }
  ]
}
```

```json
PUT _ingest/pipeline/audit
{ "processors": [ { "set": { "field": "finalised", "value": true } } ] }
```

```json
PUT logs-app
{
  "settings": {
    "index.default_pipeline": "app-pipeline",
    "index.final_pipeline": "audit"
  }
}
```

> Answer: indexing `{"hostname":"web-01"}` with no `?pipeline=` yields
> `{"host":{"name":"web-01"},"ecs":{"version":"8.11.0"},"event":{"ingested":"…"},"finalised":true}`.
> Indexing with `?pipeline=something-else` replaces `app-pipeline` but `finalised: true` is **still**
> written — that is the point of `final_pipeline`.
>
> :warning: `final_pipeline` cannot change `_index`, so a `date_index_name` or `reroute` processor
> belongs in the default pipeline, not the final one.

</details>
<hr>

:question: **Q43.** You wrote a five-processor pipeline and one document comes out wrong. How do you see the document **as it stood after each step**?

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `?verbose`. Learn this before the exam, not during it.

```json
POST _ingest/pipeline/_simulate?verbose
{
  "pipeline": {
    "processors": [
      { "set": { "field": "env", "value": "prod" } },
      { "uppercase": { "field": "env" } },
      { "fail": { "if": "ctx.amount == null", "message": "amount is required" } }
    ]
  },
  "docs": [ { "_source": { "amount": 10 } } ]
}
```

> Answer: a `processor_results` array — `set` → `success`, doc now `{"env":"prod","amount":10}`;
> `uppercase` → `success`, doc now `{"env":"PROD",…}`; `fail` → **`skipped`**, because its `if`
> evaluated false.
>
> :bulb: `status` values worth recognising: `success`, `skipped` (the `if` was false), `error`,
> `error_ignored` (`ignore_failure: true`), `dropped`.
>
> :bulb: To simulate a pipeline that is **already stored**, put its name in the path instead of the
> body: `POST /_ingest/pipeline/app-pipeline/_simulate {"docs":[…]}`.

</details>
<hr>

# Part 5 — Reindex and Update By Query

> **Objective 7.** The standard "the data is already wrong, fix it" task. It nearly always arrives
> chained to a mapping change (Part 3) or a pipeline (Part 4).

:bulb: Which API?

| Situation | API |
| --- | --- |
| The **mapping** must change (type, analyzer, `nested`, shard count) | `_reindex` into a new index |
| The mapping is fine, the **documents** are wrong | `_update_by_query` in place |
| A new **multi-field / sub-field** was added and old docs must pick it up | `_update_by_query` with **no body** |
| Moving data between **clusters** | `_reindex` with `source.remote` |

<hr>

:question: **Q44.** Copy every **female** customer's order from the eCommerce index into a new index, prefixing each `order_id` with `ORD-`, adding a `price_band` of `low` / `mid` / `high`, and routing each document into a **monthly** index named `orders-YYYY-MM-DD`. Keep only a handful of fields.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Three separate mechanisms working together, which is exactly how the exam layers this:
`source.query` filters · `source._source` trims fields · the reindex `script` rewrites values ·
`dest.pipeline` does everything an ingest pipeline can, including `date_index_name` to pick the target index.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/docs-reindex.html

```json
PUT _ingest/pipeline/order-enrich
{
  "processors": [
    { "script": { "source": "ctx.price_band = ctx.taxful_total_price < 50 ? 'low' : (ctx.taxful_total_price < 150 ? 'mid' : 'high')" } },
    { "date_index_name": {
        "field": "order_date",
        "index_name_prefix": "orders-",
        "date_rounding": "M",
        "date_formats": [ "ISO8601" ] } }
  ]
}
```

```json
POST _reindex?refresh
{
  "source": {
    "index": "kibana_sample_data_ecommerce",
    "query": { "term": { "customer_gender": "FEMALE" } },
    "_source": [ "order_id", "order_date", "customer_gender", "taxful_total_price", "category" ]
  },
  "dest": { "index": "orders", "pipeline": "order-enrich" },
  "script": { "source": "ctx._source.order_id = 'ORD-' + ctx._source.order_id" }
}
```

> Answer: 2,433 documents created, spread across `orders-YYYY-MM-01` indices (the `dest.index` value
> is overridden per-document by `date_index_name`). `price_band` comes out mid 1,503 · low 820 · high 110.
>
> :bulb: In a **reindex script** you use `ctx._source.field`, and you can also set `ctx._id`,
> `ctx._index`, `ctx._routing` and `ctx.op` (`index` / `noop` / `delete`). In an **ingest script
> processor** it is `ctx.field` — no `_source`. Mixing them up is a classic five-minute loss.

</details>
<hr>

:question: **Q45.** :fire: You added a `title.keyword` sub-field to an existing index. Aggregating on it returns nothing. Why, and what is the fix?

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Mappings describe how documents are **indexed**, and indexing already happened. A new sub-field exists in the mapping but has no entries in its inverted index until each document is re-indexed. `_update_by_query` **with no body** rewrites every document in place, which re-runs analysis.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/docs-update-by-query.html

```json
PUT articles/_mapping
{
  "properties": {
    "title": {
      "type": "text",
      "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } }
    }
  }
}
```

```json
POST articles/_update_by_query?refresh
```

> Answer: before the `_update_by_query`, a `terms` aggregation on `title.keyword` returns
> `"buckets": []`. Afterwards (`{"total":3,"updated":3}`) every title appears.
>
> :warning: This is a **whole-index rewrite** — fine on exam-sized data, but on a real index add
> `?requests_per_second=1000` to throttle, and `?wait_for_completion=false` so you get a task back
> rather than a timeout.

</details>
<hr>

:question: **Q46.** In one `_update_by_query`, move every `draft` document to `review` and stamp a review date on it — except drafts with zero views, which should be **deleted**.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `ctx.op = 'delete'` inside an update-by-query script. Also note `params` — put constants there rather than concatenating them into the script source, so the script compiles once and is cached.

```json
POST articles/_update_by_query?refresh
{
  "query": { "term": { "status": "draft" } },
  "script": {
    "source": """
      if (ctx._source.views == 0) {
        ctx.op = 'delete';
      } else {
        ctx._source.status = 'review';
        ctx._source.reviewed_at = params.now;
      }
    """,
    "params": { "now": "2026-08-25" }
  }
}
```

> Answer: `{"total":2,"updated":1,"deleted":1,"noops":0}` — one draft promoted to `review` with a
> `reviewed_at`, one zero-view draft removed. The `published` document is untouched because the
> query never selected it.
>
> :bulb: `ctx.op` accepts `index` (default), `noop` (counts under `noops`, no write) and `delete`.

</details>
<hr>

:question: **Q47.** :fire: A reindex into an index that already holds some of the documents fails part-way with version conflicts. Make it (a) fail loudly on conflicts, then (b) skip the conflicts and copy the rest.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `dest.op_type: "create"` means "only write if this `_id` does not already exist". Any collision is a version conflict, which **aborts** the whole reindex by default.

```json
POST _reindex?refresh
{
  "source": { "index": "articles" },
  "dest":   { "index": "articles-archive", "op_type": "create" }
}
```

```json
POST _reindex?refresh
{
  "conflicts": "proceed",
  "source": { "index": "articles" },
  "dest":   { "index": "articles-archive", "op_type": "create" }
}
```

> Answer: the **first** run into an empty destination gives `{"total":2,"created":2,"version_conflicts":0}`.
> Run it again and it **aborts**, reporting the failures. With `"conflicts": "proceed"` the second run
> completes cleanly: `{"total":2,"created":0,"version_conflicts":2}` — nothing written, nothing failed.
>
> :bulb: `conflicts` is a **body** parameter on `_reindex` and a **query-string** parameter on
> `_update_by_query` and `_delete_by_query` (`?conflicts=proceed`). Both spellings exist; check
> which API you are on.

</details>
<hr>

:question: **Q48.** The reindex you need to run will take several minutes and the console times out. Run it in the background, then check on it, then cancel it.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `wait_for_completion=false` on `_reindex`, `_update_by_query` and `_delete_by_query` returns a **task id** immediately. Everything after that is the Task Management API.

```json
POST _reindex?wait_for_completion=false
{
  "source": { "index": "kibana_sample_data_flights" },
  "dest":   { "index": "flights-copy" }
}
```

```json
GET _tasks/<task_id>
GET _tasks?actions=*reindex&detailed
POST _tasks/<task_id>/_cancel
```

> Answer: the submit returns `{"task":"l9IFSEfFRbKkGWVRBDr_uA:42190"}` — `<nodeId>:<taskNumber>`.
> `GET _tasks/<id>` shows `completed` plus a `status` object with `total`, `created`, `updated` and
> `version_conflicts`. Results are also written to the `.tasks` index, so
> `GET .tasks/_doc/<task_id>` works after the fact.
>
> :bulb: On a multi-shard index add `?slices=auto` to parallelise. Combined with
> `wait_for_completion=false` that is the standard "reindex a large index" answer.

</details>
<hr>

:question: **Q49.** Copy an index from a **different** cluster into this one, taking only documents from the last 30 days.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `source.remote`. The remote host must be listed in `reindex.remote.whitelist` in `elasticsearch.yml` on **every node of the destination cluster** — and that is a static setting, so it needs a restart. Expect the exam cluster to have it configured already.

```yaml
# elasticsearch.yml on the destination cluster
reindex.remote.whitelist: "oldcluster:9200, 10.0.0.*:9200"
```

```json
POST _reindex?wait_for_completion=false
{
  "source": {
    "remote": {
      "host": "http://oldcluster:9200",
      "username": "elastic",
      "password": "changeme",
      "socket_timeout": "1m",
      "connect_timeout": "30s"
    },
    "index": "legacy-events",
    "query": { "range": { "@timestamp": { "gte": "now-30d/d" } } },
    "size": 1000
  },
  "dest": { "index": "events" }
}
```

> Answer: a task id. Verify with `GET _tasks/<id>` and then `GET events/_count`.
>
> :warning: Reindex-from-remote does **not** copy mappings or settings — create the destination index
> first if it needs anything other than dynamic mapping. This is true of ordinary `_reindex` too.

</details>
<hr>

# Part 6 — Index templates, data streams and ILM

> **Objectives 3 & 4.** In practice these arrive as **one multi-step task**: component templates →
> index template with `data_stream` → ILM policy → index a document → prove it worked.

:bulb: The order you must build them in, and why: an index template is only applied at index
**creation** time. Create the policy and the component templates **first**, then the index template,
then write the first document. Build them the other way round and your data stream comes up with the
wrong settings and you have to delete and start again.

<hr>

:question: **Q50.** Build a complete logging pipeline for `ece-logs-*`: shared field definitions in one reusable block, shared settings in another, an index template that combines them into a **data stream**, and a hot/warm/cold/delete ILM policy. Then prove a document lands correctly.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: The full sequence. Note `data_stream: {}` — an empty object is what turns an index template into a data-stream template.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/set-up-a-data-stream.html

**1 — the ILM policy**

```json
PUT _ilm/policy/ece-logs-policy
{
  "policy": {
    "_meta": { "description": "30d hot, 90d total" },
    "phases": {
      "hot": {
        "actions": {
          "rollover": { "max_primary_shard_size": "50gb", "max_age": "7d" },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": { "max_num_segments": 1 },
          "shrink": { "number_of_shards": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": { "number_of_replicas": 0 },
          "set_priority": { "priority": 0 }
        }
      },
      "delete": { "min_age": "90d", "actions": { "delete": {} } }
    }
  }
}
```

**2 — component templates**

```json
PUT _component_template/ece-logs-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date", "format": "strict_date_optional_time||epoch_millis" },
        "message":    { "type": "text" },
        "host.name":  { "type": "keyword" },
        "client.ip":  { "type": "ip" },
        "http.response.status_code": { "type": "short" }
      }
    }
  },
  "_meta": { "description": "shared ECS-ish field definitions" }
}
```

```json
PUT _component_template/ece-logs-settings
{
  "template": {
    "settings": {
      "index.number_of_shards": 1,
      "index.number_of_replicas": 0,
      "index.lifecycle.name": "ece-logs-policy"
    }
  }
}
```

**3 — the index template**

```json
PUT _index_template/ece-logs-template
{
  "index_patterns": [ "ece-logs-*" ],
  "data_stream": { },
  "composed_of": [ "ece-logs-mappings", "ece-logs-settings" ],
  "priority": 500,
  "_meta": { "description": "app logs data stream" }
}
```

**4 — check before you commit**

```json
POST _index_template/_simulate_index/ece-logs-app
```

**5 — create the stream by writing to it**

```json
POST ece-logs-app/_doc
{ "@timestamp": "2026-08-25T10:00:00Z", "message": "hello", "host.name": "web-1",
  "client.ip": "10.0.0.5", "http.response.status_code": 200 }
```

> Answer: `GET _data_stream/ece-logs-app` shows
> `generation: 1`, `template: "ece-logs-template"`, `ilm_policy: "ece-logs-policy"` and one backing
> index `.ds-ece-logs-app-2026.08.25-000001`.
>
> :bulb: `_simulate_index` is free and shows you the **merged** result plus an `overlapping` array
> listing any other template that also matches. Use it before every template question.
>
> :warning: **`min_age` is relative to index creation — *until* the index rolls over, after which
> it is relative to the rollover time.** And the ages are cumulative from that origin, not from the
> previous phase: `warm.min_age: 7d` + `cold.min_age: 30d` means cold at **30 days**, not 37.

</details>
<hr>

:question: **Q51.** Force the data stream to roll over now, and show its backing indices before and after.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
POST ece-logs-app/_rollover
```

```json
GET _data_stream/ece-logs-app
```

> Answer: `{"old_index":".ds-ece-logs-app-2026.08.25-000001","new_index":".ds-ece-logs-app-2026.08.25-000002","rolled_over":true}`,
> and `generation` becomes 2 with two backing indices listed.
>
> :bulb: You can attach conditions — `POST ece-logs-app/_rollover {"conditions":{"max_docs":1}}` —
> and it only rolls if they are met. `rolled_over: false` with a `conditions` block in the response
> means it evaluated but declined.

</details>
<hr>

:question: **Q52.** :fire: Your data stream exists but the index template you wrote is being ignored — the backing index has the wrong shard count. Diagnose it.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Four causes, in the order worth checking:

1. **A higher-priority template matched instead.** `POST _index_template/_simulate_index/<name>` shows the winner, and `overlapping` shows who else matched. Built-in templates like `logs`, `metrics` and `synthetics` ship at priority **100**, so anything you write for `logs-*` needs a higher `priority`.
2. **The template was created after the index.** Templates apply at creation only. Fix: roll over (new backing index picks it up) or delete and recreate.
3. **You edited a component template but not the index template.** Component template changes only reach **new** indices; existing ones are unaffected.
4. **The setting is not dynamic.** `number_of_shards`, `codec` and analysis settings can never change on a live index.

```json
POST _index_template/_simulate_index/ece-logs-app
GET _index_template/ece-logs-template
GET .ds-ece-logs-app-2026.08.25-000001/_settings
```

> Answer: compare `_simulate_index` output against the live index's real settings. If they disagree,
> the index predates the template — the fix on a data stream is `POST ece-logs-app/_rollover`,
> because the **new** backing index is created fresh under the current template.

</details>
<hr>

:question: **Q53.** Set up a rolling time-series index **without** using a data stream, as an older application requires a fixed index name to write to.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: The pre-data-stream pattern, still examinable. Three rules people get wrong: the index name **must** end in `-000001` (or any number ILM can increment), the alias needs `is_write_index: true`, and ILM needs `index.lifecycle.rollover_alias` pointing at it.

```json
PUT ece-ts-000001
{
  "aliases": { "ece-ts": { "is_write_index": true } },
  "settings": {
    "number_of_replicas": 0,
    "index.lifecycle.name": "ece-logs-policy",
    "index.lifecycle.rollover_alias": "ece-ts"
  }
}
```

```json
POST ece-ts/_rollover
{ "conditions": { "max_docs": 1 } }
```

> Answer: `{"old_index":"ece-ts-000001","new_index":"ece-ts-000002","rolled_over":true}`, and
> `GET _cat/aliases/ece-ts?v` shows the write flag has moved to `-000002`. The application keeps
> writing to `ece-ts` and never notices.
>
> :warning: Omit `rollover_alias` and ILM parks the index in a `step: ERROR` state with
> `setting [index.lifecycle.rollover_alias] for index [...] is empty or not defined`. Check with
> `GET ece-ts-000001/_ilm/explain`.

</details>
<hr>

:question: **Q54.** An index managed by ILM is not progressing. What do you check, and how do you push it along?

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `_ilm/explain` is the answer to every "why is ILM not doing anything" question.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/ilm-explain-lifecycle.html

```json
GET ece-logs-app/_ilm/explain
GET _ilm/status
```

> Answer: read three fields — `managed` (false means no policy is attached at all), `step` (`ERROR`
> means it is stuck) and `step_info` (the actual reason).
>
> The four fixes, in order:
> - `POST _ilm/start` — if `_ilm/status` says `STOPPED`.
> - `POST <index>/_ilm/retry` — re-runs the failed step after you have fixed its cause.
> - `POST <index>/_ilm/move/<step>` — force it to a specific phase/action/step.
> - `PUT <index>/_settings {"index.lifecycle.indexing_complete": true}` — tell ILM to stop expecting
>   a rollover on an index that will never get one.
>
> :bulb: ILM checks every **10 minutes** by default (`indices.lifecycle.poll_interval`). Nothing
> happening for a few minutes is normal, not broken. In a lab you can set it to `1s` to watch it work.

</details>
<hr>

:question: **Q55.** :fire: Name the operations that behave **differently** on a data stream from a normal index.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: A frequent source of lost marks, because the error messages are not obvious.

| Operation | On a data stream |
| --- | --- |
| `PUT ds/_doc/1` | **Rejected** — `only write ops with an op_type of create are allowed in data streams` |
| `POST ds/_doc` (auto id) | Works — it is a `create` |
| Update / delete a single doc by id | Not directly. Go to the **backing index**, or use `_update_by_query` / `_delete_by_query` |
| `PUT ds/_mapping`, `PUT ds/_settings` | Applies to backing indices; for future ones, change the **template** |
| Every document | **Must** have `@timestamp` |
| `GET ds/_search` | Works normally — searches all backing indices |
| Delete | `DELETE _data_stream/ds` (not `DELETE ds`) |

> Answer: try it — `PUT kibana_sample_data_logs/_doc/1 {"a":1}` on the sample logs (which **is** a
> data stream in 8.15) returns
> `only write ops with an op_type of create are allowed in data streams`.
>
> :bulb: `GET _data_stream/<name>` gives you `generation`, `template`, `ilm_policy` and the backing
> index list — the first thing to run on any data-stream question.

</details>
<hr>

# Part 7 — Query DSL: terms, phrases and Boolean combinations

> **Objectives 10 & 11.** Two objectives, and the front door to every search task in the exam.

:bulb: The single distinction to get right:

| | Analyzed at search time? | Use on | Contributes to `_score`? |
| --- | --- | --- | --- |
| `match` | **Yes** | `text` | Yes |
| `term` / `terms` | **No** — exact bytes | `keyword`, numbers, dates, booleans, IPs | Yes, but flat |
| Inside `filter` / `must_not` | either | either | **No** — and it is cacheable |

<hr>

:question: **Q56.** :fire: In `shakespeare`, run a `term` query on `text_entry` for `Romeo`. It returns nothing. Explain, and give two ways to make it work.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: **The most common self-inflicted wound in the exam.** `text_entry` is `text`, so it was analyzed at index time and the inverted index contains the lowercased token `romeo`. `term` does **not** analyze its input, so it looks up the literal bytes `Romeo` and finds nothing.

```json
GET shakespeare/_search
{ "query": { "term": { "text_entry": "Romeo" } } }
```

> Answer: **0 hits.**
>
> Two fixes:
> ```json
> GET shakespeare/_search
> { "query": { "term": { "text_entry": "romeo" } } }
> ```
> → **122 hits** (match the analyzed token yourself), or
> ```json
> GET shakespeare/_search
> { "query": { "match": { "text_entry": "Romeo" } } }
> ```
> → **122 hits** (let `match` analyze the input for you). The second is what the exam wants.
>
> :bulb: The mirror-image mistake: `term` on `speaker` for `KING HENRY V` returns **1,086** hits,
> because `speaker` is a `keyword` and was never analyzed. Whenever a query "should work" and
> returns zero, check the field's type in `GET idx/_mapping` before anything else.

</details>
<hr>

:question: **Q57.** Find the lines containing the exact phrase `to be`. Then allow up to two other words between them, and note what happens to the count.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `match_phrase` requires the tokens **adjacent and in order**. `slop` is how many position moves you will tolerate.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-match-query-phrase.html

```json
GET shakespeare/_search
{
  "size": 0,
  "query": {
    "match_phrase": { "text_entry": { "query": "to be", "slop": 2 } }
  }
}
```

> Answer: slop 0 → **928** · slop 1 → **946** · slop 2 → **1,039** · slop 3 → **1,335**.
>
> :bulb: `slop` also permits **reordering**: "be to" costs 2 slop. If a question says "in any order,
> within N words", `slop` is the answer, not `bool`.

</details>
<hr>

:question: **Q58.** Find flights that: go to Rome; cost 500 or more; are **not** cancelled; are **not** delayed; and originate in the US **or** the UK **or** land in sunny weather.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: One `bool` covering all four clause types. The mapping to English:
`must` = must match, scored · `filter` = must match, **not** scored (use for ranges, terms, dates) ·
`should` = optional, but `minimum_should_match` makes it required · `must_not` = must not match, not scored.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-bool-query.html

```json
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        { "match": { "DestCityName": "Rome" } }
      ],
      "filter": [
        { "range": { "AvgTicketPrice": { "gte": 500 } } },
        { "term":  { "Cancelled": false } }
      ],
      "should": [
        { "term": { "OriginCountry": "US" } },
        { "term": { "OriginCountry": "GB" } },
        { "term": { "DestWeather": "Sunny" } }
      ],
      "minimum_should_match": 1,
      "must_not": [
        { "term": { "FlightDelay": true } }
      ]
    }
  }
}
```

> Answer: **27** flights.
>
> :warning: :fire: **`minimum_should_match` defaults to 0** when there is at least one `must` or
> `filter` clause — so without that line, the `should` clauses become pure score boosts and the
> "or" requirement silently disappears (you get far more than 27). If the question says "and must
> also be one of…", you need `minimum_should_match`.

</details>
<hr>

:question: **Q59.** Search `text_entry`, `play_name` and `speaker` together for `king richard`, weighting the play title three times as heavily. Then restrict it to lines where those two words appear as a phrase.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `multi_match` and its `type` — the parameter that actually changes the answer.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-multi-match-query.html

```json
GET shakespeare/_search
{
  "size": 0,
  "query": {
    "multi_match": {
      "query": "king richard",
      "fields": [ "text_entry", "play_name^3", "speaker" ],
      "type": "best_fields"
    }
  }
}
```

> Answer: `best_fields`, `most_fields` and `cross_fields` all return **1,629**; `"type": "phrase"`
> returns **50**.
>
> The types, in the order you will need them:
> - **`best_fields`** (default) — score from the single best-matching field. "Find the one document that says this."
> - **`most_fields`** — sum the scores. Same text indexed several ways (`title` + `title.english`).
> - **`cross_fields`** — treat the fields as one big field. Names, addresses, split identifiers.
> - **`phrase`** / **`phrase_prefix`** — run `match_phrase` on each field.
>
> :warning: `"type": "phrase_prefix"` over a `keyword` field **errors**:
> `Can only use phrase prefix queries on text fields - not on [play_name] which is of type [keyword]`.
> Drop the keyword field from `fields` or pick a different type.

</details>
<hr>

:question: **Q60.** Handle a user typing `romio` when they meant `romeo`, then a user typing the partial phrase `not to b` expecting to complete it.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Two different tools. Fuzziness is edit distance; phrase-prefix is autocomplete.

```json
GET shakespeare/_search
{
  "size": 0,
  "query": {
    "match": { "text_entry": { "query": "romio", "fuzziness": "AUTO" } }
  }
}
```

```json
GET shakespeare/_search
{
  "size": 0,
  "query": {
    "match_phrase_prefix": {
      "text_entry": { "query": "not to b", "max_expansions": 1000 }
    }
  }
}
```

> Answer: the fuzzy match returns **178** (identical to `{"fuzzy": {"text_entry": {"value": "romio",
> "fuzziness": 1}}}`).
>
> :warning: :fire: The phrase-prefix query returns **0 hits at the default `max_expansions` of 50**,
> and **65 hits** at 1000. The last term is expanded against the terms dictionary in alphabetical
> order and then truncated — `be` simply falls outside the first 50 terms beginning with `b`. If a
> `match_phrase_prefix` returns nothing when you are certain it should match, raise `max_expansions`
> before you doubt the rest of the query.
>
> :bulb: `fuzziness: "AUTO"` means 0 edits for terms of 1–2 chars, 1 for 3–5, 2 for 6+. That is
> almost always what you want over a hard-coded number.

</details>
<hr>

:question: **Q61.** Find flights originating in **any** of GB, FR or DE — first with the list inline, then with the list read from a document in another index.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: A `terms` **query** is `term` with an OR of values. The `terms` **lookup** form fetches the values from a live document — useful for watchlists you want to update without editing queries.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/query-dsl-terms-query.html

```json
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "query": { "terms": { "OriginCountry": [ "GB", "FR", "DE" ] } }
}
```

```json
PUT watchlists/_doc/eu?refresh
{ "countries": [ "GB", "FR", "DE" ] }
```

```json
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "query": {
    "terms": {
      "OriginCountry": { "index": "watchlists", "id": "eu", "path": "countries" }
    }
  }
}
```

> Answer: **1,209** flights, both ways.

</details>
<hr>

:question: **Q62.** :fire: Count the Shakespeare lines from any play with `Henry` in its title. Explain why the count you get first is wrong.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Two lessons in one question.

```json
GET shakespeare/_search
{
  "size": 0,
  "query": { "wildcard": { "play_name": { "value": "*Henry*" } } }
}
```

> Answer: this reports `{"value": 10000, "relation": "gte"}` — **not** the real count. Since 7.0,
> `hits.total` stops counting at 10,000 to keep searches fast. `"relation": "gte"` is the tell.
>
> ```json
> GET shakespeare/_search
> {
>   "size": 0,
>   "track_total_hits": true,
>   "query": { "wildcard": { "play_name": { "value": "*Henry*" } } }
> }
> ```
> → **19,474**, `"relation": "eq"`.
>
> :warning: Whenever an exam question asks "how many documents…", check `relation`. If it says `gte`
> you have not answered the question. `track_total_hits` also accepts a number
> (`"track_total_hits": 50000`) if you only need accuracy up to a point.
>
> :bulb: A leading-wildcard query (`*Henry*`) scans every term in the dictionary and is very slow on
> real data. Prefer `prefix` (`GET shakespeare/_search {"query":{"prefix":{"speaker":"KING"}}}`
> → **7,250**) or index the field differently.

</details>
<hr>

:question: **Q63.** Find Shakespeare lines mentioning a crown together with either a king or a queen, using a **single query string** rather than a nested `bool`.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `query_string` supports the full Lucene syntax — `AND`, `OR`, `NOT`, parentheses, `field:value`, `~` for fuzzy, `^` for boosts. It **throws** on malformed input, which is why user-facing search should use `simple_query_string` instead (it silently ignores bad syntax).

```json
GET shakespeare/_search
{
  "size": 0,
  "query": {
    "query_string": {
      "query": "(king OR queen) AND crown",
      "default_field": "text_entry"
    }
  }
}
```

> Answer: **15** lines.
>
> :bulb: The `simple_query_string` equivalent uses `+` for AND, `|` for OR and `-` for NOT:
> `{"simple_query_string": {"query": "crown +(king | queen)", "fields": ["text_entry"]}}`.

</details>
<hr>

:question: **Q64.** Find web log entries that actually recorded a `phpmemory` value, and separately those from the last seven complete days.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `exists` matches documents where the field has a **non-null** indexed value. There is no `missing` query — use `must_not` + `exists`.

Date math: `now-7d/d` means "seven days ago, **rounded down** to the start of the day". The `/d` is what makes the window stable rather than sliding by the second.

```json
GET kibana_sample_data_logs/_search
{
  "size": 0,
  "query": { "exists": { "field": "phpmemory" } }
}
```

```json
GET kibana_sample_data_logs/_search
{
  "size": 0,
  "query": {
    "range": { "@timestamp": { "gte": "now-7d/d", "lt": "now/d" } }
  }
}
```

> Answer: **552** documents have a `phpmemory` value. The date-range count depends on when your
> sample data was generated — *shape only*.
>
> :bulb: "Documents missing a field":
> ```json
> { "query": { "bool": { "must_not": [ { "exists": { "field": "phpmemory" } } ] } } }
> ```
> A field is "missing" if it is absent, `null`, `[]`, or was rejected by `ignore_malformed`
> (see `_ignored` in Q28).

</details>
<hr>

# Part 8 — Sorting, pagination, aliases and async search

> **Objectives 12, 17, 18 & 19.** Individually small; usually bundled into one "search application"
> task. Cheap marks — do not lose them.

<hr>

:question: **Q65.** Return eCommerce orders sorted by gender ascending, then by the **most expensive item in the order** descending, then by order date descending with any missing dates last.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Three things at once: multi-key sort (evaluated left to right), `mode` for multi-valued fields, and `missing`.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/sort-search-results.html

```json
GET kibana_sample_data_ecommerce/_search
{
  "size": 3,
  "_source": [ "order_id", "customer_gender", "taxful_total_price" ],
  "sort": [
    { "customer_gender": { "order": "asc" } },
    { "products.price":  { "order": "desc", "mode": "max" } },
    { "order_date":      { "order": "desc", "missing": "_last" } }
  ]
}
```

> Answer: the first hit is a FEMALE order whose priciest item is 200.00, and each hit carries a
> `sort` array like `["FEMALE", 200.0, 1787653757000]` — the values that were actually sorted on.
>
> :bulb: `mode` for multi-valued fields: `min`, `max`, `sum`, `avg`, `median`. Without it, ES picks
> `min` for ascending and `max` for descending, which is often not what the question meant.
>
> :warning: Sorting on an unmapped field **errors** —
> `No mapping found for [nope] in order to sort on`. When searching several indices where the field
> may be absent, add `{"nope": {"order": "asc", "unmapped_type": "long"}}`.

</details>
<hr>

:question: **Q66.** Sort flights by **value for money** — ticket price per kilometre — highest first. There is no such field.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Two valid answers. `_script` sort is the classic; a runtime field is the more modern one and is reusable.

```json
GET kibana_sample_data_flights/_search
{
  "size": 2,
  "_source": [ "FlightNum", "AvgTicketPrice", "DistanceKilometers" ],
  "sort": {
    "_script": {
      "type": "number",
      "script": {
        "source": "doc['AvgTicketPrice'].value / Math.max(1, doc['DistanceKilometers'].value)"
      },
      "order": "desc"
    }
  }
}
```

> Answer: the top hits are flights with `DistanceKilometers: 0` — hence the `Math.max(1, …)` guard
> against divide-by-zero. Add a `filter` on `DistanceKilometers > 0` if the question implies real
> flights only.
>
> :bulb: The runtime-field version (Q23) is usually the better answer, because you can then also
> **filter** and **aggregate** on the same expression, and script sort cannot be reused.

</details>
<hr>

:question: **Q67.** :fire: Page through all 13,014 flights, 10 at a time. Show why `from`/`size` cannot finish the job, and give the approach that can.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `from`/`size` is fine for a UI showing page 1–10 and nothing else. Deep paging is `search_after` with a point in time.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/paginate-search-results.html

**Where it breaks:**

```json
GET kibana_sample_data_flights/_search
{ "from": 10000, "size": 10, "query": { "match_all": {} } }
```
> `Result window is too large, from + size must be less than or equal to: [10000] but was [10010].`

**The right way — open a PIT, then page with `search_after`:**

```json
POST kibana_sample_data_flights/_pit?keep_alive=2m
```

```json
GET _search
{
  "size": 2,
  "pit": { "id": "<pit_id>", "keep_alive": "2m" },
  "sort": [ { "AvgTicketPrice": "desc" }, { "_shard_doc": "asc" } ],
  "_source": [ "FlightNum" ]
}
```

Take the `sort` array from the **last** hit and feed it back:

```json
GET _search
{
  "size": 2,
  "pit": { "id": "<pit_id>", "keep_alive": "2m" },
  "sort": [ { "AvgTicketPrice": "desc" }, { "_shard_doc": "asc" } ],
  "search_after": [ 1199.6428, 8298 ],
  "_source": [ "FlightNum" ]
}
```

```json
DELETE _pit
{ "id": "<pit_id>" }
```

> Answer: page 1 gives `C2YBQ05` (sort `[1199.729, 1838]`) and `LVK6HFM` (`[1199.6428, 8298]`);
> feeding the second sort array back gives `CVXI3Y9` then `XG02BA9` — continuous, no overlap.
>
> :warning: Four things that trip people up:
> - With a `pit`, **do not** put the index in the URL — it is `GET _search`, and the PIT carries the index.
> - Add a **tiebreaker** to the sort. With a PIT use `_shard_doc`; without one, use `_id` or any unique field. Without it, documents with equal sort values can repeat or be skipped.
> - `search_after` takes the **whole** sort array, in order.
> - The PIT freezes the data at open time, which is what makes the paging consistent. Delete it when done.
>
> :bulb: `scroll` still works but is deprecated for this purpose in 8.x — if the question says
> "consistent pagination", answer PIT + `search_after`.

</details>
<hr>

:question: **Q68.** Create an alias `womens-orders` over the eCommerce index that only ever exposes women's clothing orders.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: A **filtered alias** — the filter is applied to every search through the alias, invisibly.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/aliases.html

```json
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "kibana_sample_data_ecommerce",
        "alias": "womens-orders",
        "filter": { "term": { "category.keyword": "Women's Clothing" } }
      }
    }
  ]
}
```

> Answer: `GET womens-orders/_count` → **1,903**, versus 4,675 on the underlying index.
> `GET _cat/aliases/womens-orders?v` shows a `*` in the `filter` column.
>
> :bulb: You can also set `routing` / `search_routing` / `index_routing` on an alias, and
> `is_hidden` to keep it out of wildcards.

</details>
<hr>

:question: **Q69.** :fire: `app-search` currently points at `products-v1`. You have built `products-v2`. Cut over with **no window** in which the alias points at nothing or at both.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: **Every action inside one `_aliases` call is atomic.** Doing it as two calls (`remove`, then `add`) leaves a gap where searches fail. This exact scenario is a favourite.

```json
POST _aliases
{
  "actions": [
    { "remove": { "index": "products-v1", "alias": "app-search" } },
    { "add":    { "index": "products-v2", "alias": "app-search" } }
  ]
}
```

> Answer: `{"acknowledged": true, "errors": false}` and the alias moves in a single cluster-state
> update. Verify with `GET _cat/aliases/app-search?v`.
>
> :bulb: `"remove_index"` is also an alias action — it **deletes** the old index in the same atomic
> operation. Powerful, and irreversible; only use it if the question actually asks for it.

</details>
<hr>

:question: **Q70.** An alias spans several monthly indices. Make writes through it go to the current month only.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: An alias pointing at more than one index is **read-only** by default — indexing through it fails with `no write index is defined for alias`. Exactly one member must be flagged `is_write_index`.

```json
POST _aliases
{
  "actions": [
    { "add": { "index": "orders-2026-08-01", "alias": "orders-all" } },
    { "add": { "index": "orders-2026-09-01", "alias": "orders-all", "is_write_index": true } }
  ]
}
```

> Answer: `GET _cat/aliases/orders-all?v&h=alias,index,is_write_index` shows `true` against
> `orders-2026-09-01` and `false` against the other. Searches through `orders-all` still hit both.
>
> :bulb: This is precisely the machinery a data stream automates for you (Q50) — worth being able to
> say so if a question asks why you would use one over the other.

</details>
<hr>

:question: **Q71.** Run a long aggregation across every `kibana_sample_data_*` index without blocking, then collect the result and clean up.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Three endpoints, and that is the whole objective. `wait_for_completion_timeout` is how long to block before handing back an id; `keep_alive` is how long the result is retained.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/async-search.html

```json
POST kibana_sample_data_*/_async_search?wait_for_completion_timeout=1ms&keep_alive=5m
{
  "size": 0,
  "aggs": { "per_index": { "terms": { "field": "_index" } } }
}
```

```json
GET _async_search/<id>
```

```json
DELETE _async_search/<id>
```

> Answer: the submit returns `is_running`/`is_partial` plus a base64 `id`. `GET _async_search/<id>`
> on this data returns `is_running: false`, `is_partial: false` and the aggregation:
> `.ds-kibana_sample_data_logs-…` 14,074 · `kibana_sample_data_flights` 13,014 ·
> `kibana_sample_data_ecommerce` 4,675.
>
> :bulb: Details worth knowing:
> - `is_partial: true` with `is_running: false` means it finished but some shards failed.
> - `GET _async_search/status/<id>` returns just the status, cheaply, without the results.
> - Add `keep_on_completion=true` to keep results even for searches that finish inside the timeout.
> - Default `keep_alive` is 5 days; results live in a hidden system index until deleted.

</details>
<hr>

:question: **Q72.** Return only the `Origin*` fields of a flight document — but not its geo point — plus the timestamp formatted `yyyy-MM-dd`.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: `_source` filtering trims what comes back from the stored JSON; `docvalue_fields` reads from doc values and can **reformat** dates and numbers on the way out.

```json
GET kibana_sample_data_flights/_search
{
  "size": 1,
  "_source": {
    "includes": [ "Origin*" ],
    "excludes": [ "OriginLocation" ]
  },
  "docvalue_fields": [
    { "field": "timestamp", "format": "yyyy-MM-dd" },
    "AvgTicketPrice"
  ]
}
```

> Answer: `_source` contains `OriginWeather`, `OriginCityName`, `OriginCountry`, `Origin`,
> `OriginAirportID`, `OriginRegion` — no `OriginLocation` — and a separate `fields` block with
> `"timestamp": ["2026-08-17"]` and `"AvgTicketPrice": [841.265625]`.
>
> :bulb: Three retrieval mechanisms, and the exam does distinguish them:
> `_source` (raw JSON as indexed) · `docvalue_fields` (columnar, formattable, no `text` fields) ·
> `fields` (the modern one — resolves aliases, applies formats, **and is the only one that shows
> runtime fields**, see Q23).

</details>
<hr>

# Part 9 — Cluster management

> **Objectives 20–25.** Shard troubleshooting is usually its own task and is very winnable.
> Snapshots and SLM are mechanical marks. Cross-cluster work and searchable snapshots need a second
> cluster and a non-`basic` licence, so they tend to be one task at most.

:warning: Q79 (CCR) and Q80 (searchable snapshots) need a **trial or platinum licence**. On `basic` you get
`current license is non-compliant for [searchable-snapshots]` / `[ccr]`. Start a trial with
`POST /_license/start_trial?acknowledge=true` (and `POST /_license/start_basic` to go back).

<hr>

:question: **Q73.** :fire: `GET _cluster/health` reports **yellow** with 23 unassigned shards. Diagnose it precisely and fix it.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Never guess. There is an API that tells you the exact decider that said no.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/cluster-allocation-explain.html

**1 — how bad, and where:**

```json
GET _cluster/health
GET _cluster/health?level=indices
GET _cat/shards?v&h=index,shard,prirep,state,unassigned.reason&s=state
```

**2 — ask why, about one specific shard:**

```json
GET _cluster/allocation/explain
{
  "index": "kibana_sample_data_ecommerce",
  "shard": 0,
  "primary": false
}
```

> Answer, on a single-node cluster:
> ```
> "current_state": "unassigned",
> "can_allocate": "no",
> "deciders": [ { "decider": "same_shard", "decision": "NO",
>   "explanation": "a copy of this shard is already allocated to this node …" } ]
> ```
> **Yellow, not red** — every primary is assigned; only replicas are homeless, and a replica can
> never sit on the same node as its primary. On a one-node cluster this is expected, not a fault.
>
> The fix, if the question wants green on one node — name the indices, or use a wildcard that does
> not sweep up system indices:
> ```json
> PUT kibana_sample_data_*/_settings
> { "index": { "number_of_replicas": 0 } }
> ```
> `PUT _all/_settings` does the same thing in one shot but also touches every hidden `.internal-*`
> and `.ds-*` index. Fine in a lab, careless on anything you care about.
>
> :bulb: Read the colours literally: **yellow** = all primaries assigned, some replicas are not.
> **red** = at least one **primary** is unassigned, so some data is unreachable. Yellow is a
> resilience problem; red is a data problem.
>
> :bulb: `GET _cluster/allocation/explain` with **no body** picks a random unassigned shard —
> handy for a first look, but always name the shard once you know which one matters.

</details>
<hr>

:question: **Q74.** Work through the other common `unassigned.reason` values and the fix for each.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Learn to read the decider name — it names the fix.

| Decider / reason | What happened | Fix |
| --- | --- | --- |
| `same_shard` | Replica cannot share a node with its primary | Add a node, or reduce `number_of_replicas` |
| `disk_threshold` | Node above the low/high watermark (85 % / 90 %) | Free space, or raise `cluster.routing.allocation.disk.watermark.*` |
| `filter` | An index or cluster allocation filter excludes every node | Clear `index.routing.allocation.*` / `cluster.routing.allocation.exclude.*` |
| `data_tier` | Index requires a tier (`data_warm`) that no node provides | Fix `index.routing.allocation.include._tier_preference`, or add a node with that role |
| `max_retry` | Allocation failed 5 times and gave up | Fix the cause, then `POST _cluster/reroute?retry_failed=true` |
| `NODE_LEFT` / `CLUSTER_RECOVERED` | Normal recovery in progress | Wait; watch `GET _cat/recovery?v&active_only=true` |
| Allocation disabled | Someone set `cluster.routing.allocation.enable: none` | Set it back to `all` |

```json
PUT _cluster/settings
{ "persistent": { "cluster.routing.allocation.enable": "all" } }
```

```json
POST _cluster/reroute?retry_failed=true
```

```json
GET _cluster/settings?include_defaults=false&flat_settings=true
```

> Answer: :bulb: The single most useful habit — before touching anything, run
> `GET _cluster/settings` and check nobody has left an `exclude` or `enable: none` behind. That is
> the deliberately planted fault in most exam-style "repair this cluster" tasks.
>
> :warning: `POST _cluster/reroute?retry_failed=true` retries; `allocate_stale_primary` and
> `allocate_empty_primary` **accept data loss** and need `"accept_data_loss": true`. Only reach for
> those when the question explicitly says the data is gone.

</details>
<hr>

:question: **Q75.** Register a filesystem snapshot repository, take a snapshot of two named indices, and confirm it succeeded.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: The path **must** be listed in `path.repo` in `elasticsearch.yml` on every node, or the registration is refused. On a multi-node cluster it must be a shared filesystem.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshots-register-repository.html

```yaml
# elasticsearch.yml
path.repo: ["/mount/backups"]
```

```json
PUT _snapshot/backups
{
  "type": "fs",
  "settings": {
    "location": "/mount/backups",
    "compress": true
  }
}
```

```json
POST _snapshot/backups/_verify
```

```json
PUT _snapshot/backups/snapshot-1?wait_for_completion=true
{
  "indices": "kibana_sample_data_ecommerce,kibana_sample_data_flights",
  "include_global_state": false,
  "ignore_unavailable": true
}
```

> Answer: `{"snapshot":{"snapshot":"snapshot-1","state":"SUCCESS","shards":{"total":2,"failed":0,"successful":2}}}`.
> List them with `GET _cat/snapshots/backups?v` or `GET _snapshot/backups/_all`.
>
> :bulb: `include_global_state` decides whether cluster settings, index templates, ILM policies and
> ingest pipelines travel with the snapshot. `true` for a whole-cluster backup, `false` when you are
> only moving data. Snapshots are **incremental** — snapshot-2 only stores segments that changed.

</details>
<hr>

:question: **Q76.** :fire: Restore that snapshot. Do it twice: once over the original index name, and once alongside the original under a new name.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: You cannot restore over an **open** index. Two ways round it, and the exam asks for both.

**(a) Over the original — close it first:**

```json
POST kibana_sample_data_ecommerce/_close
```

```json
POST _snapshot/backups/snapshot-1/_restore?wait_for_completion=true
{ "indices": "kibana_sample_data_ecommerce" }
```

**(b) Alongside — rename on the way in:**

```json
POST _snapshot/backups/snapshot-1/_restore?wait_for_completion=true
{
  "indices": "kibana_sample_data_ecommerce",
  "rename_pattern": "(.+)",
  "rename_replacement": "restored-$1",
  "index_settings": { "index.number_of_replicas": 0 },
  "ignore_index_settings": [ "index.lifecycle.name" ]
}
```

> Answer: restoring over an open index fails with
> `cannot restore index [...] because an open index with same name already exists in the cluster.
> Either close or delete the existing index or restore the index under a different name by
> providing a rename pattern and replacement name` — the error tells you both fixes.
> After a close-then-restore the index is reopened automatically and comes back with its documents.
>
> :bulb: `rename_pattern` is a **regex** and `rename_replacement` uses `$1` for its capture groups.
> `index_settings` overrides settings on restore; `ignore_index_settings` drops them entirely —
> the usual use is stripping an ILM policy so the restored copy is not immediately deleted again.

</details>
<hr>

:question: **Q77.** Automate a nightly snapshot of every `logs-*` index at 01:30, keeping 30 days but never fewer than 5 and never more than 50 snapshots. Then run it once immediately to prove it works.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: SLM. The `name` uses **date math** in angle brackets, which must be URL-escaped if you use `curl` but can be pasted as-is in Dev Tools.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/snapshot-lifecycle-management.html

```json
PUT _slm/policy/nightly-logs
{
  "schedule": "0 30 1 * * ?",
  "name": "<logs-snap-{now/d}>",
  "repository": "backups",
  "config": {
    "indices": [ "logs-*" ],
    "include_global_state": false,
    "ignore_unavailable": true
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}
```

```json
POST _slm/policy/nightly-logs/_execute
GET _slm/policy/nightly-logs
GET _slm/stats
```

> Answer: `GET _slm/policy/nightly-logs` returns the policy plus `next_execution_millis` and a
> `stats` block (`snapshots_taken`, `snapshots_failed`, `snapshots_deleted`). `_execute` returns the
> generated snapshot name.
>
> :bulb: The schedule is a **Quartz** cron expression with **six** fields —
> `second minute hour day-of-month month day-of-week`. `0 30 1 * * ?` is 01:30 daily. Writing a
> five-field Unix cron here is a very easy way to lose the mark.
>
> :bulb: Retention runs on its own schedule (`slm.retention_schedule`, 01:30 daily by default), not
> at snapshot time. Force it with `POST _slm/_execute_retention`.

</details>
<hr>

:question: **Q78.** Configure this cluster to search an index on a second cluster, and run a query spanning both.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Cross-cluster search is a **local** cluster setting plus a **prefixed** index name. Nothing is copied.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/modules-cross-cluster-search.html

```json
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_two": {
          "seeds": [ "10.0.0.2:9300" ],
          "skip_unavailable": true
        }
      }
    }
  }
}
```

```json
GET _remote/info
```

```json
GET kibana_sample_data_flights,cluster_two:kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": { "per_cluster": { "terms": { "field": "_index" } } }
}
```

> Answer: `GET _remote/info` shows `"connected": true` and the number of `num_nodes_connected`.
> The search returns hits from both, and `_index` values on remote hits are prefixed
> `cluster_two:`.
>
> :bulb: Details the exam likes:
> - Seeds use the **transport** port (9300), not 9200.
> - `skip_unavailable: true` stops an unreachable remote from failing the whole search.
> - Wildcards work: `GET *:logs-*/_search` hits every configured remote.
> - Remove a remote by setting its config to `null`.
> - `mode: proxy` instead of `sniff` when the remote is behind a load balancer.
>
> :warning: Not executable on a single-cluster lab — build a second container from
> [docker-compose.yml](docker-compose.yml) on a different port to practise it properly.

</details>
<hr>

:question: **Q79.** Continuously replicate an index from `cluster_two` into this cluster, and then automatically follow every new index matching `logs-*` there.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: CCR needs the same remote-cluster configuration as CCS (Q78) — set that up first. Follower indices are **read-only**.

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/xpack-ccr.html

```json
PUT logs-follower/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster": "cluster_two",
  "leader_index": "logs-leader"
}
```

```json
PUT _ccr/auto_follow/logs-pattern
{
  "remote_cluster": "cluster_two",
  "leader_index_patterns": [ "logs-*" ],
  "follow_index_pattern": "{{leader_index}}-replica"
}
```

```json
GET logs-follower/_ccr/stats
GET _ccr/stats
```

> Answer: `GET _ccr/stats` shows `follower_global_checkpoint` tracking `leader_global_checkpoint`.
>
> :bulb: The lifecycle you may be asked to demonstrate:
> `POST <idx>/_ccr/pause_follow` → `POST <idx>/_ccr/resume_follow` →
> `POST <idx>/_close` then `POST <idx>/_ccr/unfollow` to turn it into a normal writable index.
> You must pause before you can unfollow.
>
> :warning: Requires a **platinum or trial** licence on both clusters, and soft deletes on the
> leader index (on by default since 7.0).

</details>
<hr>

:question: **Q80.** Move older indices to a searchable snapshot so they still answer queries but stop consuming local storage — manually, and then as part of an ILM policy.

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: A searchable snapshot **mounts** a snapshot as a real, queryable index. `full_copy` keeps a local copy (faster, `cold` tier); `shared_cache` streams from the repository on demand (much less disk, `frozen` tier).

https://www.elastic.co/guide/en/elasticsearch/reference/8.15/searchable-snapshots.html

**Manually:**

```json
POST _snapshot/backups/snap-logs-2026-01?wait_for_completion=true
{ "indices": "logs-2026-01", "include_global_state": false }
```

```json
POST _snapshot/backups/snap-logs-2026-01/_mount?wait_for_completion=true&storage=shared_cache
{
  "index": "logs-2026-01",
  "renamed_index": "logs-2026-01-frozen",
  "index_settings": { "index.number_of_replicas": 0 }
}
```

**Via ILM:**

```json
PUT _ilm/policy/logs-with-frozen
{
  "policy": {
    "phases": {
      "hot":  { "actions": { "rollover": { "max_age": "7d", "max_primary_shard_size": "50gb" } } },
      "cold": { "min_age": "30d",
                "actions": { "searchable_snapshot": { "snapshot_repository": "backups", "force_merge_index": true } } },
      "frozen": { "min_age": "90d",
                  "actions": { "searchable_snapshot": { "snapshot_repository": "backups", "storage": "shared_cache" } } },
      "delete": { "min_age": "365d", "actions": { "delete": { "delete_searchable_snapshot": false } } }
    }
  }
}
```

> Answer: the mounted index is searchable exactly like any other, and shows in
> `GET _cat/indices` with a much smaller `store.size`. Query it to prove it.
>
> :warning: :fire: On a `basic` licence **the policy will not even save**:
> `policy [...] defines the [searchable_snapshot] action but the current license is non-compliant
> for [searchable-snapshots]`. If you see that in your lab, `POST /_license/start_trial?acknowledge=true`.
>
> :bulb: Searchable-snapshot indices are read-only, and `delete_searchable_snapshot: false` in the
> delete phase keeps the underlying snapshot when the index is removed.

</details>
<hr>

:question: **Q81.** Free space urgently: the cluster is above the high watermark. What do you check and what can you actually do?

<details>
  <summary>View Solution (click to reveal)</summary>

:bulb: Watermarks: **low 85 %** (stop putting new shards here) · **high 90 %** (move shards away) · **flood-stage 95 %** (set every index on this node to read-only — this is the one that pages people at 3am).

```json
GET _cat/allocation?v
GET _cat/indices?v&s=store.size:desc
GET _cluster/settings?include_defaults=true&filter_path=**.disk.watermark*
```

```json
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low": "90%",
    "cluster.routing.allocation.disk.watermark.high": "93%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "97%"
  }
}
```

```json
PUT _all/_settings
{ "index.blocks.read_only_allow_delete": null }
```

> Answer: `GET _cat/allocation?v` gives `disk.percent` per node — that is where you start.
> After a flood-stage event, clearing `index.blocks.read_only_allow_delete` is what lets writes
> resume; **8.x clears it automatically once disk drops back below the high watermark**, but you may
> still need to do it manually if the question implies the block is stuck.
>
> :bulb: The `_cat` APIs worth having in muscle memory:
> `_cat/health?v` · `_cat/nodes?v` · `_cat/indices?v&health=red` · `_cat/shards?v` ·
> `_cat/allocation?v` · `_cat/recovery?v&active_only=true` · `_cat/pending_tasks?v` ·
> `_cat/thread_pool/write?v`. Add `&s=<column>` to sort and `&v` for headers — every one of them
> takes them.

</details>
<hr>

# Part 10 — Timed mock: ten chained tasks in three hours

> The real exam is roughly **10 tasks in 3 hours**, and they **chain** — task 8 searches what task 2
> built, and breaking an early index quietly breaks the later ones. This mock reproduces that shape.
>
> **How to run it:** set a 3-hour timer. Read all ten tasks **before** starting. Do not open a
> solution until the timer stops. Everything here runs on `kibana_sample_data_ecommerce` plus four
> log lines supplied in Task 6 — nothing else to download.

<details>
  <summary>Reset between attempts (click to reveal)</summary>

```json
DELETE _data_stream/mock-logs-web
DELETE mock-orders
DELETE _index_template/mock-logs-template
DELETE _component_template/mock-logs-mappings
DELETE _component_template/mock-logs-settings
DELETE _ingest/pipeline/mock-order-prep
DELETE _ingest/pipeline/mock-web-parse
DELETE _ilm/policy/mock-logs-policy
```

</details>

<hr>

### :question: Task 1 — Define the target index

Create `mock-orders` with 1 primary shard, no replicas, and a mapping that:

- rejects any field not listed;
- stores `order_id` for exact matching only;
- makes `customer_full_name`, `category` and `manufacturer` full-text searchable **and** aggregatable, and searchable together through a single field `search_all`;
- analyzes `category` so that searching `shoe` finds `Shoes`;
- stores `taxful_total_price` to two decimal places;
- reserves `price_band` and `order_month` as exact-match strings.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT mock-orders
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "analysis": {
      "analyzer": {
        "product_en": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [ "lowercase", "asciifolding", "porter_stem" ]
        }
      }
    }
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "order_id":           { "type": "keyword" },
      "order_date":         { "type": "date" },
      "customer_gender":    { "type": "keyword" },
      "customer_full_name": { "type": "text", "copy_to": "search_all",
                              "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } } },
      "category":           { "type": "text", "analyzer": "product_en", "copy_to": "search_all",
                              "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } } },
      "manufacturer":       { "type": "text", "copy_to": "search_all",
                              "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } } },
      "search_all":         { "type": "text" },
      "taxful_total_price": { "type": "scaled_float", "scaling_factor": 100 },
      "total_quantity":     { "type": "integer" },
      "price_band":         { "type": "keyword" },
      "order_month":        { "type": "keyword" }
    }
  }
}
```

> :bulb: "Searchable **and** aggregatable" is always a multi-field. "Searchable together" is
> `copy_to`. "Finds `Shoes` when you type `shoe`" is a stemmer.

</details>
<hr>

### :question: Task 2 — Populate it through a pipeline

Copy every eCommerce order into `mock-orders`, keeping only the fields Task 1 mapped. On the way, derive `price_band` (`low` < 50, `mid` < 150, otherwise `high`) and `order_month` as `yyyy-MM`.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT _ingest/pipeline/mock-order-prep
{
  "processors": [
    { "script": { "source": "ctx.price_band = ctx.taxful_total_price < 50 ? 'low' : (ctx.taxful_total_price < 150 ? 'mid' : 'high')" } },
    { "date": { "field": "order_date", "target_field": "order_month",
                "formats": [ "ISO8601" ], "output_format": "yyyy-MM" } },
    { "remove": { "field": "products", "ignore_missing": true } }
  ]
}
```

```json
POST _reindex?refresh
{
  "source": {
    "index": "kibana_sample_data_ecommerce",
    "_source": [ "order_id", "order_date", "customer_gender", "customer_full_name",
                 "category", "manufacturer", "taxful_total_price", "total_quantity" ]
  },
  "dest": { "index": "mock-orders", "pipeline": "mock-order-prep" }
}
```

> Answer: `{"total": 4675, "created": 4675}`, and `price_band` comes out
> **mid 2,782 · low 1,633 · high 260**.
>
> :warning: With `"dynamic": "strict"` in Task 1, **any** field you forget to exclude aborts the
> reindex. `source._source` is the clean way to control that; the `remove` processor is the belt and
> braces.

</details>
<hr>

### :question: Task 3 — Add a computed field without reindexing

`mock-orders` must expose `avg_item_price` — order value divided by item count — permanently, and without rewriting any documents. Report its min, max and average.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT mock-orders/_mapping
{
  "runtime": {
    "avg_item_price": {
      "type": "double",
      "script": {
        "source": """
          if (doc['total_quantity'].size() == 0 || doc['total_quantity'].value == 0) { return; }
          emit(doc['taxful_total_price'].value / doc['total_quantity'].value);
        """
      }
    }
  }
}
```

```json
GET mock-orders/_search
{
  "size": 0,
  "aggs": { "item_price": { "stats": { "field": "avg_item_price" } } }
}
```

> Answer: count 4,675 · min **6.99** · max **281.24** · avg **34.72**.

</details>
<hr>

### :question: Task 4 — The report

For the **3 categories with the highest revenue**, show that revenue and the **2 manufacturers contributing most revenue** within each.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET mock-orders/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category.keyword", "size": 3, "order": { "revenue": "desc" } },
      "aggs": {
        "revenue": { "sum": { "field": "taxful_total_price" } },
        "top_manufacturers": {
          "terms": { "field": "manufacturer.keyword", "size": 2, "order": { "mfr_revenue": "desc" } },
          "aggs": { "mfr_revenue": { "sum": { "field": "taxful_total_price" } } }
        }
      }
    }
  }
}
```

> Answer:
> | category | revenue | top manufacturers |
> | --- | --- | --- |
> | Men's Clothing | 149,385.30 | Low Tide Media 89,108.12 · Elitelligence 83,528.87 |
> | Women's Clothing | 135,091.88 | Tigress Enterprises 58,766.91 · Pyramidustries 50,626.61 |
> | Women's Shoes | 105,465.55 | Tigress Enterprises 47,333.88 · Pyramidustries 36,765.81 |
>
> :bulb: Slightly different from Q8's figures because `taxful_total_price` is now `scaled_float`
> rather than `half_float` — more precise, and a good illustration of why the exam cares which
> numeric type you choose.

</details>
<hr>

### :question: Task 5 — Expose it to applications

Add two aliases on `mock-orders`: `mock-high-value`, which only ever returns `high` price-band orders, and `mock-orders-write`, which applications index through.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
POST _aliases
{
  "actions": [
    { "add": { "index": "mock-orders", "alias": "mock-high-value",
               "filter": { "term": { "price_band": "high" } } } },
    { "add": { "index": "mock-orders", "alias": "mock-orders-write", "is_write_index": true } }
  ]
}
```

> Answer: `GET mock-high-value/_count` → **260**.

</details>
<hr>

### :question: Task 6 — Ingest raw web logs

Set up `mock-logs-web` as a **data stream** — 1 shard, no replicas, rolling over daily or at 25 GB, force-merged in warm after 3 days, deleted after 30 — and parse these lines into `client.ip`, `http.request.method`, `url.path`, `http.response.status_code`, `http.response.bytes` and `@timestamp`. Unparseable lines must still be indexed, tagged `event.kind: parse_failure`, and **must keep their raw message**; parsed lines must not.

```
10.0.0.7 - - [25/Aug/2026:10:15:32 +0000] "GET /checkout HTTP/1.1" 200 1043
10.0.0.9 - - [25/Aug/2026:10:15:35 +0000] "POST /api/pay HTTP/1.1" 503 210
192.168.4.4 - - [25/Aug/2026:10:16:01 +0000] "GET /products/42 HTTP/1.1" 404 155
garbage line with no structure
```

<details>
  <summary>View Solution (click to reveal)</summary>

**The pipeline first** — the component template references it, so create it before the template.

```json
PUT _ingest/pipeline/mock-web-parse
{
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": [
          "%{IPORHOST:client.ip} - - \\[%{HTTPDATE:ts}\\] \"%{WORD:http.request.method} %{URIPATHPARAM:url.path} HTTP/%{NUMBER}\" %{NUMBER:http.response.status_code:int} %{NUMBER:http.response.bytes:long}"
        ],
        "on_failure": [
          { "set": { "field": "event.kind", "value": "parse_failure" } },
          { "set": { "field": "@timestamp", "value": "{{{_ingest.timestamp}}}" } }
        ]
      }
    },
    { "date": { "field": "ts", "target_field": "@timestamp",
                "formats": [ "dd/MMM/yyyy:HH:mm:ss Z" ], "if": "ctx.ts != null" } },
    { "set": { "field": "event.kind", "value": "event", "if": "ctx.event?.kind == null" } },
    { "remove": { "field": "ts", "ignore_missing": true } },
    { "remove": { "field": "message", "ignore_missing": true,
                  "if": "ctx.event?.kind != 'parse_failure'" } }
  ]
}
```

```json
PUT _ilm/policy/mock-logs-policy
{
  "policy": {
    "phases": {
      "hot":    { "actions": { "rollover": { "max_primary_shard_size": "25gb", "max_age": "1d" },
                               "set_priority": { "priority": 100 } } },
      "warm":   { "min_age": "3d",
                  "actions": { "forcemerge": { "max_num_segments": 1 }, "set_priority": { "priority": 50 } } },
      "delete": { "min_age": "30d", "actions": { "delete": {} } }
    }
  }
}
```

```json
PUT _component_template/mock-logs-mappings
{
  "template": { "mappings": { "properties": {
    "@timestamp":                 { "type": "date" },
    "client.ip":                  { "type": "ip" },
    "http.request.method":        { "type": "keyword" },
    "url.path":                   { "type": "keyword" },
    "http.response.status_code":  { "type": "short" },
    "http.response.bytes":        { "type": "long" },
    "event.kind":                 { "type": "keyword" },
    "message":                    { "type": "text" }
  } } }
}
```

```json
PUT _component_template/mock-logs-settings
{
  "template": { "settings": {
    "index.number_of_shards": 1,
    "index.number_of_replicas": 0,
    "index.lifecycle.name": "mock-logs-policy",
    "index.default_pipeline": "mock-web-parse"
  } }
}
```

```json
PUT _index_template/mock-logs-template
{
  "index_patterns": [ "mock-logs-*" ],
  "data_stream": { },
  "composed_of": [ "mock-logs-mappings", "mock-logs-settings" ],
  "priority": 600
}
```

```json
POST mock-logs-web/_bulk?refresh
{"create":{}}
{"message":"10.0.0.7 - - [25/Aug/2026:10:15:32 +0000] \"GET /checkout HTTP/1.1\" 200 1043"}
{"create":{}}
{"message":"10.0.0.9 - - [25/Aug/2026:10:15:35 +0000] \"POST /api/pay HTTP/1.1\" 503 210"}
{"create":{}}
{"message":"192.168.4.4 - - [25/Aug/2026:10:16:01 +0000] \"GET /products/42 HTTP/1.1\" 404 155"}
{"create":{}}
{"message":"garbage line with no structure"}
```

> Answer: `{"errors": false}` for all four. The first parses to
> `{"@timestamp":"2026-08-25T10:15:32.000Z","client":{"ip":"10.0.0.7"},"http":{"request":{"method":"GET"},"response":{"status_code":200,"bytes":1043}},"url":{"path":"/checkout"},"event":{"kind":"event"}}`
> with no `message`; the last keeps
> `{"message":"garbage line with no structure","event":{"kind":"parse_failure"},"@timestamp":"<ingest time>"}`.
> `GET _data_stream/mock-logs-web` shows generation 1, template `mock-logs-template`, policy
> `mock-logs-policy`.
>
> :warning: Three things to get right here:
> - `_bulk` into a data stream must use **`create`**, not `index`.
> - Every document needs `@timestamp` — hence setting it in `on_failure` too, or the garbage line is rejected outright.
> - `priority: 600` beats the built-in `logs` template at 100. Always check with `_simulate_index`.

</details>
<hr>

### :question: Task 7 — Find the failures

From the log data stream, return the requests that failed (status ≥ 400), newest first, and separately count requests by status class.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET mock-logs-web/_search
{
  "query": { "range": { "http.response.status_code": { "gte": 400 } } },
  "sort": [ { "@timestamp": "desc" } ]
}
```

```json
GET mock-logs-web/_search
{
  "size": 0,
  "runtime_mappings": {
    "status_class": {
      "type": "keyword",
      "script": {
        "source": """
          if (doc['http.response.status_code'].size() == 0) { return; }
          long c = doc['http.response.status_code'].value;
          emit((c / 100) + 'xx');
        """
      }
    }
  },
  "aggs": { "classes": { "terms": { "field": "status_class" } } }
}
```

> Answer: 2 failures (503 and 404); classes `2xx` 1, `4xx` 1, `5xx` 1.
>
> :bulb: `(c / 100)` on a `long` is integer division — 503 / 100 = 5. No rounding needed.

</details>
<hr>

### :question: Task 8 — The application query

Against `mock-orders`, find orders that mention `Oceanavigations` anywhere in the searchable text, cost 100 or more, were placed by women, are **not** in the `Women's Accessories` category, and are **either** high value **or** placed in September 2026. Return them most expensive first, two per page, with a stable tiebreaker.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
GET mock-orders/_search
{
  "size": 2,
  "query": {
    "bool": {
      "must": [ { "match": { "search_all": "Oceanavigations" } } ],
      "filter": [
        { "range": { "taxful_total_price": { "gte": 100 } } },
        { "term":  { "customer_gender": "FEMALE" } }
      ],
      "should": [
        { "term": { "price_band": "high" } },
        { "term": { "order_month": "2026-09" } }
      ],
      "minimum_should_match": 1,
      "must_not": [ { "term": { "category.keyword": "Women's Accessories" } } ]
    }
  },
  "sort": [ { "taxful_total_price": "desc" }, { "order_id": "asc" } ],
  "_source": [ "order_id", "taxful_total_price" ]
}
```

> Answer: **38** matching orders. Page 1 is order 733089 at 233.96, then 576600 at 222.98 —
> each hit carries `"sort": [233.96, "733089"]`, which is what you feed to `search_after` for page 2.
>
> :warning: Without `minimum_should_match: 1` this returns far more than 38 — the `should` clauses
> would only boost, not filter.

</details>
<hr>

### :question: Task 9 — Rename a value in place

Business has renamed the `high` price band to `premium`. Update every affected document without reindexing, and then check nothing else broke.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
POST mock-orders/_update_by_query?refresh&conflicts=proceed
{
  "query": { "term": { "price_band": "high" } },
  "script": {
    "source": "ctx._source.price_band = params.band",
    "params": { "band": "premium" }
  }
}
```

> Answer: `{"total": 260, "updated": 260}` and the terms aggregation now shows
> mid 2,782 · low 1,633 · **premium 260**.
>
> :warning: :fire: **This silently breaks Task 5.** `GET mock-high-value/_count` now returns **0** —
> the alias filter still says `"price_band": "high"` and nothing matches it any more. Chained
> breakage like this is exactly what the real exam does to you. Fix it in the same atomic call:
>
> ```json
> POST _aliases
> {
>   "actions": [
>     { "remove": { "index": "mock-orders", "alias": "mock-high-value" } },
>     { "add":    { "index": "mock-orders", "alias": "mock-high-value",
>                   "filter": { "term": { "price_band": "premium" } } } }
>   ]
> }
> ```
>
> :bulb: Also update the `mock-order-prep` pipeline, or the next reindex writes `high` again.

</details>
<hr>

### :question: Task 10 — Back it up, then explain the cluster's health

Register a repository, snapshot both `mock-orders` and the log data stream, schedule that snapshot nightly at 02:00 with 14-day retention, and then explain why `GET _cluster/health` reports yellow.

<details>
  <summary>View Solution (click to reveal)</summary>

```json
PUT _snapshot/backups
{ "type": "fs", "settings": { "location": "/mount/backups", "compress": true } }
```

```json
PUT _snapshot/backups/mock-final?wait_for_completion=true
{
  "indices": "mock-orders,mock-logs-web",
  "include_global_state": false,
  "ignore_unavailable": true
}
```

```json
PUT _slm/policy/mock-nightly
{
  "schedule": "0 0 2 * * ?",
  "name": "<mock-snap-{now/d}>",
  "repository": "backups",
  "config": { "indices": [ "mock-*" ], "include_global_state": false, "ignore_unavailable": true },
  "retention": { "expire_after": "14d", "min_count": 5, "max_count": 50 }
}
```

```json
GET _cluster/health
GET _cat/shards?v&h=index,shard,prirep,state,unassigned.reason&s=state
GET _cluster/allocation/explain
{ "index": "kibana_sample_data_ecommerce", "shard": 0, "primary": false }
```

> Answer: the snapshot succeeds with `"state": "SUCCESS"` and reports
> `["mock-orders", ".ds-mock-logs-web-2026.08.25-000001"]` — naming a **data stream** in `indices`
> resolves to its backing indices automatically.
>
> The cluster is yellow because `same_shard` refuses to put a replica on the node that already holds
> its primary — a single-node cluster can never be green while any index has replicas. `mock-orders`
> and the log stream were both created with `number_of_replicas: 0`, so they are **green**; the
> Kibana sample indices ship with 1 replica and are yellow. Confirm per index with
> `GET _cluster/health?level=indices`.
>
> :bulb: If a task asks you to *make* it green on one node, drop replicas on the offending indices:
> `PUT kibana_sample_data_*/_settings {"index": {"number_of_replicas": 0}}`.

</details>
<hr>

## Marking yourself

The real exam is graded per task, with partial credit. Score yourself the same way:

| Score | Meaning |
| --- | --- |
| **9–10 tasks, inside 3 hours** | You are ready. |
| **7–8** | Likely pass. Find the two you were slowest on and drill those parts. |
| **5–6** | Not yet. The gap is almost always speed in the docs, not knowledge — redo Parts 1, 2 and 4 against a clock. |
| **< 5** | Work through Parts 1–9 in order before attempting the mock again. |

:bulb: **Time management on the day.** Read every task first and do the ones you know cold — an
ingest pipeline or an aggregation you can write from memory is worth the same as the cross-cluster
task you will fight for 40 minutes. Never leave a task blank; partial credit is real, and a mapping
that is 80 % right scores more than an empty answer.
