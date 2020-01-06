# Overview
I am playing around with Confluent Replicator with two Apache Kafka running in Confluent Cloud.
* Source is in GCP - I call this cluster cmbigquery (the source-cluster)
* Destination is in AWS - I call the cluster adidas (the destination-cluster)
My environment looks like this
![alt text](images/mysetup.png)

I did prepare a couple of useful plays:
* Replicator Executable - easy configuration
* Replicator as standalone connector - only for Dev/Test
* Replicator with Connect Worker - JSON data and Avro data - recommend setup for production
* Security stuff
* Schema Translation
* Finally GKE on k8s Deployment for Replicator with replication into CCloud


# Prereqs
* create 2 Confluent Clusters
* Install ccloud cli on your desktop
* have some tools available jq, curl
* Have Confluent Platform 5.3.1 and 5.4 installed on your desktop
* copy files from this githib project on your computer and change the properties files with the information of your clusters
* on source cluster create topic cmorders: ccloud kafka topic create cmorders

# 1 Replicator executable
```bash
cd replicatorplayground/
```
source consumer is cmbigquery (GCP) in CCLOUD
```bash
cat cmbigquery.properties
```
destination producer to cloud adidas
```bash
cat adidas.properties
```
replication.properties need no entries
list topic from source ccloud
```bash
kafka-topics --list --bootstrap-server source-cluster:9092 --command-config cmbigquery.properties
kafka-console-consumer --topic cmorders --consumer.config cmbigquery.properties --bootstrap-server source-cluster:9092 --from-beginning
```
List topic from destination ccloud
```bash
kafka-topics --list --bootstrap-server destination-cluster:9092 --command-config adidas.properties
```
Start a new Terminal 1 and produce data:
```bash
kafka-console-producer --broker-list source-cluster:9092 --producer.config cmbigquery.properties --topic cmorders
{ "Name": "Table", "Count": 1 }
{ "Name": "iPad", "Count": 1 }
{ "Name": "iPhone", "Count": 1 }
```
In a new Terminal 2 run replicator (no schema)
```bash
# ccloud to ccloud
replicator --cluster.id replicator --consumer.config cmbigquery.properties --producer.config adidas.properties --whitelist cmorders
```
check if data was replicated in a new Termina 3:
```bash
kafka-console-consumer --topic cmorders --consumer.config adidas.properties --bootstrap-server destination-cluster:9092
```
For cleanup, delete replicated topic in adidas 
```bash
kafka-topics --delete --bootstrap-server destination-cluster:9092 --topic cmorders --command-config adidas.properties
kafka-topics --list --bootstrap-server destination-cluster:9092 --command-config adidas.properties
```
and Stop the Replicator.

# 2 Standalone Connect Replicator
connect-standalone properties should use with destitation (adidas) source (cmbigquery):
```bash
cat connect-standalone.properties
```
replicator properties
```bash
cat replicator.properties
```
Produce Data into source (cmbigquery)
```bash
kafka-console-producer --broker-list source-cluster:9092 --producer.config cmbigquery.properties --topic cmorders
{ "Name": "2Table", "Count": 1 }
{ "Name": "2iPad", "Count": 1 }
{ "Name": "2iPhone", "Count": 1 }
```
Run the Replicator in new Terminal 2
```bash
connect-standalone ./connect-standalone.properties ./replicator.properties
```
You can check in [ccloud portal](https://confluent.cloud/login) in destination (adidas)
or check on prompt:
```bash
# check if data was replicated
kafka-console-consumer --topic cmorders.replica --consumer.config adidas.properties --bootstrap-server destination-cluster:9092 
```
Clean-Up environment:
```bash
### Clean (adidas destination)
ccloud login
ccloud kafka topic list
for i in $(ccloud kafka topic list | grep _confluent); do echo ${i}; ccloud kafka topic delete ${i}; done
for i in $(ccloud kafka topic list | grep connect); do echo ${i}; ccloud kafka topic delete ${i}; done
ccloud kafka topic cmorders.replica
# delete offset file
rm connect.offsets 
```
Stop connector.

#  3. Replicator as distributed connector
Show distributed worker setup
```bash
cat connect-distributed.properties
```
Show replicator setup
```bash
cat replicator.json
```
install replicator via confluent-hub client
```bash
confluent-hub install confluentinc/kafka-connect-replicator:5.3.1
```
or use plug-in from CP, set plug.in-path in connect-distributed.properties
```bash
# in my case
plugin.path=/software/confluent/share/java
```
Start Connect Worker in Terminal 1
```bash
connect-distributed connect-distributed.properties
```

in terminal 2  check connect worker and run Replicator:
```bash
curl localhost:8083/ | jq
# connector plugins
curl localhost:8083/connector-plugins | jq
# Run Replicator
curl -X POST -d @replicator.json  http://localhost:8083/connectors --header "content-Type:application/json"
# Running connectors
curl localhost:8083/connectors
# Status Replicator Connector
curl localhost:8083/connectors/replicate-topic/status | jq
```
Produce data to source
```bash
kafka-console-producer --broker-list source-cluster:9092 --producer.config cmbigquery.properties --topic cmorders
kafka-console-producer --broker-list source-cluster:9092 --producer.config cmbigquery.properties --topic cmorders
{ "Name": "2Table", "Count": 1 }
{ "Name": "2iPad", "Count": 1 }
{ "Name": "2iPhone", "Count": 1 }
```
In terminal 3 check if data was replicated
```bash
kafka-console-consumer --topic cmorders.replica --consumer.config adidas.properties --bootstrap-server destination-cluster:9092
```
Cleanup:
```bash
### Clean (adidas destination)
ccloud login
ccloud kafka topic list
for i in $(ccloud kafka topic list | grep _confluent); do echo ${i}; ccloud kafka topic delete ${i}; done
for i in $(ccloud kafka topic list | grep connect); do echo ${i}; ccloud kafka topic delete ${i}; done
ccloud kafka topic delete cmorders.replica
```
Stop Connector

#  3.1 Replicator as distributed connector with AVRO and SR
Show distributed worker setup
```bash
cat connect-avro_distributed.properties
```
Show replicator setup
```bash
cat replicator_avro.json
```
install replicator
```bash
confluent-hub install confluentinc/kafka-connect-replicator:5.3.1
# or use plug-in from CP, set plug.in-path in connect-distributed.properties
plugin.path=/software/confluent/share/java
```
Create topic in Source cluster
```bash
kafka-topics --create --bootstrap-server source-cluster:9092 \
--replication-factor 3 \
--partitions 10 --topic cmorders_avro --command-config cmbigquery.properties
```
list topics
```bash
kafka-topics --list --bootstrap-server source-cluster:9092 --command-config cmbigquery.properties
```
Make Schema Registry clean
```bash
curl -X GET -u SRKEY:SRSECRET https://SR-URL/subjects/
curl -X DELETE -u SRKEY:SRSECRET https://SR-URL/subjects/cmorders_avro-value
```
check schema
```bash
curl -u SRKEY:SRSECRET https://SR-URL/subjects/
curl --silent -X GET -u SRKEY:SRSECRET https://SR-URL/subjects/cmorders_avro-value/versions/latest | jq 
```
produce data to source-cluster in AVRO:
```bash
kafka-avro-console-producer --broker-list source-cluster:9092 --topic cmorders_avro --producer.config cmbigquery.properties \
 --property value.schema='{"type":"record","name":"schema","fields":[{"name":"name","type":"string"},{"name":"count", "type": "int"}]}' \
 --property basic.auth.credentials.source=USER_INFO \
 --property schema.registry.url=https://SR-URL \
 --property schema.registry.basic.auth.user.info=SRKEY:SRSECRET
{"name":"Apple Magic Mouse","count":1}
{"name":"Mac Book Pro","count":1}
```
check schema again
```bash
curl -u SRKEY:SRSECRET https://SR-URL/subjects
curl --silent -X GET -u SRKEY:SRSECRET https://SR-URL/subjects/cmorders_avro-value/versions/latest | jq 
```
In terminal 1 start Connect Worker
```bash
connect-distributed connect-avro_distributed.properties
```
In rerminal 2 check connect worker
```bash
curl localhost:8083/ | jq
# connector plugins
curl localhost:8083/connector-plugins | jq
```
Run Replicator
```bash
#curl -X DELETE http://localhost:8083/connectors/replicate-topic
curl -X POST -d @replicator_avro.json  http://localhost:8083/connectors --header "content-Type:application/json"
```
check Running connectors
```bash
curl localhost:8083/connectors
# Status Replicator Connector
curl localhost:8083/connectors/replicate-topic/status | jq
```
Produce data to source
```bash
kafka-avro-console-producer --broker-list source-cluster:9092 --topic cmorders_avro --producer.config cmbigquery.properties \
 --property value.schema='{"type":"record","name":"schema","fields":[{"name":"name","type":"string"},{"name":"count", "type": "int"}]}' \
 --property basic.auth.credentials.source=USER_INFO \
 --property schema.registry.url=https://SR-URL \
 --property schema.registry.basic.auth.user.info=SRKEY:SRSECRET
{"name":"Apple Magic Mouse","count":1}
{"name":"Mac Book Pro","count":1}
```
In terminal 3 check if data was replicated
```bash
kafka-avro-console-consumer --bootstrap-server destination-cluster:9092 --topic cmorders_avro \
 --consumer.config adidas.properties \
 --property basic.auth.credentials.source=USER_INFO \
 --property schema.registry.url=https://SR-URL \
 --property schema.registry.basic.auth.user.info=SRKEY:SRSECRET
```
Check in ccloud UI if schema is on Topic cmorders_avro, chema should be set.
Now, clean up destination:
```bash
# delete topic
kafka-topics --delete --topic cmorders_avro --bootstrap-server destination-cluster:9092 --command-config adidas.properties
### Clean (adidas destination)
ccloud login
ccloud environment use t6463
ccloud kafka cluster use lkc-4jko2
ccloud kafka topic list
for i in $(ccloud kafka topic list | grep _confluent); do echo ${i}; ccloud kafka topic delete ${i}; done
for i in $(ccloud kafka topic list | grep connect); do echo ${i}; ccloud kafka topic delete ${i}; done
```
and finally stop the Connector.

# 4. Security 
Follow Replicator Security Workshop [here](https://github.com/confluentinc/examples/tree/5.3.1-post/replicator-security)

# 5. Schema Translation
see [environment](https://github.com/confluentinc/examples/tree/5.3.1-post/replicator-schema-translation)
running the [demo](https://docs.confluent.io/current/tutorials/examples/replicator-schema-translation/docs/index.html?utm_source=github&utm_medium=demo&utm_campaign=ch.examples_type.community_content.replicator-schema-translation)

# 6. Performance Testing  
Very simple Test
* a. start replicator
* b. one terminal produce to src
* c. other terminal consume from dest
* brew install dateutils

cd replicatorplayground
terminal 1, Start Worker
```bash
connect-distributed connect-distributed.properties
```
terminal 2, Start Replicator
```bash
curl -X POST -d @replicator.json  http://localhost:8083/connectors --header "content-Type:application/json"
```
Consume Data from destination
```bash
kafka-console-consumer --topic cmorders.replica --consumer.config adidas.properties --bootstrap-server destination-cluster:9092 --property print.timestamp=true
```
Consume Data from source
```bash
kafka-console-consumer --topic cmorders.replica --consumer.config cmbigquery.properties --bootstrap-server source-cluster:9092 --property print.timestamp=true
```

terminal 3:Produce Data
```bash
for i in `seq 1 10`; do echo "{ \"Name\": \"Table\", \"Count\": ${i} }" | kafka-console-producer --broker-list source-cluster:9092 --producer.config cmbigquery.properties --topic cmorders; date +%s;  done
```
check last timestamp consumer destination with consumer source cluster
```bash
ddiff -i '%s' 1578226436479 1578226436474
```
clean (adidas destination)
```bash
ccloud login
ccloud kafka topic list
for i in $(ccloud kafka topic list | grep _confluent); do echo ${i}; ccloud kafka topic delete ${i}; done
for i in $(ccloud kafka topic list | grep connect); do echo ${i}; ccloud kafka topic delete ${i}; done
ccloud kafka topic delete cmorders.replica
```

Manual testing to produce to destination cluster. Execute three simple perftest against dest (adidas), to measure 
```bash
ccloud kafka topic create cmperftest
# unlimited throughput
kafka-producer-perf-test --topic cmperftest --num-records 10000000 --throughput -1 --producer.config adidas.properties --producer-props compression.type=snappy --record-size 200
# throttle to 300000 messages/sec
kafka-producer-perf-test --topic cmperftest --num-records 10000000 --throughput 300000 --producer.config adidas.properties --producer-props compression.type=snappy --record-size 200
# throttle to 100000 messages/sec
kafka-producer-perf-test --topic cmperftest --num-records 5000000 --throughput 100000 --producer.config adidas.properties --producer-props compression.type=snappy --record-size 100
# throttle to 10000 messages/sec
kafka-producer-perf-test --topic cmperftest --num-records 5000000 --throughput 10000 --producer.config adidas.properties --producer-props compression.type=snappy --record-size 100
# Delete perftest topic
ccloud kafka topic delete cmperftest
```
Do an end2endlatency test:
```bash
ccloud login
ccloud kafka topic list
ccloud kafka topic create cmtest
#  java kafka.tools.EndToEndLatency$ broker_list topic num_messages producer_acks message_size_bytes [optional] properties_file
# see also Jays blog https://gist.github.com/jkreps/c7ddb4041ef62a900e6c
kafka-run-class kafka.tools.EndToEndLatency destination-cluster:9092 cmtest 10000 all 200 adidas.properties
# Delete topic
ccloud kafka topic delete cmtest
```

## c3 replicator monitoring with Confluent Platform 5.4
Change your setup to 5.4, in my case I am doing this:
```bash
cd ~/software/
rm confluent
ln -s confluent-5.4.0/ confluent
export CLASSPATH=/software/confluent/share/java/kafka-connect-replicator/replicator-rest-extension-5.4.0-SNAPSHOT.jar
```
add in replicatorplayground/connect-distributed.properties 
```bash
  rest.extension.classes=io.confluent.connect.replicator.monitoring.ReplicatorMonitoringExtension
```
Start Connect Worker and Replicator
```bash
connect-distributed connect-distributed.properties
# check connect worker
curl localhost:8083/ | jq
# connector plugins
curl localhost:8083/connector-plugins | jq
# Run Replicator
curl -X POST -d @replicator.json  http://localhost:8083/connectors --header "content-Type:application/json"
# Running connectors
curl localhost:8083/connectors
# Status Replicator Connector
curl localhost:8083/connectors/replicate-topic/status | jq
curl localhost:8083/ReplicatorMetrics?
```
Start Control Center 5.4 from your local environment
```bash
# start C3
confluent local start
control-center-stop
control-center-start ./c3_ccloud_adidas.properties
http://localhost:9021
```
Now produce data in another terminal:
```bash
for i in `seq 1 100`; do echo "{ \"Name\": \"Table\", \"Count\": ${i} }" | kafka-console-producer --broker-list source-cluster:9092 --producer.config cmbigquery.properties --topic cmorders; date +%s;  done
```
In terminal 3, check if data was replicated
```bash
kafka-console-consumer --topic cmorders.replica --consumer.config adidas.properties --bootstrap-server destination-cluster:9092
```
In C3 a new feature for monitoring Replicator is enabled, go into [C3](http://localhost:9021) :
* Consumers replicate topic
* Replicators

Clean (adidas destination)
```bash
ccloud login
ccloud kafka topic list
for i in $(ccloud kafka topic list | grep _confluent); do echo ${i}; ccloud kafka topic delete ${i}; done
for i in $(ccloud kafka topic list | grep connect); do echo ${i}; ccloud kafka topic delete ${i}; done
ccloud kafka topic delete cmorders.replica
# Stop C3 cluster
control-center-stop
confluent local stop
# switch back to 5.3.1
cd ~/software
rm confluent
ln -s confluent-5.3.1/ confluent
```

Additional Performance Tests on-prem setup in AWS see [here](https://github.com/cwgdata/replicator-benchmark)

# 7. Firewall Rule

Firewall rules setup: Find the cloud brokers
```bash
export CCLOUD_BROKERS=cluster:9092
export CCLOUD_ACCESS_KEY_ID=KEY
export CCLOUD_SECRET_ACCESS_KEY=SECRET
kafkacat -b ${CCLOUD_BROKERS} -L -X securitcdy.protocol=SASL_SSL -X sasl.mechanisms=PLAIN \
 -X sasl.username=${CCLOUD_ACCESS_KEY_ID} -X sasl.password=${CCLOUD_SECRET_ACCESS_KEY} \
 -X ssl.ca.location=/usr/local/etc/openssl/cert.pem -X api.version.request=true
# check IP Adresse 
dig "hostname of broker" +short
```

# 8. Finally Running Replicator in k8s environment
For a k8s deployment of Replicator I used this example:
* [GKE with Replicator to CCloud](https://docs.confluent.io/current/tutorials/examples/kubernetes/replicator-gke-cc/docs/index.html?utm_source=github&utm_medium=demo&utm_campaign=ch.examples_type.community_content.kubernetes)

## How to use it
```bash
git clone git@github.com:confluentinc/examples.git
cd examples/kubernetes/replicator-gke-cc
```
Check correct Project in GCP
```bash
gcloud config list --format 'value(core.project)'
```
Create Base Cluster in GKE, make sure you are using the correct version of GKE, in my case 1.13.11-gke.14 worked
```bash
make gke-create-cluster
```
A k8s Cluster in GCP on GKE is running afterwards, with 7 Nodes.
Create Values File for the cloud destination confluent cluster, replace destination-cluster, APIKEY and SECRET:
```bash
cat <<'EOF' > ./cfg/my-values.yaml
destinationCluster: &destinationCluster
  name: replicator-gke-cc-demo
  tls:
    enabled: true
    internal: true
    authentication:
      type: plain
  bootstrapEndpoint: destination-cluster:9092
  username: APIKEY
  password: SECRET

controlcenter:
  dependencies:
    monitoringKafkaClusters:
    - <<: *destinationCluster

replicator:
  replicas: 1
  dependencies:
    kafka:
      <<: *destinationCluster
EOF
```
Check file
```bash
cat ./cfg/my-values.yaml
```
To verify your GKE cluster status:
```bash
gcloud container clusters list
```
check if proper context
```bash
kubectl config current-context
```
Attention do not use helm 3, use version 2, e.g. v2.14.3
```bash
curl -L https://git.io/get_helm.sh | bash -s -- --version v2.14.3
```
Create the Replicator Demo
```bash
make demo
```
A Kafka Cluster: 3 Zookeeper, 3 Brokers and Confluent Operator will be installed into GKE and the replicator environment, SR, C3, Connectors, Datagen
Forward Port to access C3:
```bash
kubectl -n operator port-forward controlcenter-0 12345:9021
```
and open [Control Center running on GKE](http://localhost:12345) in your Browser
Check topic in source cluster
```bash
kubectl -n operator exec -it client-console bash
kafka-console-consumer --bootstrap-server kafka:9071 --consumer.config /etc/kafka-client-properties/kafka-client.properties --topic stock-trades --property print.value=false --property print.key=true --property print.timestamp=true
```
View topic in destination cluster
```bash
kubectl -n operator exec -it client-console bash
kafka-console-consumer --bootstrap-server $(cat /etc/destination-cluster-client-properties/destination-cluster-bootstrap)  --consumer.config /etc/destination-cluster-client-properties/destination-cluster-client.properties --topic stock-trades --property print.value=false --property print.key=true --property print.timestamp=true
```
# Destroy everything
```Bash
make destroy-demo
make gke-destroy-cluster
```
If this is not working, go to GCP UI and delete the complete cluster, check if all disks and computes are deleted. In my case 10 disks have to be deleted.

## Clean CCloud destination environment
```Bash
ccloud login
ccloud environment use t6463
ccloud kafka cluster use lkc-4jko2
ccloud kafka topic list
for i in $(ccloud kafka topic list | grep _confluent); do echo ${i}; ccloud kafka topic delete ${i}; done
for i in $(ccloud kafka topic list | grep operator.replicator); do echo ${i}; ccloud kafka topic delete ${i}; done
ccloud kafka topic delete stock-trades
```
