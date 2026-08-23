# Elastic Certified Engineer Exam — 8.15

Practice notes and worked exercises for the **Elastic Certified Engineer** exam, aligned to the **8.15** exam objectives.

Originally based on https://github.com/mohclips/Elastic-Certified-Engineer-Exam-Notes (7.x), then updated for 8.1, and now re-aligned to the 8.15 objective list.
<br>
Big thank you to Cosmospnw.com for their reliable data.

## What each file is

| File | What it is | How to use it |
| --- | --- | --- |
| [example.md](example.md) | **Guided end-to-end walkthrough.** One dataset, every objective in order, each with a :question: task and a collapsed solution. | The main drill. Try each task before revealing the answer. |
| [Data_Management.md](Data_Management.md), [Searching_Data.md](Searching_Data.md), [Developing_Search_Applications.md](Developing_Search_Applications.md), [Data_Processing.md](Data_Processing.md), [Cluster_Management.md](Cluster_Management.md) | **Per-objective reference + exercises.** Deeper coverage of one exam section each, with the syntax tables, gotchas and API options. | Go here when a topic in `example.md` doesn't stick, or to revise one weak area. |
| [concepts_and_terms.md](concepts_and_terms.md) | **Glossary.** Plain-English definitions of the concepts the exam assumes. | Skim early, revisit when a term confuses you. |
| [documentation_reference.md](documentation_reference.md) | **Link index**, objective by objective, all pointing at the 8.15 docs. | Use it to practise *navigating the official docs*, which is all you get on exam day. |

:warning: **None of this is a timed mock exam.** Every task here shows you its answer. Real exam conditions mean 3 hours, no solutions, and only the official docs — so once the material is familiar, re-do `example.md` against a clock with the solutions collapsed.

## :floppy_disk: Datasets — load these first

Three datasets are used across this repo. **Load the one a file needs before starting it**, or its exercises will return zero results.

| Dataset | File | Used by | Load into |
| --- | --- | --- | --- |
| **Eclipse totality** (190 state parks) | `example-date/full-eclipse-data.json` | [example.md](example.md) | `totality-raw` |
| **Shakespeare** (111k lines) | `shakespeare_6.0.json` | [Searching_Data.md](Searching_Data.md), [Data_Processing.md](Data_Processing.md), [Developing_Search_Applications.md](Developing_Search_Applications.md) | `shakespeare` |
| **Bank accounts** (1000 docs) | `accounts.json` | [Data_Processing.md](Data_Processing.md), [Developing_Search_Applications.md](Developing_Search_Applications.md), [Data_Management.md](Data_Management.md) | `accounts-raw` |

A few exercises also use the **Kibana sample data** (eCommerce, Flights, Logs). Add those from **Kibana → Home → Try sample data**.

### Loading them

```bash
# 1. Eclipse data - the action lines already name the index, so POST to /_bulk
curl -k -u "elastic:Password01" -H "Content-Type: application/x-ndjson" \
  -XPOST "localhost:9200/_bulk?refresh" --data-binary "@example-date/full-eclipse-data.json"
```

```bash
# 2. Bank accounts - action lines have no index, so name it in the URL
curl -k -u "elastic:Password01" -H "Content-Type: application/x-ndjson" \
  -XPOST "localhost:9200/accounts-raw/_bulk?refresh" --data-binary "@accounts.json"
```

For Shakespeare, create the mapping first, then bulk load:
```json
PUT /shakespeare
{
  "mappings": {
    "properties": {
      "speaker":       { "type": "keyword" },
      "play_name":     { "type": "keyword" },
      "line_id":       { "type": "integer" },
      "speech_number": { "type": "integer" }
    }
  }
}
```
```bash
curl -k -u "elastic:Password01" -H "Content-Type: application/x-ndjson" \
  -XPOST "localhost:9200/shakespeare/_bulk?refresh" --data-binary "@shakespeare_6.0.json"
```

Verify all three:
```json
GET totality-raw/_count     // 190
GET accounts-raw/_count     // 1000
GET shakespeare/_count      // 111396
```

:warning: **Load `accounts-raw` *before* you create the `accounts-*` index template** in [Data_Management.md](Data_Management.md). That template maps `gender` as an explicit `keyword`, whereas every `gender.keyword` query elsewhere in the repo relies on `accounts-raw` having been **dynamically** mapped (`text` + a `.keyword` sub-field). Create the template first and those queries silently return zero hits. See the warning in [Data_Management.md](Data_Management.md) for the details and the fix.

## :rotating_light: Version warning

- The exam is on **Elastic 8.15** through **31 August 2026**.
- **From 1 September 2026 the exam moves to Elastic 9.3.** If your exam date is on or after that day, these notes are still ~90% correct (the objectives are near identical), but you should double-check 9.x behaviour for: removal of legacy `_template`, security defaults, and any deprecated syntax.

## Exam objectives (8.15) → where they live in this repo

### 1. Data Management — [Data_Management.md](Data_Management.md)
| Objective | Covered |
| --- | --- |
| Define an index that satisfies a given set of requirements | :white_check_mark: |
| Define and use a dynamic template that satisfies a given set of requirements | :white_check_mark: |
| Define an Index Lifecycle Management policy for a time-series index | :white_check_mark: |
| Define an index template that creates a new data stream | :white_check_mark: |

### 2. Searching Data — [Searching_Data.md](Searching_Data.md)
| Objective | Covered |
| --- | --- |
| Write and execute a search query for terms and/or phrases in one or more fields of an index | :white_check_mark: |
| Write and execute a search query that is a Boolean combination of multiple queries and filters | :white_check_mark: |
| Write an asynchronous search | :white_check_mark: |
| Write and execute metric and bucket aggregations | :white_check_mark: |
| Write and execute aggregations that contain sub-aggregations | :white_check_mark: |
| Write and execute a query that searches across multiple clusters | :white_check_mark: |
| Write and execute a search that utilizes a runtime field | :white_check_mark: |

### 3. Developing Search Applications — [Developing_Search_Applications.md](Developing_Search_Applications.md)
| Objective | Covered |
| --- | --- |
| Sort the results of a query by a given set of requirements | :white_check_mark: |
| Implement pagination of the results of a search query | :white_check_mark: |
| Define and use index aliases | :white_check_mark: |

### 4. Data Processing — [Data_Processing.md](Data_Processing.md)
| Objective | Covered |
| --- | --- |
| Define a mapping that satisfies a given set of requirements | :white_check_mark: |
| Define and use multi-fields with different data types and/or analyzers | :white_check_mark: |
| Use the Reindex API and Update By Query API to reindex and/or update documents | :white_check_mark: |
| Define and use an ingest pipeline that satisfies a given set of requirements | :white_check_mark: |
| Define runtime fields to retrieve custom values using Painless scripting | :white_check_mark: |

### 5. Cluster Management — [Cluster_Management.md](Cluster_Management.md)
| Objective | Covered |
| --- | --- |
| Diagnose shard issues and repair a cluster's health | :white_check_mark: |
| Backup and restore a cluster and/or specific indices | :white_check_mark: |
| Configure a snapshot to be searchable | :white_check_mark: |
| Configure a cluster for cross-cluster search | :white_check_mark: |
| Implement cross-cluster replication | :white_check_mark: |
| Automate snapshots with Snapshot Lifecycle Management | :white_check_mark: |

## :books: Topics that were on older objective lists but are NOT listed for 8.15

These are kept in the repo, clearly marked as **bonus**. They are still genuinely useful — several are prerequisites for the listed objectives (you cannot create a data stream without an index template, and you cannot build a multi-field with a custom analyzer without knowing analyzers) — but do not spend your last revision hours on them:

- Define and use an index template for a given pattern *(prerequisite for data streams — still learn it)*
- Define and use a custom analyzer *(folded into the multi-fields objective — still learn it)*
- Highlight the search terms in the response of a query
- Define and use a search template
- Configure an index so that it properly maintains the relationships of nested arrays of objects
- Define role-based access control using Elasticsearch Security

## Resources
Here are some resources that you can use for studying: <br>
https://github.com/LisaHJung/Part-6-Troubleshooting-beginner-level-Elasticsearch-Errors/blob/main/README.md <br>
https://github.com/mohclips/Elastic-Certified-Engineer-Exam-Notes <br>
https://github.com/LisaHJung/Beginners-Crash-Course-to-Elastic-Stack-Series-Table-of-Contents <br>

This will help you understand what to expect for exam questions and exam related questions: <br>
https://www.youtube.com/watch?v=dzo_uR3IsbQ

Official pages: <br>
https://www.elastic.co/training/elastic-certified-engineer-exam <br>
https://www.elastic.co/training/certification/faq

## Important Notes:
1) Exam fee is **$500 USD per attempt** (it was $400 for the 8.1-era exam — the price went up, budget accordingly).
2) The exam lasts 3 hours. This 3 hours starts when you start the exam. The exam does **NOT** begin when you join the proctored exam session. It begins when you see the screen to login and start the exam. See the YouTube video for more information around this.
3) A failed attempt means **buying a new attempt** and waiting **at least 14 days** before re-sitting. Retakes are not discounted.
4) You have **one year** from purchase to use an exam attempt.
5) During the exam you get the official Elastic documentation at https://www.elastic.co/guide/index.html. Google, Stack Overflow, and every other site are blocked. **Practise navigating the official docs only** — this repo will not be available to you on exam day.
6) All of the questions and topics can be solved using the APIs. Some items can also be done in the Kibana UI (creating an ILM policy or an SLM policy, for example). Everything the UI can do, the API can do, and the entire exam can be done from the Dev Tools console.

### :bulb: Practical exam tips
- **Read the whole paper first.** Tasks are frequently chained (task 4 reindexes what task 2 built). If you break an early index, later tasks fail too.
- **Check whether a task is destructive.** Prefer `_simulate` (ingest pipelines), `_simulate/index` (index templates), and a plain `_search` before converting it into a `_reindex`/`_update_by_query`.
- **`filter_path` is your friend** for eyeballing output quickly, e.g. `GET idx/_search?filter_path=hits.total.value`.
- **Bookmark the docs sections you use most** at the start of the exam: Query DSL, aggregations, mapping parameters, ILM actions, snapshot/restore.
