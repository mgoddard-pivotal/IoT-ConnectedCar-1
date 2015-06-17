# IoT Realized: The Connected Car
## (Pivotal EDU fork)

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
**Note: if incoming data is associated with a VIN value for which no model exists, this
ingest stream will break.  Fixing this is on the TODO list.**

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
   [Pull from S3](https://s3.amazonaws.com/goddard.phd3/jdk-7u67-linux-x64.rpm)
2. [Spring XD](https://spring.io/projects/spring-xd) release 1.2.0.M1 or higher
   [I used this](http://repo.spring.io/libs-snapshot/org/springframework/xd/spring-xd/1.2.0.M1/spring-xd-1.2.0.M1-dist.zip)
   [Pull from S3](https://s3.amazonaws.com/iot.connected.car/opt_pivotal_spring-xd.tar.gz)
   Edit the following Spring XD files (replace `[NAMENODE_HOSTNAME_OR_IP]` with the appropriate value):
     * spring-xd/xd/config/servers.yml: `fsUri: hdfs://[NAMENODE_HOSTNAME_OR_IP]:8020`
     * spring-xd/xd/config/hadoop.properties: `fs.default.name=hdfs://[NAMENODE_HOSTNAME_OR_IP]:8020`
   **Distributed mode**: It looks like running
   [Spring XD on YARN](http://docs.spring.io/spring-xd/docs/current-SNAPSHOT/reference/html/#running-on-YARN)
   is potentially interesting as we could share resources with Hadoop, depoying on the same nodes (DataNodes).
   Here's a section on Spring XD in
   [distributed mode](http://docs.spring.io/spring-xd/docs/current-SNAPSHOT/reference/html/#running-distributed-mode),
   where the notes on [RabbitMQ in clustered mode](https://www.rabbitmq.com/clustering.html) will be needed
   as well.  See also [course materials](https://github.com/S2EDU/RabbitMQ-ILT).
3. Hadoop install compatible with Spring XD release (we'll be using Pivotal HD 3.0)
   NOTE: If running in a single node, run Ambari on port 8888 to avoid conflicts with Gemfire REST service
4. Spark 1.2 or higher
   * [Install Docs](http://pivotalhd.docs.pivotal.io/docs/install-manually.html#ref-0a9f3ecc-bf89-4537-91ac-e0bf85752c96)
   * Spark will run in YARN-Client mode, and **it only needs to be installed onto the node from which you'll run it**
   * `yum -y install spark spark-python` (as root)
   * Add spark user to the NameNode: `useradd -g hdfs spark`
   * Edit /etc/spark/conf/java-opts, so it contains this single line:
     ```
      -Dphd.version=3.0.0.0-249 -Dstack.name=phd -Dstack.version=3.0.0.0-249
     ```
   * Ensure /etc/spark/conf/spark-env.sh contains the following two lines:
     ```
      export HADOOP_HOME=/usr/phd/3.0.0.0-219/hadoop
      export HADOOP_CONF_DIR=/usr/phd/3.0.0.0-219/hadoop/conf
     ```
   * Edit /etc/spark/conf/spark-defaults.conf:
     ```
      spark.driver.extraJavaOptions -Dphd.version=3.0.0.0-249 -Dstack.version=3.0.0.0-249 -Dstack.name=phd
      spark.yarn.am.extraJavaOptions -Dphd.version=3.0.0.0-249 -Dstack.version=3.0.0.0-249 -Dstack.name=phd
     ```
5. GemFire 8 or higher
   * [Pull from S3](https://s3.amazonaws.com/iot.connected.car/opt_pivotal_gemfire.tar.gz)
   * [Example of distributed deployment](https://github.com/lshannonPivotal/gemfire-hellogbye-poc)
   * **NOTE:** deploy the Gemfire locator to a host which resolves to "gem-locator" (update your /etc/hosts
     on all your nodes).  Run this on port 9001.  Set up your Gemfire server processes on this same host and
     on two others; name them gem-server1 and gem-server2 (again, in /etc/hosts on all nodes).
6. [Miniconda](http://conda.pydata.org/miniconda.html) distribution of Python 2.1.0 or higher
   [Pull from S3](https://s3.amazonaws.com/iot.connected.car/opt_miniconda.tar.gz), or install it
   and then perform the following:
  * Post-install for Miniconda: update the [pivotal.sh](/IoT-Scripts/pivotal.sh) file to point to
    its install directory, and **copy pivotal.sh into /etc/profile.d/ (as root)**
  * Install all the required Python modules (after having sourced that pivotal.sh file)
```
#!/bin/bash

for i in IPython brewer2mpl collections copy datetime folium glob json logging \
  math matplotlib numpy operator pandas pylab random sklearn statsmodels sys \
  time toolz uuid scipy
do
  find /opt/miniconda -name "$i*" | grep $i >/dev/null || conda install $i
done

# These needed some extra help
conda install binstar
conda install --channel https://conda.binstar.org/IOOS-RHEL6 brewer2mpl
conda install --channel https://conda.binstar.org/IOOS-RHEL6 folium
```
7. Node.JS (provides NPM, install as root): https://nodejs.org/download/
   * NOTE: this package is built from source, so you'll need GNU Autotools, make, gcc-c++
   * I had to run `yum -y install gcc-c++` to get this to build; this will vary
   * Download, extract, build, install:
```
curl http://nodejs.org/dist/v0.12.3/node-v0.12.3.tar.gz | tar xzvf -
cd node-v0.12.3/ && ./configure && make && sudo make install && cd -
```
8. Grunt CLI v0.1.13 or higher
   * Run these as **root**:
```
    yum -y install git
    yum -y install rubygems
    yum -y install ruby-devel
    gem update --system && gem install compass
```
   (05/15/2015: based on Yeoman tutorial)
   * Run these as **the user which will run the build**, responding to the prompts as appropriate:
```
    git clone https://github.com/glenpike/npm-g_nosudo
    ( cd ./npm-g_nosudo/ && ./npm-g-no-sudo.sh )
```
   * Log out and then back in, or just `. ~/.bashrc` so your environment gets updated
   * `npm install --global yo bower grunt-cli`
   * Verify they got installed correctly: `yo --version && bower --version && grunt --version` (watch for errors)
   * FIXME: The next few steps **are not very repeatable**, so they may require some iteration:
```
   cd IoT-ConnectedCar/IoT-Dashboard/
   npm install
   npm install imagemin-gifsicle
   bower install
```
9. Install pivotal-rabbitmq-server
10. Install pivotal-redis
11. Install the "data" subdir into /opt/pivotal/
[Pull from S3](https://s3.amazonaws.com/iot.connected.car/opt_pivotal_data.tar.gz)

## Building from source
There are two main pieces needed to build this project from source:

1. The Dashboard
2. Everything else

### Building the dashboard
The HTML/AngularJS based dashboard was originally created via a Yeoman generator and
therefore uses it's standard stack to perform builds.  To be able to perform a build of
the dashboard's static assets (JavaScript, HTML, etc), you'll need
[Grunt](http://gruntjs.com/).
**Consider replacing Grunt.js with [(Gulp.js)](http://gulpjs.com/)?**

There is a dependency on injecting the IP number (or, hostname, if it resolves for browser
based clients) of the Gemfire REST API host.  The placeholder I inserted for now is
`%GEM_REST_API_HOST_IP%`, into the following two files **Need to edit, for now**:
```
	config/environments/production.json
	src/main/resources/public/scripts/scripts.js
```

With Grunt installed (per step 8, above), you can build the dashboard
from the root of the IoT-Dashboard module:

```
$ grunt clean build
```
Images won't make it into the build (we commented out line 363 in Gruntfile.js), so
**you must do this**: `cp -r app/images src/main/resources/public/`

**TODO**: Figure out how to get the grunt build, or maybe the gradle build, to make this replacement.
[This](https://github.com/outaTiME/grunt-replace) might do it.

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

