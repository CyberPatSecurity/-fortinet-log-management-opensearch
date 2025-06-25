# Fortinet Log Management with OpenSearch

## Overview
This project sets up a complete Fortinet log management pipeline using open-source tools:
- **Rsyslog** to receive logs from a Fortinet firewall
- **Logstash** to parse and forward logs
- **OpenSearch** to store and search logs
- **OpenSearch Dashboards** for visualization

## Architecture
```
Fortinet Firewall --> Rsyslog (UDP 514) --> Logstash --> OpenSearch --> OpenSearch Dashboards
```

## Components

### 1. Rsyslog
Receives syslog messages from the Fortinet firewall and writes them to `/var/log/fortinet.log`.

Configuration example:
```bash
module(load="imudp")
input(type="imudp" port="514")

template(name="FortinetFormat" type="string"
         string="/var/log/fortinet.log")

if ($fromhost-ip startswith "10.160.99.") then {
    action(type="omfile" file="/var/log/fortinet.log")
}
```

### 2. Logstash
Parses the logs and sends them to OpenSearch.

`logstash/conf.d/fortinet-to-opensearch.conf`:
```conf
input {
  file {
    path => "/var/log/fortinet.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    type => "syslog"
  }
}

filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:timestamp} %{SYSLOGHOST:hostname} %{DATA:program}: %{GREEDYDATA:msg}" }
    }
    date {
      match => [ "timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
      target => "@timestamp"
    }
  }
}

output {
  opensearch {
    hosts => ["https://localhost:9200"]
    index => "fortinet-logs-%{+YYYY.MM.dd}"
    user => "admin"
    password => "admin"
    ssl => true
    ssl_certificate_verification => false
  }
}
```

### 3. OpenSearch
Stores and indexes the parsed logs.

### 4. OpenSearch Dashboards
Used to explore logs via the `fortinet-logs-*` index pattern, with `@timestamp` as the time field.

## Result
Over 1.6 million logs parsed and searchable with visual dashboards ready for filtering and analytics.

## Author
Patrizio Egidi

## License
MIT
