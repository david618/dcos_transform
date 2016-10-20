<h1>Transforming CSV using a spatial filter</h1>

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


<h1> Create a DCOS Cluster </h1>

For my example I created a cluster with one master, three agents, and one public agent in Azure.

<h1> Compiled Applications </h1>

- <a href="https://github.com/david618/rtsource">Real-Time Source</a> 
- <a href="https://github.com/david618/rtsink">Real-Time Sink</a>
- <a href="https://github.com/david618/Simulator">Simulator</a> 

<h1> Create rtlib.tgz </h1>

Combined target/lib for both Source and Sink into one folder "lib".  Then created tgz.

<pre>$ tar cvzf rtlib.tgz lib/*</pre>

<h1> Copy files to DCOS Master </h1>

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

<h1> Edit TransformGeotagSimile.properties </h1>

<pre>
$ vi com.esri.rtsink.TransformGeotagSimFile.properties
fenceUrl=http://m1/data/airports1000FS.json
fieldName=iata_faa
filter=true
</pre>

- fenceUrl: The Marathon task kafka-transform-kafka downloads polygons from this URL.
- fieldName: The Marathon task kafka-transform-kafka adds this field and if point is in a polygon the value from polygon feature
- filter: If true the Marathon task kafka-transform-kafka only writes output for points in a polygon. 

<h1> Move files to Web Server on Master </h1>

<pre>
# mkdir /opt/mesosphere/active/dcos-ui/usr/apps
# cp /home/azureuser/rtlib.tgz /opt/mesosphere/active/dcos-ui/usr/apps/
# cp /home/azureuser/rtsource.jar /opt/mesosphere/active/dcos-ui/usr/apps/
# cp /home/azureuser/rtlib.tgz /opt/mesosphere/active/dcos-ui/usr/apps/
# cp /home/azureuser/rtlib.tgz /opt/mesosphere/active/dcos-ui/usr/apps/
</pre>
<pre>
# mkdir /opt/mesosphere/active/dcos-ui/usr/data
# cp /home/azure/airports1000FS.json /opt/mesosphere/active/dcos-ui/usr/data/
</pre>
