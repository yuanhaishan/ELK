# ELK
1.拉取镜像
```
docker pull docker.elastic.co/elasticsearch/elasticsearch:6.6.2
docker pull docker.elastic.co/kibana/kibana:6.6.2
docker pull docker.elastic.co/beats/filebeat:6.6.2
docker pull docker.elastic.co/logstash/logstash:6.6.2
```
2.部署es
```
docker run -d \
    --user root \
    -p 127.0.0.1:9200:9200 \
    -p 9300:9300 \
    --log-driver json-file \
    --log-opt max-size=10m \
    --log-opt max-file=3 \
    --name elasticsearch \
    --restart=always \
    -e "discovery.type=single-node" \
    -v /data/elasticsearch:/usr/share/elasticsearch/data \
    docker.elastic.co/elasticsearch/elasticsearch:6.6.2
    
chmod 777 -R /data/elasticsearch
```
3.部署Logstash 
```mkdir -pv /data/conf
cat > /data/conf/logstash.conf << "EOF"
input {
  beats {
    host => "0.0.0.0"
    port => "5043"
  }
}
filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:log-timestamp} %{LOGLEVEL:log-level} %{JAVALOGMESSAGE:log-msg}" }
  }
  mutate {
#    remove_field => ["message"]
    remove_field => ["beat"]
   }
}
output {
  stdout { codec => rubydebug }
  elasticsearch {
        hosts => [ "elasticsearch:9200" ]
        index => "%{containername}-%{+YYYY.MM.dd}"
  }
}
EOF

docker run -p 5043:5043 -d \
    --user root \
    --log-driver json-file \
    --log-opt max-size=10m \
    --log-opt max-file=3 \
    --name logstash \
    --link elasticsearch \
    --restart=always \
    -v /data/conf/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
    docker.elastic.co/logstash/logstash:6.6.2
```
4.部署filebeat
```
cat > /data/conf/filebeat.yml << "EOF"
filebeat.inputs:
- type: log
  paths:
   - '/var/lib/docker/containers/*/*.log'
# json解码
  json.add_error_key: true
  json.overwrite_keys: true
  json.message_key: log
# 多行合并
#  multiline:
#    pattern: ^\d{4}
#    negate: true
#    match: after
#
# 排除或删除通配符行
#  exclude_lines: ['^DBG']
#  include_lines: ['ERROR', 'WARN', 'INFO']
# 排除通配文件
#  exclude_files: ['\.gz$']
#
# 添加docker元数据
  processors:
    - add_docker_metadata:
        match_source: true
# 选取添加docker元数据的层级
        match_source_index: 4
# 处理器重命名和删除无用字段
processors:
- rename:
    fields:
     - from: "json.log"
       to: "message"
     - from: "docker.container.name"
       to: "containername"
     - from: "log.file.path"
       to: "filepath"
- drop_fields:
    fields: ["docker","metadata","beat","input","prospector","host","source","offset"]
# 日志输出
output.logstash: # 输出地址
  hosts: ["192.168.30.42:5043"]
#output.elasticsearch:
#  hosts: ["192.168.30.42:9200"]
#  protocol: "http"
#output.file:
#  path: "/tmp"
#  filename: filebeat.out
#output.console:
#  pretty: true
EOF


docker run -d \
    --name filebeat \
    --user root \
    --log-driver json-file \
    --log-opt max-size=10m \
    --log-opt max-file=3 \
    --restart=always \
    -v /data/conf/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro \
    -v /var/lib/docker/containers/:/var/lib/docker/containers/ \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    docker.elastic.co/beats/filebeat:6.6.2
```
5.部署kibana

```cat > /data/conf/kibana.yml << "EOF"
# Default Kibana configuration from kibana-docker.

server.name: kibana
server.host: "0"
elasticsearch.hosts: http://elasticsearch:9200
xpack.monitoring.ui.container.elasticsearch.enabled: true

docker run -p 5601:5601 -d \
    --user root \
    --log-driver json-file \
    --log-opt max-size=10m \
    --log-opt max-file=3 \
    --name kibana \
    --link elasticsearch \
    --restart=always \
    -e ELASTICSEARCH_URL=http://elasticsearch:9200 \
    -v /data/conf/kibana.yml:/usr/share/kibana/config/kibana.yml \
    -v /data/elastalert-kibana-plugin:/usr/share/kibana/plugins/elastalert-kibana-plugin \
    docker.elastic.co/kibana/kibana:6.6.2
```
