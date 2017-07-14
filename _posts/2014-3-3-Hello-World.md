---
layout: post
title: Logstash: Parsing ELB Access Logs from S3
---

Built using Elasticsearch Logstash Kibana (ELK) Stack
* Elasticsearch 5.x
* LogStash 5.5.0
* Kibana 5.5.0

Alot of research and hours of trial by error were put into getting a working configuration of consuming AWS ELB Access Logs from S3 using Logstash to transform data into Elasticsearch and analyze the data using Kibana. The below configuration is the latest version that works for me and may save you many hours of frustration if you are working on the same type of project. 


```

input {
  s3 {
    type => "elb-access-log"
    bucket => "s3_bucket_name_goes_here"
    region => "us-west-2"
    access_key_id => "access_key_id_value_goes_here"
    secret_access_key => "secret access_key_value_goes_here"
    sincedb_path => "/tmp/.prod_s3_elb_logs_since.db"
  }
}

filter {

  if [type] == "elb-access-log" {

    grok {
      match => [ "message", "%{TIMESTAMP_ISO8601:timestamp} %{NOTSPACE:elb} %{IP:clientip}:%{INT:clientport:int} (?:(%{IP:backendip}:?:%{INT:backendport:int})|-) %{NUMBER:request_processing_time:float} %{NUMBER:backend_processing_time:float} %{NUMBER:response_processing_time:float} (?:-|%{INT:elb_status_code:int}) (?:-|%{INT:backend_status_code:int}) %{INT:received_bytes:int} %{INT:sent_bytes:int} \"%{ELB_REQUEST_LINE}\" \"(?:-|%{DATA:user_agent})\" (?:-|%{NOTSPACE:ssl_cipher}) (?:-|%{NOTSPACE:ssl_protocol})" ]
    }

    date {
      match => [ "timestamp", "ISO8601" ]
    }

    fingerprint {
       source => ["message"]
       target => "[@metadata][fingerprint]"
       method => "MURMUR3"
    }


    mutate {
      add_field => { "indexname" => "elb-%{elb}" }
    }

    mutate {
      lowercase => [ "indexname" ]
    }
  }
}

output {
   elasticsearch {
      hosts => ["172.31.45.27:9200"]
      document_id => "%{[@metadata][fingerprint]}"
   }
}
```