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

This task consumes the Kafka topic simFileTrans and writes records to the standard output. 

Command line:

<pre>
$MESOS_SANDBOX/jre1.8.0_91/bin/java -cp $MESOS_SANDBOX/rtsink.jar com.esri.rtsink.KafkaTransformStdout confluent-kafka simFileTrans group1 noOp 1 $PORT0
</pre>

Runs com.esri.rtsink.KafkaTransformStdout in the rsink.jar. Parameters:
- confluent-kafka (name of the kafka application in Marathon)
- simFileTrans (topic to consume)
- group1 (group name to consume under
- noOp (No transformation)
- 1 (Show every record)
- $PORT0 (Listen on default marathon port for health check and resets)

Run the simulation again.

<pre>
$ java -cp Simulator-jar-with-dependencies.jar com.esri.simulator.Tcp tcp-kafka.marathon.mesos 5565 simFile_1000_10s.dat 100 1000
</pre>

This time run 1000 features at a rate of 100 per second.  You should see output of count,rate on both tcp-kafka and on kafka-transform-kafka.  You should also see some transformed; filtered, geotagged json in stdout of kafka-noop-stdout.

