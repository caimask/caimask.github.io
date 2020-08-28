---
layout: post
title: Logstash
tags: [Logstash, MySQL, Kafka]
author: caimask
---

## Logstash 를 Data Feeding System

### MySQL 데이터의 입력

### Kafka 로 데이터 출력

5 초 마다 DB 에 select 를 해서 전달된 결과를 kafka 의 특정 Topic 으로 Produce 하는 작업을 수행해 준다.

나중에 binlog 기반으로 데이터를 kafka 에 전달 할 수 있는 방법을 공부해 봐야 겠다.

```
input {
  jdbc {
    jdbc_driver_library => "/etc/logstash/mysql-connector-java-5.1.39-bin.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://log-db.test.net:3306/ggevent?useUnicode=true&characterEncoding=utf8"
    jdbc_user => "log_admin"
    jdbc_password => "Log19)#"
    use_column_value => true
    tracking_column => "id"
    schedule => "*/5 * * * * *"
    statement => "SELECT id, json_extract(`event`, '$') as event, json_unquote(json_extract(`event`,'$.table.id')) as table_id FROM hand WHERE id > :sql_last_value limit 10000"
    last_run_metadata_path => "/etc/logstash/last_sql_index"
  }
}
output {
  kafka {
    topic_id => "Data-HandServer"
    message_key => "%{table_id}"
    codec => plain {
      format => "%{event}"
    }
    bootstrap_servers => "internal.kafka01.test.net:6667,internal.kafka03.test.net:6667"
  }
}
```

