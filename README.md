# abbegelasticstack
##1. Getting Started with Logstash
U16
```
apt update && apt install -y openjdk-8-jre openjdk-8-jdk
wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb http://packages.elastic.co/logstash/2.4/debian stable main" | sudo tee -a /etc/apt/sources.list.d/logstash.list
apt update && apt install logstash
```

test (works under u16)
```
cd /opt/logstash/bin/
./logstash -e 'input { stdin { } } output { stdout {}}'
```
ctrl d to exit.
```
vim /etc/logstash/conf.d/logstash-sample.conf
```
edit
```
input {
  stdin {}
}
output {
  stdout {}
}
```
```
./logstash -f /etc/logstash/conf.d/logstash-sample.conf
```


##2. Getting Started with Elasticsearch
```
wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add - && echo "deb https://packages.elastic.co/elasticsearch/2.x/debian stable main" | sudo tee -a /etc/apt/sources.list.d/elasticsearch-2.x.list
apt update && apt install elasticsearch && update-rc.d elasticsearch defaults
```

```
service elasticsearch start
curl -X GET 'http://localhost:9200'
```

### Configuring Elasticsearch on Ubuntu 16.04.1 LTS
check health
```
curl 'localhost:9200/_cat/health?v'
```

put first index
```
curl -XPUT 'localhost:9200/sampleindex?pretty'
```
list indices
```
curl 'localhost:9200/_cat/indices?v'
```
delete the index
```
curl -XDELETE 'localhost:9200/sampleindex?pretty'
```
lsit all plugins
```
cd /usr/share/elasticsearch/
bin/plugin list
```



##3. Getting Started with Kibana
```
echo "deb http://packages.elastic.co/kibana/4.6/debian stable main" | sudo tee -a /etc/apt/sources.list.d/kibana.list && apt update && apt install kibana && update-rc.d kibana defaults 95 10
```
[update-rc.d cheat sheet](https://www.jamescoyle.net/cheat-sheets/791-update-rc-d-cheat-sheet)
######Exaplanation
```
update-rc.d apache2 defaults [START] [KILL]
```
The below command will start mysql first, then apache2. On shutdown, the kill will be the reverse of the start with apache2 being killed first and mysql second.
```
update-rc.d apache2 defaults 90 90
update-rc.d mysql defaults 10 10
```


configure using httpd/apache2
```
vim /etc/logstash/conf.d/01-webserver.conf
```
edit
```
input {
  file {
    path => "/var/httpd/logs/access_log"   #for u16 "/var/log/apache2/access_log"
    start_position => "beginning"
  }
}
filter {
if [type] == "apache-access"
{
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
  }
  date {
    match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
}
output {
  elasticsearch {
    hosts => ["localhost:9200"]
  }
  stdout { codec => rubydebug }
}
```
###Kibana Plug-ins
```
cd /opt/kibana/bin/
./kibana plugin -i elasticsearch/graph/latest
```
####remove
```
./kibana plugin --remove graph
```


##6. Graphite Monitoring and Graphs
c7
```
yum check-update
yum update && yum install -y epel-release
```
```
yum install pycairo django bitmap bitmap-fonts mod_wsgi python-pip python-devel gcc httpd
```
```
pip install --upgrade pip
```
```
pip install carbon
 pip install whisper
pip install graphite-web
pip install "django-tagging<0.4"
```

then
```
cp /opt/graphite/conf/carbon.conf.example /opt/graphite/conf/carbon.conf
```




##7. Configuring Elasticsearch Watcher
```
cd /usr/share/elasticsearch/
bin/plugin install license
bin/plugin install watcher
service elasticsearch restart
```
To test whether Watcher is working, run the following commands:
```
curl -XGET 'http://localhost:9200/_watcher/stats?pretty'
```

or
```
curl -XPUT 'http://localhost:9200/_watcher/watch/logstash_watch' -d '{
  "trigger" : {
    "schedule" : { "interval" : "15s" }
  },
  "input" : {
    "search" : {
      "request" : {
        "indices" : [ "logstash" ],
        "body" : {
          "query" : {
            "match" : { "message": "error" }
          }
        }
      }
    }
  }
,
"condition" : {
    "compare" : { "ctx.payload.hits.total" : { "gt" : 0 }}
  }
,
"actions" : {
    "log_error" : {
      "logging" : {
        "text" : "Found {{ctx.payload.hits.total}} errors in the logs"
      }
    }
  }
}    
```

###Installing Sense Editor
```
cd /opt/kibana/
bin/kibana plugin --install elastic/sense
```
open
```
http://yourip:5601/app/sense
```
search
```
GET .watch_history-2016.09.29
/_search
        "query": {
        "match_all": {}
      }
```

###Deleting Watches
```
curl -XDELETE 'http://localhost:9200/_watcher/watch/logstash_watch'
```




##8. Securing the ELK Stack with Shield
install shield on u16
```
cd /usr/share/elasticsearch/ && bin/plugin install license && bin/plugin install shield
```
###IP Filtering
allow
```
shield.transport.filter.allow: "192.168.1.1"
```
disallow
```
shield.transport.filter.deny: "192.168.1.10"
```
allow rules will always appear first, followed by deny rules.
```
shield.transport.filter.allow: "192.168.1.1"
shield.transport.filter.deny: "192.168.1.10"
```
lists:
```
shield.transport.filter.allow: ["192.168.1.1",  "192.168.1.11", "192.168.1.21",  192.168.1.99"]
```
all connections that are not allowed:
```
shield.transport.filter.deny: _all
```
hostname filtering
```
shield.transport.filter.deny: '*.yahoo.com'
```

###Adding a User to Shield
```
vim /etc/elasticsearch/shield/roles.yml
```
create user
```
cd /usr/share/elasticsearch/
bin/shield/esusers useradd es_admin -r admin -p admin123  
```
login elasticsearch using user
```
curl -u es_admin -XGET 'http://localhost:9200/'
```

kibana
```
vim /opt/kibana/config/kibana.yml
```
edit
```
elasticsearch.username: "es_admin"
elasticsearch.password: "admin123"
```

###Configuring Logstash to Use Authentication
```
output {
  elasticsearch {
    hosts => ["localhost:9200"]
   user => es_admin
    password => admin123
  }
}
```
###Configuring Filebeat to Use Authentication
```
vim /etc/filebeat/filebeat.yml
```
edit
```
protocol: "http"
username: "es_admin"
password: "admin123"
```

```
cp storage-schemas.conf.example storage-schemas.conf
```


```
cp storage-schemas.conf.example storage-schemas.conf
vim storage-schemas.conf
```

```
[carbon]
pattern = ^carbon\.
retentions = 60:90d
[default_1min_for_1day]
pattern = .*
retentions = 60s:1d
```
```
 
