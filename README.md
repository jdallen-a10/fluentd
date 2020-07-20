# fluentd
A fluentd container with build-in support ElasticSearch.<br>
<br>
[Fluentd](<https://www.fluentd.org/>) is a [CNCF Project](<https://landscape.cncf.io/category=observability-and-analysis&format=card-mode&grouping=category&selected=fluentd>) created to take in many different logging and data sources and provide a unified logging layer between these backend systems and other solutions that can use the data. Multiple nodes and of different types can be combined into a datastream for the different analytical and observability solutions. As of this writing, there are [42 different solutions](<https://www.fluentd.org/dataoutputs>) that Fluentd can output to!

In this example, we are going to create a custom ``fluentd`` configuration to push our log records over to ElasticSearch for storage and later retrieval by Kibana.

### Build the Container

One nice thing about containers, is that you can take existing ones, and add on to them to create your own custom version.  In this case, we are taking a default `fluentd` container and adding a couple of plug-ins to support EleasticSearch:

```dockerfile
FROM fluent/fluentd:latest

USER root

RUN apk add --no-cache --update --virtual .build-deps \
  sudo build-base ruby-dev \
  && sudo gem install fluent-plugin-elasticsearch \
  && sudo gem install fluent-plugin-record-modifier \
  && sudo gem install fluent-plugin-concat \
  && sudo gem install fluent-plugin-multi-format-parser \
  && sudo gem sources --clear-all \
  && apk del .build-deps \
  && rm -rf /home/fluent/.gem/ruby/2.5.0/cache/*.gem

```

Build this container:

```bash
docker build -t a10networks/fluentd:latest .
```

Next we need to create the custom configuration to take in the Thunder log records and push them over to the ElasticSearch node:

```xml
# -- fluent-to-ES.conf
# Based on this: https://blog.bitsrc.io/setting-up-a-logging-infrastructure-in-nodejs-ec34898e677e
#
# Recieve events from 24224/tcp
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

# Fluentd will decide what to do here if the event is matched
# In our case, we want all the data to be matched hence **
<match **>
# We want all the data to be copied to elasticsearch using the in-built
# copy output plugin https://docs.fluentd.org/output/copy
  @type copy
  <store>
  # We want to store our data to elastic search using out_elasticsearch plugin
  # https://docs.fluentd.org/output/elasticsearch. See Dockerfile for installation
    @type elasticsearch
    time_key timestamp_ms
    host 0.0.0.0
    port 9200
    # Use conventional index name format (logstash-%Y.%m.%d)
    logstash_format true
    # We will use this when kibana reads logs from ES
    logstash_prefix fluentd
    logstash_dateformat %Y-%m-%d
    flush_interval 1s
    reload_connections false
    reconnect_on_error true
    reload_on_failure true
  </store>
</match>

```
You will want to change the ``host 0.0.0.0`` line to match either the IP address of the ElasticSearch node, or its FQDN.

### Start the Custom Fluentd Container

Now you are ready to start the ``fluentd`` container:

```bash
docker run -d --restart=always \
  --name=fluentd \
  -p 24224:24224 \
  -p 24224:24224/udp \
  -v $(pwd)/fluentd:/fluentd/log \
  -v $(pwd)/fluent-to-ES.conf:/fluentd/etc/fluent.conf \
  a10networks/fluentd:latest
```
