<h2>Transforming CSV using a spatial filter</h2>

Three Marathon tasks 
- <b>tcp-kafka</b> Listens on port 5565 for lines of comma separated values; write lines to Kafka Topic 
- <b>kafka-transform-kafka</b> Parse csv values; create json; create point feature from lon/lat fields; if point is in one of polygons then geotag; write lines to another Kafka Topic
- <b>kafka-noop-stdout</b> Read lines from second Kafka Topic and print to Standar Output 

Sample Input Line
<pre>
468935966122,138,19-Jul-2016 08:46:06.006,IAH-IAD,-88.368,34.02488,238.75427650928157,57.53489
</pre>

Sample Output Line
<pre>
8d39e86f-252e-4472-97f9-810205fda218:{"attr":{"rt":"YCY-YIO","dtg":"19-Jul-2016 08:49:29.029","spd":295.15789708093814,"brg":-57.96348,"tm":1468936169220,"id":551,"geotag":"YIO"},"geom":{"x":-77.80259,"y":72.65293,"spatialReference":{"wkid":4326}}}
</pre>


<h2> Create a DCOS Cluster </h2>

For my example I created a cluster with one master, three agents, and one public agent in Azure.

<h2> Compiled Applications </h2>

- <a href="https://github.com/david618/rtsource">Real-Time Source</a> 
- <a href="https://github.com/david618/rtsink">Real-Time Sink</a>
- <a href="https://github.com/david618/Simulator">Simulator</a> 

<h2> Create rtlib.tgz </h2>

Combined target/lib for both Source and Sink into one folder "lib".  Then created tgz.

<pre>$ tar cvzf rtlib.tgz lib/*</pre>

<h2> Copy files to DCOS Master </h2>

<pre>
$ scp -i azureuser rtlib.tgz azureuser@IPADDRESS:. 
$ scp -i azureuser rtsource.jar azureuser@IPADDRESS:. 
$ scp -i azureuser rtsink.jar azureuser@IPADDRESS:.
$ scp -i azureuser jre-8u91-linux-x64.tar.gz azureuser@IPADDRESS:.  
$ scp -i azureuser Simulator-jar-with-dependencies.jar azureuser@IPADDRESS:. 
$ scp -i azureuser com.esri.rtsink.TransformGeotagSimFile.properties azureuser@IPADDRESS:. 
$ scp -i azureuser airports1000FS.json azureuser@IPADDRESS:. 
$ scp -i azureuser simFile_1000_10s.dat azureuser@IPADDRESS:. 
$ scp -i azureuser2 NetBeansProjects/Simulator/pythonScripts/get_counts.py azureuser@23.99.2.155:.
$ scp -i azureuser2 NetBeansProjects/Simulator/pythonScripts/reset_counts.py  azureuser@23.99.2.155:.

</pre>

Make the Python Scripts Executable
<pre>
$ chmod u+x get_counts.py
$ chmod u+x reset_counts.py
</pre>

The Python scripts use a Library that needs to be installed.

<pre>
$ sudo su -
# yum install epel-release 
If you get an error about cannot open the Package database. Reboot the server  # reboot
Give it a couple of minutes to reboot. DC/OS takes a few minutes to start.

# yum install python-httplib2
</pre>

The jre can be whatever is the most current version.  You'll need to adjust marathon apps as needed below.

<h2> Edit TransformGeotagSimile.properties </h2>

<pre>
$ vi com.esri.rtsink.TransformGeotagSimFile.properties
fenceUrl=http://m1/data/airports1000FS.json
fieldName=iata_faa
filter=true
</pre>

- fenceUrl: The Marathon task kafka-transform-kafka downloads polygons from this URL.
- fieldName: The Marathon task kafka-transform-kafka adds this field and if point is in a polygon the value from polygon feature
- filter: If true the Marathon task kafka-transform-kafka only writes output for points in a polygon. 

<h2> Package properties file with rtsink.jar </h2>

<pre>
$ tar cvzf rtsink.tgz com.esri.rtsink.TransformGeotagSimFile.properties rtsink.jar
</pre>

<h2> Move files to Web Server on Master </h2>

<pre>
$ sudo su -
# mkdir /opt/mesosphere/active/dcos-ui/usr/apps
# cp /home/azureuser/rtlib.tgz /opt/mesosphere/active/dcos-ui/usr/apps/
# cp /home/azureuser/rtsource.jar /opt/mesosphere/active/dcos-ui/usr/apps/
# cp /home/azureuser/rtsink.jar /opt/mesosphere/active/dcos-ui/usr/apps/
# cp /home/azureuser/rtsink.tgz /opt/mesosphere/active/dcos-ui/usr/apps/
# cp /home/azureuser/jre-8u91-linux-x64.tar.gz /opt/mesosphere/active/dcos-ui/usr/apps/
</pre>
<pre>
# mkdir /opt/mesosphere/active/dcos-ui/usr/data
# cp /home/azureuser/airports1000FS.json /opt/mesosphere/active/dcos-ui/usr/data/
</pre>

<h2> Deploy Kafka From Universe </h2>

I used confluent-kafka and defaults.

<h2> Create tcp-kafka Marathon App </h2>

Use this json <a href="tcp-kafka.json">tcp-kafka.json</a>

Review the json.  You may need to tweak some of the elements.  For example if you didn't use master server to host the url or you used a different jre version.

Watch Mesos to make sure the application deploys. If it fails stop in Marathon and correct any errors reported in Sandbox stderr or stdout.

<h2> Run Simulator </h2>

<p> The goal of this simulation run is to verify the tcp-kafka is working and it will also create the Kafka topic. </p>

<pre>
$ java -cp Simulator-jar-with-dependencies.jar com.esri.simulator.Tcp tcp-kafka.marathon.mesos 5565 simFile_1000_10s.dat 10 100
</pre>

Simulator Parameters for com.esri.simulator.Tcp:
- host (Host or IP to call; tcp-kafka.marathon.mesos is the mesos DNS name for the tcp-kafka task.)
- port (Port number to use; sends to port 5565)
- simulator file (This is a file with lines of CSV data; each line is an event)
- rate (Number of lines to send per second; 10)
- number (Number of events to send; 100)

Check stdout of the tcp-kafka.  You should see something like: <br>

<pre>
100 , 10
</pre>

Stdout reports that 100 features were read at a rate of 10 per second.  Rate might not be 10; I've seen 9,10,and 11.  This is the rate measured for the input and with a small sample set it can vary. 

<h2> Create kafka-transform-kafka Marathon App </h2>

Use this json <a href="kafka-transform-kafka.json">kafka-transform-kafka.json</a>

This application picks the CSV lines off of Kafka topic simFile performs a transformation and writes to Kafka topic simFileTrans.  

Command Line

<pre>
$MESOS_SANDBOX/jre1.8.0_91/bin/java -cp $MESOS_SANDBOX/rtsink.jar com.esri.rtsink.KafkaTransformKafka confluent-kafka simFile group1 com.esri.rtsink.TransformGeotagSimFile simFileTrans $PORT0 
</pre>

Runs com.esri.rtsink.KafkaTransformKafka in the rsink.jar. Parameters:
- confluent-kafka (name of the kafka application in Marathon)
- simFile (topic to consume)
- group1 (group name to consume under
- com.esri.rtsink.TransformGeotagSimFile (Transformation Class; this class reads properties file bundled in rtsink.tgz)
- simFileTrans (topic to write results too)
- $PORT0 (Listen on default marathon port for health check and resets)

Run the simulation again.

<pre>
$ java -cp Simulator-jar-with-dependencies.jar com.esri.simulator.Tcp tcp-kafka.marathon.mesos 5565 simFile_1000_10s.dat 100 1000
</pre>

This time run 1000 features at a rate of 100 per second.  You should see output of count,rate on both tcp-kafka and on kafka-transform-kafka. In the stdout of kafka-transform-kafka; you should also see a line as it reads the 100 events you sent earlier. Then a second line with the 1,000 events you just sent.  The rate for both should be around 100.  


<h2> Create kafka-noop-stdout Marathon App </h2>

Use this json <a href="kafka-noop-stdout.json">kafka-noop-stdout.json</a>

This task consumes the Kafka topic simFileTrans and writes records to the standard output. As before verify that it deploys successfully in Mesos.

Command line from Marathon App:

<pre>
$MESOS_SANDBOX/jre1.8.0_91/bin/java -cp $MESOS_SANDBOX/rtsink.jar com.esri.rtsink.KafkaTransformStdout confluent-kafka simFileTrans group1 noOp 1 $PORT0
</pre>

Runs com.esri.rtsink.KafkaTransformStdout in the rsink.jar. Parameters:
- confluent-kafka (name of the kafka application in Marathon)
- simFileTrans (topic to consume)
- group1 (group name to consume under
- noOp (No transformation)
- 10 (Show every 10th line of output)
- $PORT0 (Listen on default marathon port for health check and resets)

Run the simulation again.  This time we'll send at even a faster rate

<pre>
$ java -cp Simulator-jar-with-dependencies.jar com.esri.simulator.Tcp tcp-kafka.marathon.mesos 5565 simFile_1000_10s.dat 1000 10000
</pre>

This time run 10,000 features at a rate of 1,000 per second.  You should see output of count,rate on both tcp-kafka and on kafka-transform-kafka.  You should also see some transformed; filtered, geotagged json in stdout of kafka-noop-stdout.  The last line will be the rate send received on this task. 

<pre>
0>> 572d7838-87fe-4cd7-a73a-a762ea7bb89d:{"attr":{"rt":"BEG-LHR","dtg":"19-Jul-2016 08:46:06.006","spd":207.60457680454897,"brg":-68.60878,"tm":1468935966302,"id":451,"geotag":"CRL"},"geom":{"x":4.36824,"y":50.39944,"spatialReference":{"wkid":4326}}}
10>> 2199e9f3-e303-42b0-ac8c-e0493aa8bc58:{"attr":{"rt":"STN-SZG","dtg":"19-Jul-2016 08:46:13.013","spd":214.09156501569387,"brg":111.42974,"tm":1468935973393,"id":762,"geotag":"STN"},"geom":{"x":0.26367,"y":51.87805,"spatialReference":{"wkid":4326}}}
15,Infinity
0>> a355b3fd-3f4a-4eda-b70b-e4a2077becc7:{"attr":{"rt":"BEG-LHR","dtg":"19-Jul-2016 08:46:06.006","spd":207.60457680454897,"brg":-68.60878,"tm":1468935966302,"id":451,"geotag":"CRL"},"geom":{"x":4.36824,"y":50.39944,"spatialReference":{"wkid":4326}}}
10>> b68721b1-3794-4e6e-9e6d-e4d30754a9ce:{"attr":{"rt":"GRO-NCL","dtg":"19-Jul-2016 08:46:14.014","spd":238.9206340687432,"brg":-13.34098,"tm":1468935974029,"id":481,"geotag":"LTN"},"geom":{"x":-0.41302,"y":51.95455,"spatialReference":{"wkid":4326}}}
20>> f34075a1-2628-4276-8180-1df0853eee54:{"attr":{"rt":"BCN-KTW","dtg":"19-Jul-2016 08:46:21.021","spd":203.14428141194298,"brg":58.52995,"tm":1468935981635,"id":271,"geotag":"KTW"},"geom":{"x":19.02586,"y":50.45318,"spatialReference":{"wkid":4326}}}
30>> 4be4a809-a442-4deb-9626-5575b63303c9:{"attr":{"rt":"WAW-MMX","dtg":"19-Jul-2016 08:46:29.029","spd":233.42546691628627,"brg":-56.18527,"tm":1468935989582,"id":743,"geotag":"MMX"},"geom":{"x":13.371639,"y":55.530193,"spatialReference":{"wkid":4326}}}
40>> 17aaa078-418e-4837-b241-f7f623d1c553:{"attr":{"rt":"KTW-BGO","dtg":"19-Jul-2016 08:46:35.035","spd":211.31381374850787,"brg":-44.5738,"tm":1468935995776,"id":975,"geotag":"BGO"},"geom":{"x":5.24879,"y":60.27799,"spatialReference":{"wkid":4326}}}
50>> 9a4ffb95-c58b-4b5b-871b-3f757eb9caf7:{"attr":{"rt":"VDB-OSL","dtg":"19-Jul-2016 08:46:44.044","spd":272.84134800736103,"brg":133.48584,"tm":1468936004222,"id":735,"geotag":"OSL"},"geom":{"x":11.100361,"y":60.193917,"spatialReference":{"wkid":4326}}}
60>> ebc03f30-2e30-412e-9357-24cf025d32aa:{"attr":{"rt":"ELL-JNB","dtg":"19-Jul-2016 08:46:58.058","spd":294.1188557910152,"brg":169.59022,"tm":1468936018640,"id":755,"geotag":"JNB"},"geom":{"x":28.23731,"y":-26.09672,"spatialReference":{"wkid":4326}}}
70>> e2f48288-8fec-4651-85ec-89ccc59e950a:{"attr":{"rt":"VLC-MAN","dtg":"19-Jul-2016 08:47:15.015","spd":228.20332606224298,"brg":-5.43013,"tm":1468936035250,"id":879,"geotag":"BOH"},"geom":{"x":-1.85622,"y":50.72544,"spatialReference":{"wkid":4326}}}
80>> 735ff7aa-b4fe-4b28-931d-e6980c14633d:{"attr":{"rt":"BJA-ALG","dtg":"19-Jul-2016 08:47:32.032","spd":248.35710403475096,"brg":-91.28128,"tm":1468936052555,"id":385,"geotag":"ALG"},"geom":{"x":3.32227,"y":36.693,"spatialReference":{"wkid":4326}}}
90>> f297295d-b355-4210-8e67-1c4bd52107d4:{"attr":{"rt":"BSL-DJE","dtg":"19-Jul-2016 08:47:46.046","spd":219.07433060977658,"brg":170.77712,"tm":1468936066357,"id":733,"geotag":"SFA"},"geom":{"x":10.60943,"y":34.72432,"spatialReference":{"wkid":4326}}}
91,9.270578647106765
</pre>

In this case only 91 of the 10,000 features we're within the polygons; the other features we're filtered out.

<p> You can test even faster rates now. With one instance kafka-transform-kafka the max rate I see on my Azure cluster was around 10,000 events per second. </p>

<h2>Running multiple instances of Kafka-transform-Kafka</h2>

<p>In order to effectively use multiple instances; you need to increase the number of partitions used by the topic. The default Kafka configuration creates a single partition for each topic (e.g. simFile). You can use the DCOS CLI interface to increase the number of partitions for a topic. </p>

<pre>
$ dcos kafka topic --name confluent-kafka list
[
    "simFile",
    "simFileTrans",
    "__consumer_offsets"
]

$ dcos kafka topic --name confluent-kafka describe simFile
{
    "partitions": [
        {
            "0": {
                "controller_epoch": 1,
                "isr": [
                    0
                ],
                "leader": 0,
                "leader_epoch": 0,
                "version": 1
            }
        }
    ]
}

$ dcos kafka topic --name confluent-kafka --help
Usage: dcos-kafka kafka topic [OPTIONS] COMMAND [ARGS]...

  Kafka Topic maintenance

Options:
  --name TEXT  Name of the Kafka instance to query
  --help       Show this message and exit.

Commands:
  create                       Creates a new topic
  delete                       Deletes an existing topic
  describe                     Describes a single existing topic
  list                         Lists all available topics in a framework
  offsets                      Returns the current offset counts for a topic
  partitions                   Alters partition count for an existing topic
  producer_test                Produce some test messages against a topic
  unavailable_partitions       Gets info for any unavailable partitions
  under_replicated_partitions  Gets info for any under-replicated partitions

$ dcos kafka topic --name confluent-kafka partitions simFile 6
{
    "message": "Output: WARNING: If partitions are increased for a topic that has a key, the partition logic or ordering of the messages will be affected\nAdding partitions succeeded!\n"
}

</pre>


<p>From Marathon you can increase the number of instances of the task. Click Scale Application and set a value (e.g. 2). In Mesos you should see another instance of the task start.  Since "hostanme:UNIQUE" is set each instance will run on a different server. <b>NOTE:</b> After changing the topic partitions you should scale to 0 then scale back up to a number you want. </p>

<p>With multiple instances; each instance will process a portion of the events that appear on the Kafka topic. You'd would need to sum up the rate of each instance to get the total throughput of for these two instances. To make it easier the tasks have a rest interface that can be used to retrieve the counts and rates and reset the numbers. The Python scripts provided can be used to collect the rates.  Before running a test use reset python script to clear the results.</p>



<pre>
$ ./reset_counts.py tcp-kafka kafka-transform-kafka
Sources
172.17.2.4:25442
{u'done': True}

Sinks
172.17.2.7:8271
{u'done': True}
172.17.2.5:13650
{u'done': True}


$ ./get_counts.py tcp-kafka kafka-transform-kafka
[u'172.17.2.4:25442']
[u'172.17.2.7:8271', u'172.17.2.5:13650']
Sources
172.17.2.4:25442
1477408492786

Sinks
172.17.2.7:8271
1477408492791
172.17.2.5:13650
1477408492791

Summary
Source Count: 0
Source Rate: 0.0
Source Latency: 0.0
Number of Sources: 0

Sink Count: 0
Sink Rate: 0.0
Sink Latency: 0.0
Number of Sinks: 0

0,0,0,0
</pre>

The last line shows count,rate,count,rate. First pair is for tcp-kafka; second is for kafka-transform-kafka.  After running reset and verifying the counts are zero, run Simulation.

<pre>
$ java -cp Simulator-jar-with-dependencies.jar com.esri.simulator.Tcp tcp-kafka.marathon.mesos 5565 simFile_1000_10s.dat 20000 200000
</pre>

Now check the throughput using Python script.

<pre>
$ ./get_counts.py tcp-kafka kafka-transform-kafka
[u'172.17.2.4:25442']
[u'172.17.2.9:8040', u'172.17.2.6:15587']
Sources
172.17.2.4:25442
0:200000:18892.8773852:0
1477409375178

Sinks
172.17.2.9:8040
0:85336:7998.50032805
1477409356224
172.17.2.6:15587
0:114664:9513.31618684
1477409356228

Summary
Source Count: 200000
Source Rate: 18892.8773852
Source Latency: 0.0
Number of Sources: 1

Sink Count: 200000
Sink Rate: 17511.8165149
Sink Latency: 0.0
Number of Sinks: 2

200000,18893,200000,17512
</pre>

The above results show an input rate of 18,893 with an output rate of 17,512. The input rate is how fast the events were received and written to the simFile Kafka topic. The output rate is how fast the events were parsed, converted to Json, compared to 10,000 polygons, and geotagged with information with any intersecting polygon. <br>

Tests have show that the transform scales nearly linearly at about 10,000 e/s for instance of kafka-transform-kafa task.

<h1>The End</h1>
