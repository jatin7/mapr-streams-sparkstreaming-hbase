

Commands to run :

Step 1: First compile the project on eclipse: Select project  -> Run As -> Maven Install

Step 2: use scp to copy the ms-sparkstreaming-1.0.jar to the mapr sandbox or cluster

also use scp to copy the data  sensor.csv file from the data folder to the cluster
put this file in a folder called data. The producer reads from this file to send messages.

copy the jar from the target folder to the sandbox or cluster:

scp  ms-sparkstreaming-1.0.jar mapr@ipaddress:/user/mapr/.

if you are using vmware:
scp  ms-sparkstreaming-1.0.jar mapr@<ipaddress>:/user/mapr/.

if you are using virtualbox:
scp -P 2222 ms-sparkstreaming-1.0.jar mapr@127.0.0.1:/user/mapr/.

ssh into mapr cluster or sandbox  

if you are using vmware:
 ssh mapr@<ipaddress>

if you are using virtualbox:
 ssh mapr@127.0.0.1 -p 2222

Create the topics

maprcli stream create -path /user/mapr/pump -produceperm p -consumeperm p -topicperm p
maprcli stream topic create -path /user/mapr/pump -topic sensor -partitions 3
maprcli stream topic create -path /user/mapr/pump -topic sensoralert -partitions 3

get info on the topic
maprcli stream info -path /user/mapr/pump

maprcli stream topic info -path /user/mapr/pump -topic sensor -json

To run the MapR Streams Java producer and consumer:

java -cp ms-sparkstreaming-1.0.jar:`mapr classpath` solution.MyProducer

java -cp ms-sparkstreaming-1.0.jar:`mapr classpath` solution.MyConsumer

Step 3:  To run the Spark  streaming consumer:

http://maprdocs.mapr.com/51/index.html#Spark/Spark_IntegrateMapRStreams_Consume.html

start the streaming app
 
spark-submit --class solution.SensorStreamConsumer --master local[2] ms-sparkstreaming-1.0.jar

ctl-c to stop 

step 4: Spark streaming app writing to hbase

first make sure that you have the correct version of HBase installed, it should be 1.1.1:

http://maprdocs.mapr.com/51/index.html#Spark/IntegrateSpark-IntegrateS_29656196-d3e150.html

cat /opt/mapr/hbase/hbaseversion 
1.1.1

Next make sure the Spark HBase compatibility version is correctly configured here: 
cat  /opt/mapr/spark/spark-1.6.1/mapr-util/compatibility.version 
hbase_versions=1.1.1

If this is not 1.1.1 fix it.


Create an hbase table to write to:
launch the hbase shell
$hbase shell

create '/user/mapr/sensor', {NAME=>'data'}, {NAME=>'alert'}, {NAME=>'stats'}

Step 4: start the streaming app writing to HBase
 
spark-submit --class solution.HBaseSensorStream --master local[2] ms-sparkstreaming-1.0.jar

ctl-c to stop

Scan HBase to see results:

$hbase shell

scan '/user/mapr/sensor' , {'LIMIT' => 5}

scan '/user/mapr/sensor' , {'COLUMNS'=>'alert',  'LIMIT' => 50}

 java -cp ms-sparkstreaming-1.0.jar:`hbase classpath` dao.SensorTablePrint

Step 5:
exit, run spark (not streaming) app to read from hbase and write daily stats 

spark-submit --class solution.HBaseReadRowWriteStats --master local[2] ms-sparkstreaming-1.0.jar

$hbase shell

scan '/user/mapr/sensor' , {'COLUMNS'=>'stats',  'LIMIT' => 50}


cleanup if you want to re-run:

maprcli stream topic delete -path /user/mapr/pump -topic sensor 
maprcli stream topic create -path /user/mapr/pump -topic sensor -partitions 3
hbase shell 
truncate '/user/mapr/sensor'

******  For a Spark streaming to read from a MapR Streams topic, write alerts to another topic. Then a MapR Streams consumer to read the alerts 
and write to HBase:
http://maprdocs.mapr.com/51/index.html#Spark/Spark_IntegrateMapRStreams_Produce.html

create alert topic, if you did not already do so

maprcli stream topic create -path /user/mapr/pump -topic sensoralert -partitions 3

Run Spark Streaming Consumer which reads from sensor topic, filters and publishes to alert topic

spark-submit --class solution.SensorStreamConsumerProducer --master local[2] ms-sparkstreaming-1.0.jar

Run MapR Streams Consumer which writes to HBase 

java -cp ms-sparkstreaming-1.0.jar:`mapr classpath`:`hbase classpath` solution.MyConsumerWriteHBase

Run HBase client which prints out 20 rows

java -cp ms-sparkstreaming-1.0.jar:`mapr classpath`:`hbase classpath` dao.SensorTablePrint

$hbase shell

scan '/user/mapr/sensor' , {'COLUMNS'=>'alert',  'LIMIT' => 50}


