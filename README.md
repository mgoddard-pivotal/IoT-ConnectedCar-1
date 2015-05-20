# IoT Realized: The Connected Car (Pivotal EDU fork)
The internet of things (IoT) is one of the hottest buzzwords in 2015 and for good 
reason. The promise of leveraging data generated by machines that are interconnected can 
provide vast operational efficiencies in virtually every industry. At Pivotal's 
SpringOne2GX conference 2014, a team from Pivotal demonstrated how components of Pivotal's 
open source portfolio can be combined to increase speed to market of IoT solutions.

Pivotal has developed a sample application to illustrate it's open source IoT platform 
via the use case of a connected car.  In this demo, data from a moving vehicle is streamed
real time to a server running Spring XD.  Spring XD ingests the data and makes predictions
about where the vehicle is going and how far it can go.  An HTML/AngularJS based dashboard
provides real time analytics illustrating the current position of the car, some of the 
basic telemetry available from the car as well as the predictions provided by Pivotal's 
Data Science team.

This code repository consists of all of the source code used in the demo application's 
server deployment.  A separate repository for the iOS application used will be open 
sourced at a future date.

You can view the video of the SpringOne2GX demo online here: 
[SpringOne2GX IoT-Realized: The Connected Car](https://www.youtube.com/watch?v=cejQ46IQpUI)

## Architecture

![Connected Car Architecture](/src/main/resources/images/ConnectedCarArchitecture.jpg)

There are three main prongs to the architecture:

1. Ingestion
2. Batch training
3. Realtime dashboard

### Ingestion
The source of the data for this project is the OBD II data from a moving vehicle.  In the 
live demo, the iOS application Herbie was used to send live data to the server.  In 
offline demonstrations as well as testing, recorded data was replayed via a Spring Batch 
job in the IoT-CarSimulator module.
[Here's an OBD II _dongle_ that works.](http://gopoint-technology.mybigcommerce.com/bt1a-for-android-and-apple/)

The foundation of the ingestion is Spring XD.  Using Spring XD, an ingestion stream was 
developed that accepted the inbound input, performed minor transformations, routed the 
data through the realtime predictions module, and then persisted the output to HDFS.

The stream definition itself is as follows (as seen in the IoT-Scripts/stream-create.xd 
script):

```
http | filter --script=file:DataFilter.groovy | acmeenrich | shell --inputType=application/json --command='python ./StreamPredict.py' --workingDir=IoT-ConnectedCar/IoT-Data-Science/PythonModel/ | hdfs
```

The above stream accepts JSON POSTed to the http source listening on the default port 
(9000).  It then routes the data through a filter that determines if all the required data
is available.  If required data is missing, the record is dropped and the record (and 
reason) are logged.  If all the required data is available, the message is passed onto an
enrichment transformer that performs some basic enrichment and translation for data 
consistency.  The enriched data is then passed to the shell module that is running a 
python module developed by Pivotal's Data Science team.  That python module looks at the 
current journey and, using historical data for that particular vehicle's VIN, creates a
prediction of where it's going and its expected MPG (fuel economy) for the journey.
The results of the shell processor are stored in HDFS.

The above stream is then _tapped_ via the following definition:

```
tap:stream:IoT-HTTP.shell > typeconversiontransformer | gemfire-server --regionName=car-position --keyExpression=payload.vin
```

This tap copies everything coming out of the shell processor and sends it to a transformer.
That transformer changes the type from a simple String containing JSON to a POJO to be 
persisted into GemFire.  The transformed object is then sent to the _gemfire-server_ sink
for persistence into GemFire so that it can be made available to the dashboard.

The modules involved in the ingestion of data in this project are the IoT-DataScience, 
IoT-DataFilter, IoT-EnrichmentTransformer, IoT-GemFireCommons, and IoT-GemfireTransformer 
modules.

### Batch Training
Before the data science module can make a prediction with the real time data, a model must
be generated off of the historical data of previous journeys.  The batch training occurs 
offline with the resulting model used as input to the predictive module in the ingest 
phase.

### Real Time Dashboard
The real time dashboard is an HTML/AngularJS based application that is powered by the data
in GemFire.  The IoT-GemFireREST module provides a set of REST APIs that are called by the
IoT-Dashboard module to provide data about the selected vehicle's current location, 
telemetry, as well as the current state of the predictions.  By default, this app listens
for client connections on **port 9889**.

## Project dependencies
This is intended to illustrate a complete solution and as such has a few dependencies for
it's environment.  Specifically:

1. [Java 7 or higher](http://www.oracle.com/technetwork/java/javase/downloads/index.html)
2. [Spring XD](https://spring.io/projects/spring-xd) release 1.2.0.M1 or higher
   (http://repo.spring.io/libs-snapshot/org/springframework/xd/spring-xd/1.2.0.M1/spring-xd-1.2.0.M1-dist.zip)
   /opt/pivotal/spring-xd/xd/config/servers.yml: Set the value for fsUri here, to match your config:
     fsUri: hdfs://namenode:8020
   /opt/pivotal/spring-xd/xd/config/hadoop.properties: similar to above
     fs.default.name=hdfs://namenode.localdomain:8020
3. Hadoop install compatible with Spring XD release (see above)
   ** NOTE: Run Ambari on port 8888 if running single node VM, to avoid conflicts with IoT app(s)
4. Spark 1.2 or higher (install from RPM -- we have spark-1.2.1.3 in PHD 3.0)
   Ref. http://pivotalhd.docs.pivotal.io/docs/install-manually.html#ref-0a9f3ecc-bf89-4537-91ac-e0bf85752c96
   PySpark will be located here: /usr/phd/3.0.0.0-249/spark/python/pyspark
   NOTE: for spark, you need to edit /etc/spark/conf/java-opts, so it contains this single line:
     -Dphd.version=3.0.0.0-249 -Dstack.name=phd -Dstack.version=3.0.0.0-249
   - Spark will run in YARN-Client mode (seems most appropriate)
   - Where to install Spark, which set of nodes? (Dan B. says just on the node you launch from)
   - yum -y install spark spark-python
   - Ensure that `HADOOP_CONF_DIR` points to the directory which contains the (client side)
     configuration files for the Hadoop cluster. These configs are used to write to the dfs and connect to
     the YARN ResourceManager (ref. https://spark.apache.org/docs/1.2.1/running-on-yarn.html).
     * This is set in the spark-env.sh file configured by the install *
   - Add user `spark` with group `hdfs` to all nodes (not sure if all are req'd., but we'll try it).
     useradd spark
     usermod -G hdfs spark

5. GemFire 8 or higher (install from RPM: pivotal-gemfire)
6. [Anaconda](http://continuum.io/downloads) distribution of Python 2.1.0 or higher
   (try http://conda.pydata.org/miniconda.html)
7. Node.JS (provides NPM, install as root): https://nodejs.org/download/
   - NOTE: this package is built from source, so you'll need GNU Autotools, make, gcc-c++
     (`yum -y install gcc-c++` is the only one I actually had to install)
```
curl http://nodejs.org/dist/v0.12.3/node-v0.12.3.tar.gz | tar xzvf -
cd node-v0.12.3/ && ./configure && make && sudo make install && cd -
```
8. Grunt CLI v0.1.13 or higher:
   (run these as *root*)
```
    yum -y install rubygems
    yum -y install ruby-devel
    gem update --system && gem install compass
```
   (05/15/2015: based on Yeoman tutorial)
   (run these as *your user account*)
```
    sudo yum -y install git
    git clone https://github.com/glenpike/npm-g_nosudo
    ( cd ./npm-g_nosudo/ && ./npm-g-no-sudo.sh )
```
    log out and then back in, or just `. ~/.bashrc` so your environment gets updated
    `npm install --global yo bower grunt-cli`
    Verify they got installed correctly: `yo --version && bower --version && grunt --version` (watch for errors)
    (EXPERIMENTAL BELOW)
    cd IoT-ConnectedCar/IoT-Dashboard/
    npm install
    npm install -g imagemin-gifsicle
    bower install
```
9. pivotal-rabbitmq-server
10. pivotal-redis

## Building from source
There are two main pieces needed to build this project from source:

1. The Dashboard
2. Everything else

### Building the dashboard
The HTML/AngularJS based dashboard was originally created via a Yeoman generator and
therefore uses it's standard stack to perform builds.  To be able to perform a build of
the dashboard's static assets (JavaScript, HTML, etc), you'll need
[Grunt](http://gruntjs.com/).  With Grunt installed, you can build the dashboard via the
following from the root of the IoT-Dashboard module:

```
$ grunt clean build
```
Images won't make it into the build (commented out line 363 in Gruntfile.js), so you now
must do this: `cp -r app/images src/main/resources/public/`

The output of this process will end up in `IoT-Dashboard/src/main/resources/public` for 
Spring Boot packaging.

### Building everything else
Once you've built the dashboard, building the rest of the code is executed simply by 
executing

```
$ ./gradlew clean build
```

from the root of this project.  That will build all modules and package them up
apropriately for deployment (Spring Boot jars for the most part).

## Contributing to IoT-ConnectedCar

Here are some ways for you to get involved in the community:

* Create [Github](https://github.com/Pivotal-Field-Engineering/IoT-ConnectedCar/issues) 
tickets for bugs and new features and comment and vote on the ones that you are interested 
in.
* Github is for social coding: if you want to write code, we encourage contributions 
through pull requests from [forks of this repository](http://help.github.com/forking/).  
If you want to contribute code this way, please familiarize yourself with the process 
outlined for contributing to Pivotal projects here: 
[Contributor Guidelines](https://github.com/Pivotal-Field-Engineering/IoT-ConnectedCar/blob/master/CONTRIBUTING.md).
* Watch for upcoming articles on Pivotal by 
[subscribing](http://blog.pivotal.io/feed) to spring.io

