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
