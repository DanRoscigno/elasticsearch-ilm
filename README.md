# elasticsearch-ilm

## Build an instance group in GCP

I used a five node non-autoscaling in a single GCP availability sone.  I used sall n-1 (3.75GB RAM) Centos 7 images.

## Setup virtual memory and max files

`sudo vi /etc/sysctl.d/100-elastic.conf`

```
vm.max_map_count=262144
```

`sudo vi /etc/security/limits.conf`

```
# elasticsearch
*       soft    nofile  65535
*       hard    nofile 65535
```

# Configure master nodes

I have five nodes.  I am requiring two master eligible nodes.  

The first node (`external-elastic-one`) has this config after following the [security blog](https://www.elastic.co/blog/tips-to-secure-elasticsearch-clusters-for-free-with-encryption-users-and-more):

```
node.name: external-elastic-one
#node.attr.box_type: warm
node.master: true 
node.data: true 
node.ingest: true 
node.ml: false 
xpack.ml.enabled: false 
discovery.zen.minimum_master_nodes: 2
network.host:  0.0.0.0
discovery.seed_hosts: ["roscigno-ilm-sr76", "roscigno-ilm-q8dx"]
cluster.initial_master_nodes: ["external-elastic-one", "external-elastic-two"]
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

Note: Come back and add info on security, or maybe ust link to the securing your cluster doc.

and the second master eligible:

```
node.name: external-elastic-two
#node.attr.box_type: warm
node.master: true 
node.data: true 
node.ingest: true 
node.ml: false 
xpack.ml.enabled: false 
discovery.zen.minimum_master_nodes: 2
network.host:  0.0.0.0
discovery.seed_hosts: ["roscigno-ilm-sr76", "roscigno-ilm-q8dx"]
cluster.initial_master_nodes: ["external-elastic-one", "external-elastic-two"]
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

node 5 is not master eligible or an ingest node.  It is a warm node (`node.attr.box_type: warm`)

```
node.name: external-elastic-five
node.attr.box_type: warm
node.master: false 
node.data: true 
node.ingest: false 
node.ml: false 
xpack.ml.enabled: false 
discovery.zen.minimum_master_nodes: 2
network.host:  0.0.0.0
discovery.seed_hosts: ["roscigno-ilm-sr76", "roscigno-ilm-q8dx"]
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

## Kibana

Note that I am running Kibana on `localhost` and not the hostname as I am going to use an SSH tunnel and map port `8601` on my laptop to port `5601` on the GCE VM

```
#server.host: "roscigno-ilm-sr76"
server.host: "localhost"

elasticsearch.hosts: ["http://roscigno-ilm-sr76:9200", "http://roscigno-ilm-q8dx:9200", "http://roscigno-ilm-6tml:9200"]

elasticsearch.username: "kibana"
elasticsearch.password: "kibanapassword"

# Disable reporting because of a Chromium issue on my VM
xpack.reporting.enabled: false
```

## Tunnel to GCP

This uses the Google `gcloud` command to ssh to a VM in my GCE projject and port forward port `8601` on my laptop to port `5601` on the remote VM.  See the Kibana config above, I have Kibana listening on the loopback (localhost) address.

```
gcloud compute ssh --project elastic-customer-success \
  --zone us-central1-a roscigno-ilm-sr76 \
  -- -L 8601:localhost:5601
```

## Transfer files

If you need to scp files from a GCE VM to your laptop:
```
gcloud compute scp \
  dan_roscigno@roscigno-ilm-sr76:/home/dan_roscigno/install/elasticsearch-7.5.2/config/elastic-certificates.p12 \
  . \
  --project elastic-customer-success --zone us-central1-a 
```

# Logstash

Dowload latest.

## Java
Reuse the JDK supplied with Elasticsearch
```
export JAVA_HOME=$HOME/install/elasticsearch-7.5.2/jdk
```

## Grab some logs
Sample logs used in [Logstash getting started]() are at https://download.elastic.co/demos/logstash/gettingstarted/logstash-tutorial.log.gz
