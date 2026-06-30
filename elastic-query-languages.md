# TryHackMe — Elastic: Query Languages

**Platform:** TryHackMe
**Module:** Advanced ELK
**Completed by:** Aysu Nezirova
**Date:** June 29, 2026
**Tasks completed:** 8

---

## Overview

This room covers advanced search techniques in Kibana, including KQL, Lucene, and ES|QL syntax, nested data queries, regular expressions, range queries, fuzzy and proximity searches, and core Elastic Stack architecture concepts (ECS and EQL).

---

## Task 1 — Introduction

No questions. Learning objectives reviewed and ready to proceed.

---

## Task 2 — Elastic Languages and Syntax

### Supported Query Languages

| Language | Description | Example |
|----------|-------------|---------|
| Lucene | Core Elasticsearch query syntax, supports advanced pattern matching | `flag: thm` |
| KQL | Simple, user-friendly language for filtering in Kibana | `flag: thm` |
| ES\|QL | Pipe-based language for structured data analysis | `FROM index \| WHERE flag == "thm"` |

### Special Characters

Characters such as `+ - && || ! ( { [ ^ ~ * ? : /` are interpreted as operators and must be escaped or quoted.

```
Quotes:    flag: "THM{kql!}"
Backslash: flag: THM\{kql\!\}
```

### Field Types: text vs keyword

| Type | Behavior | Wildcard Impact |
|------|----------|------------------|
| text | Tokenized into separate words during indexing | Wildcards match per-token, not full value |
| keyword | Stored exactly as written, not analyzed | Wildcards match the full string |

### Questions

**Q: The value `david:davidson` exists in the `user.name` field. What query would you use to search for this exact value?**
**A:** `user.name: "david:davidson"`

**Q: Suppose you want to find values such as `hack` and `hacking` in the `activity` field. What wildcard query would return these results?**
**A:** `activity: hack*`

---

## Task 3 — Navigating Nested Data

Nested fields store arrays of objects, allowing queries against specific attributes within those objects (e.g., `comments.author`, `comments.text`).

```
comments.author: *                              # match any value
comments.author: "Alice"                        # exact match
comments.author: "Alice" AND comments.text: "attack"
comments.author: ("Alice" AND "Bob") AND comments.text: "attack"
comments.author: ("Alice" OR "Bob") AND comments.text: "attack"
```

### Query 1 — Exact filename match

```
affected_systems.affected_files.file_name: "marketing_strategy_2023_07_23.pptx"
```

### Query 2 — Wildcard + system type

```
affected_systems.affected_files.file_name: marketing_strategy* AND affected_systems.system_type: "File Server"
```

### Query 3 — Multiple users + Web Server

```
affected_systems.system_type: "Web Server" AND affected_systems.logged_on_users: ("admin" AND "it")
```

### Questions

**Q: How many incidents involve the file `marketing_strategy_2023_07_23.pptx`?**
**A:** `4`

**Q: How many incidents involve files on a File Server with names that start with `marketing_strategy`?**
**A:** `135`

**Q: Which Web Server had a true positive incident where both `admin` and `it` users were logged in?**
**A:** `web-server-77`

---

## Task 4 — Investigating With Regular Expressions

Lucene regex queries are wrapped in forward slashes `/pattern/`. Behavior differs between `keyword` fields (matches the full value) and `text` fields (matches per analyzed token).

```
Event_Type: /.*/                  # match anything
Event_Type: /(S|M).*/             # starts with S or M
Description: /(s|m).*/            # matches individual words (text field)
Description: /(s|m).*/ AND /user.*/
```

### Questions

**Q: How many events exist where a `client_list` file was affected by `ransomware`?**
**A:** `69` (using `incident_comments: /ransomware/`)

**Q: Investigate `affected_systems.affected_files.file_name` using `/.*project.*/`. What is the affected system name on the latest event managed by `EVenis`?**
**A:** `web-server-37`

---

## Task 5 — Defining Ranges and Boundaries

Range queries use comparison operators (`>=`, `<=`, `>`, `<`) to match numeric values, timestamps, or other ranges.

```
response_time_seconds >= 100
response_time_seconds < 300
response_time_seconds >= 100 AND response_time_seconds <= 300
@timestamp >= "2023-01-01"
@timestamp >= "2023-01-01" AND @timestamp < "2023-03-01"
```

### Query 1 — Data Leak with high severity

```
incident_type: "Data Leak" AND severity_level >= 9
```

### Query 2 — AJohnston before Dec 1, 2022

```
team_members.name: "AJohnston" AND @timestamp <= "2022-12-01" AND affected_systems.system_type: ("Email Server" OR "Web Server")
```

### Query 3 — incident_id range + false positive lookup

```
incident_id >= 1 AND incident_id <= 500 AND incident_type: "Data Leak" AND affected_systems.system_name: "file-server-65"
```

### Questions

**Q: How many `Data Leak` incidents have a `severity_level` of `9` or higher?**
**A:** `52`

**Q: Check out incidents investigated by `AJohnston` on or before `December 1, 2022`. How many events occurred on an `Email Server` or `Web Server`?**
**A:** `65`

**Q: Investigate `incident_id` 1-500. Which analyst stated that the `Data Leak` on `file-server-65` was a false positive?**
**A:** `JLim`

---

## Task 6 — Searching Variants and Proximity

Fuzzy searches (`~`) find terms with character variations. Proximity searches find terms within a specified word distance using `"phrase"~slop_value`.

```
server01~1                       # fuzzy: 1 character difference
server01~2                       # fuzzy: 2 character differences
log_message: "server error"~1    # proximity: within 1 position
```

### Query 1 — Fuzzy search on "true"

```
incident_comments: true~1 AND team_members.name: "JLim"
```

### Query 2 — Proximity search

```
incident_comments: "data-leak true-negative"~3
```

### Query 3 — Proximity + AJohnston

```
incident_comments: "detected negative"~2 AND team_members.name: "AJohnston"
```

### Questions

**Q: How many incidents has `JLim` handled in which the word `true` appears with a single character difference?**
**A:** `110`

**Q: How many events contain `data-leak true-negative` within three words of each other?**
**A:** `33`

**Q: How many incidents handled by `AJohnston` contain `detected` and `negative` within two positions?**
**A:** `40`

---

## Task 7 — Elastic Query Architecture

### Elastic Common Schema (ECS)

ECS is a standardized set of field-naming conventions that normalizes how fields are structured across different data sources, enabling consistent cross-source querying.

```
VPN server:  src_ip          ->  ECS: source.ip
Web server:  client_address  ->  ECS: source.ip
Firewall:    source_address  ->  ECS: source.ip
```

### Event Query Language (EQL)

EQL is a correlation-focused query language for security use cases. Unlike KQL/Lucene, it identifies relationships between multiple events over time, and is primarily used in detection engineering and threat hunting.

```
file where host.os.type == "linux" and
 event.action in ("rename", "creation") and
 file.path in (
   "/etc/crontab",
   "/etc/cron.allow",
   "/etc/cron.deny"
 )
```

### Questions

**Q: Which Elastic component standardizes field names across different data sources?**
**A:** `ECS` (Elastic Common Schema)

**Q: Which Elastic language is often used to write event correlation detection rules?**
**A:** `EQL` (Event Query Language)

---

## Task 8 — Conclusion

Room completed successfully! 🎉

---

## Key Takeaways

- KQL is simple and user-friendly; Lucene supports regex and fuzzy/proximity search; ES|QL uses pipe syntax
- `text` fields are tokenized (word-based matching); `keyword` fields match the full string exactly
- Nested data fields use dot notation (e.g., `comments.author`) and support AND/OR grouping with parentheses
- Lucene regex patterns are wrapped in forward slashes: `/pattern/`
- Range queries use `>=`, `<=`, `>`, `<` for numbers, dates, and other comparable values
- Fuzzy search (`~N`) finds spelling variations; proximity search (`"phrase"~N`) finds nearby terms
- ECS standardizes field names across data sources for consistent cross-source querying
- EQL is used for multi-event correlation in detection rules, not for ad-hoc Kibana searches

---

## References

- [Kibana Query Language](https://www.elastic.co/guide/en/kibana/current/kuery-query.html)
- [Lucene Query Syntax](https://lucene.apache.org/core/)
- [Elastic Common Schema](https://www.elastic.co/docs/reference/ecs/ecs-getting-started)
- [Event Query Language](https://www.elastic.co/docs/explore-analyze/query-filter/languages/eql)
- [TryHackMe](https://tryhackme.com)
