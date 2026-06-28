# TryHackMe Write-Up

# Elastic: Using Logstash

| **Field** | **Value** |
|-----------|-----------|
| Platform | TryHackMe |
| Module | Advanced ELK |
| Completed by | Aysu Nezirova |
| Date | June 27, 2026 |
| Points earned | 128 |
| Tasks completed | 9 |

---

## Overview

This room covers the installation, configuration, and integration of Logstash as part of the ELK Stack. It includes creating real Logstash pipelines using input, filter, and output plugins, and analyzing the results in Kibana.

### Logstash Pipeline

```
Input  →  Filter  →  Output
```

| **Part** | **Role** |
|----------|----------|
| Input | Where data comes from (files, network, agents) |
| Filter | How data is processed and normalized |
| Output | Where processed data is sent (Elasticsearch, etc.) |

---

## Task 1 — Introduction

No questions. Learning objectives reviewed and ready to proceed.

---

## Task 2 — Installing Logstash

Logstash was installed using the .deb package:

```bash
dpkg -i logstash.deb
```

**Q:** Which Elastic stack component offers advanced filtering, parsing, and enrichment of log data?  
**A:** `Logstash`

**Q:** Which Logstash version number did you just install?  
**A:** `9.2.4`

---

## Task 3 — How Logstash Works

Logstash uses plugins for each pipeline stage. Key filter plugins:

| **Plugin** | **Role** |
|------------|----------|
| Grok | Parses unstructured logs using custom patterns |
| Mutate | Renames, replaces, modifies event fields |
| Date | Parses and formats timestamps |
| Prune | Removes fields using whitelist/blacklist |
| Drop | Drops entire events from the pipeline |
| KV | Parses key=value pairs into separate fields |

**Q:** Which plugin parses unstructured log data using custom patterns?  
**A:** `Grok`

**Q:** Which filter plugin is used to perform mutation of event fields?  
**A:** `Mutate`

---

## Task 4 — Configuring Logstash

Pipeline config files are stored in `/etc/logstash/conf.d/`. The `=>` operator defines field-value pairs. Example pipeline:

```ruby
input {
  tcp {
    port => 5456
    codec => json
  }
}

filter {
  json {
    source => "message"
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "your_index_name"
  }
}
```

**Q:** Check the File input plugin documentation. Which configuration option is required?  
**A:** `path`

**Q:** Which field does the JSON filter plugin read the contents of?  
**A:** `message`

---

## Task 5 — Configuring Inputs

Logstash supports multiple input plugins:

| **Plugin** | **Default Port** | **Use Case** |
|------------|-----------------|--------------|
| File | — | Local log files |
| Beats | 5044 | Filebeat, Winlogbeat agents |
| TCP | 5000 | Custom apps, network devices |
| UDP | 514 | Syslog, network devices |
| HTTP | 8080 | Webhooks, REST APIs |

**Q:** Which Logstash input plugin would you use to monitor and ingest the locally hosted syslog file?  
**A:** `file`

**Q:** Check the Google Cloud Storage input plugin documentation. What is the one required input configuration setting?  
**A:** `bucket_id`

---

## Task 6 — Filter Configurations

### Mutate Filter

```ruby
filter {
  mutate {
    add_field => { "new_field" => "new_value" }
    rename    => { "IPAddress" => "client_ip" }
  }
}
```

### Grok Filter

```ruby
filter {
  grok {
    match => { "message" => "%{IP:client_ip} %{GREEDYDATA:msg}" }
  }
}
```

### Prune Filter

```ruby
filter {
  prune {
    whitelist_names => ["field1", "field2"]
  }
}
```

### KV Filter

```ruby
filter {
  kv {
    field_split => "&"
    value_split => "="
  }
}
# user=alice&status=active  =>  { "user": "alice", "status": "active" }
```

**Q:** Which filter plugin is used to rename, replace, and modify event fields?  
**A:** `mutate`

**Q:** Which filter plugin is used to remove fields from events?  
**A:** `prune`

**Q:** Change the mutate filter configuration to rename the field `src_ip` to `source_ip`.  
**A:** `rename => { "src_ip" => "source_ip" }`

---

## Task 7 — Defining the Destination

### Elasticsearch Output

```ruby
output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "your_index_name"
  }
}
```

### Stdout Output (debugging)

```ruby
output {
  stdout { }
}
```

### RabbitMQ Output

```ruby
output {
  rabbitmq {
    host        => "localhost"
    exchange    => "my_exchange"
    routing_key => "my_routing_key"
  }
}
```

**Q:** Which Logstash output plugin is used to print events to the console?  
**A:** `stdout`

**Q:** What are the values of `setting1`, `setting2`, and `setting3` for the syslog output?  
**A:** `host, port, protocol`

---

## Task 8 — Putting Pipelines Into Practice

Goal: Monitor `/var/log/auth.log` → parse with Logstash → send to Elasticsearch → analyze in Kibana.

### Step 1 — Create configuration file

```bash
sudo nano /etc/logstash/conf.d/auth.conf
```

### Step 2 — Input configuration

```ruby
input {
  file {
    path           => "/var/log/auth.log"
    start_position => "beginning"
    sincedb_path   => "/dev/null"
  }
}
```

### Step 3 — Filter + Output configuration

```ruby
filter {
  grok {
    match => {
      "message" => "^%{TIMESTAMP_ISO8601:timestamp} %{NOTSPACE:[host][name]} %{WORD:program}(?:\[%{NUMBER:pid}\])?: %{GREEDYDATA:msg}"
    }
  }
  date {
    match  => ["timestamp", "ISO8601"]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts                 => ["https://localhost:9200"]
    user                  => "elastic"
    password              => "pn00IuML9u43_yKb688y"
    index                 => "auth-logs-%{+YYYY.MM.dd}"
    ssl_verification_mode => "none"
    ssl_enabled           => true
  }
}
```

### Step 4 — Grant permissions and start Logstash

```bash
usermod -aG adm logstash      # give Logstash read access to auth.log
systemctl restart logstash    # restart Logstash
systemctl status logstash     # verify: active (running)
```

### Step 5 — Test with new user creation

```bash
useradd -m detective_elastic
passwd detective_elastic
```

**Q:** Which two filters did you use in your `auth.conf` configuration file?  
**A:** `grok, date`

**Q:** What is the `program` field value for the latest events?  
**A:** `useradd`

**Q:** What is the full `msg` field value for the password change event?  
**A:** `pam_unix(passwd:chauthtok): password changed for detective_elastic`

---

## Task 9 — Conclusion

Room completed successfully! 🎉

---

## Key Takeaways

- Logstash pipeline = **Input → Filter → Output**
- `grok` is the most powerful filter for parsing unstructured logs
- `date` filter ensures correct timestamp handling in Kibana
- `sincedb_path => "/dev/null"` resets file tracking on each restart
- `usermod -aG adm logstash` is required to read `/var/log/auth.log`
- Index naming with `%{+YYYY.MM.dd}` creates daily indices automatically
- Default ports: Beats=5044, Elasticsearch=9200, Kibana=5601, UDP/Syslog=514

---

## References

- [Logstash Documentation](https://www.elastic.co/docs/reference/logstash)
- [Input Plugins](https://www.elastic.co/docs/reference/logstash/plugins/plugins-inputs-file)
- [Filter Plugins](https://www.elastic.co/docs/reference/logstash/plugins/plugins-filters-grok)
- [Output Plugins](https://www.elastic.co/docs/reference/logstash/plugins/plugins-outputs-elasticsearch)
- [TryHackMe](https://tryhackme.com)
