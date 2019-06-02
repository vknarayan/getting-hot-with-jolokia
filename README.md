# getting-hot-with-jolokia
## Java metrics collection with jolokia agent and metricbeat

### Java Management Extensions
Java Management Extensions (JMX) is a Java technology that supplies tools for managing and monitoring applications, system objects, devices (such as printers) and service-oriented networks. Those resources are represented by objects called MBeans (for Managed Bean). In the API, classes can be dynamically loaded and instantiated. Managing and monitoring applications can be designed and developed using the Java Dynamic Management Kit.
https://www.oracle.com/technetwork/java/javase/tech/javamanagement-140525.html

### Jolokia
Jolokia is remote JMX with JSON over HTTP. Jolokia is a JMX-HTTP bridge giving an alternative to JSR-160 connectors. It is an agent based approach with support for many platforms. In addition to basic JMX operations it enhances JMX remoting with unique features like bulk requests and fine grained security policies.
https://jolokia.org/

### JConsole
The JConsole graphical user interface is a monitoring tool that complies to the Java Management Extensions (JMX) specification. JConsole uses the extensive instrumentation of the Java Virtual Machine (Java VM) to provide information about the performance and resource consumption of applications running on the Java platform.
https://docs.oracle.com/javase/7/docs/technotes/guides/management/jconsole.html

#### How to activate JMX on a JVM for access with jconsole?
Note: No settings required on ubuntu, since jconsole can work with default settings. But the below settings may be needed for othr OS variants. 

java -Dcom.sun.management.jmxremote \ -Dcom.sun.management.jmxremote.port=9010 \ -Dcom.sun.management.jmxremote.local.only=false \ -Dcom.sun.management.jmxremote.authenticate=false \ -Dcom.sun.management.jmxremote.ssl=false \ -jar Notepad.jar

Note: This may also be needed for some OS variants: -Djava.rmi.server.hostname=127.0.0.1

#### Exaplanation on settings:
##### com.sun.management.jmxremote
Enables the JMX remote agent and local monitoring via a JMX connector published on a private interface used by JConsole and any other local JMX clients that use the Attach API. JConsole can use this connector if it is started by the same user as the user that started the agent. No password or access files are checked for requests coming via this connector. true / false. Default is true.

In the Java SE 6 platform, it is no longer necessary to set this system property. Any application that is started on the Java SE 6 platform will support the Attach API, and so will automatically be made available for local monitoring and management when needed.

##### com.sun.management.jmxremote.port
Enables the JMX remote agent and creates a remote JMX connector to listen through the specified port. By default, the SSL, password, and access file properties are used for this connector. It also enables local monitoring as described for the com.sun.management.jmxremote property. Port number. No default. When we use the agent, the agent will have the default port of 8778.

##### com.sun.management.jmxremote.authenticate
If this property is false then JMX does not use passwords or access files: all users are allowed all access. true / false. Default is true.

##### com.sun.management.jmxremote. ssl
Enables secure monitoring via SSL. If false, then SSL is not used. true / false. Default is true.

### Jolokia Javaagent
We need to attach javaagent to the targetserver to access jmx mbeans from a client application.

Here is the command to supply javaagent along with application:

*java -javaagent:/home/orange/jolokia/jolokia-1.5.0/agents/jolokia-jvm-1.5.0-agent.jar=port=8778,host=localhost -jar jmx_example.jar*

Note: default port is 8778. For host, you could use host IP for remote access. You can replace jmx_example.jar with your java application

It is possible to attach java agent to an already running process

*java -jar /home/orange/jolokia/jolokia-1.5.0/agents/jolokia-jvm-1.5.0-agent.jar start 28646*

28646 is the pid of the application. You can know the pid using jconsole ( or with ps command ). 

jolokia endpoint examples

http://localhost:8778/jolokia/read/java.lang:type=Runtime
http://localhost:8778/jolokia/read/java.lang:type=Memory
http://localhost:8778/jolokia/read/java.lang:type=Memory/HeapMemoryUsage
http://localhost:8778/jolokia/read/java.lang:type=Memory/HeapMemoryUsage/committed
http://localhost:8778/jolokia/read/java.lang:type=OperatingSystem/Name
http://localhost:8778/jolokia/read/java.lang:type=MemoryPool,name=Metaspace/Usage
http://localhost:8778/jolokia/read/java.nio:type=BufferPool,name=direct/TotalCapacity

### How to handle metrics from different custom applications on the same target ?
To make it possible to differentiate between metrics from multiple similar applications running on the same host, you should configure multiple modules.

There are a large number of metrics which can be monitored in a java application. They are grouped under various mbean. The list of all MBeans and their attributes can be found when we run JConsole.

The below table gives the list of mbean and important metrics being collected in this example.

- java.lang:type=ClassLoading
    - LoadedClassCount, UnloadedClassCount, TotalLoadedClassCount
- java.lang:type=GarbageCollector                     
    - CollectionCount, CollectionTime
- java.lang:type=Memory
    - HeapMemoryUsage, NonHeapMemoryUsage
- java.lang:type=MemoryPool                           
    - Usage, PeakUsage
- java.lang:type=Operating System                     
    - OpenFileDescriptorCount, MaxFileDescriptorCOunt, CommittedVirtualMemorySize, TotalSwapSpaceSize, FreeSwapSpaceSize, ProcessCpuLoad, SystemCpuLoad, SystemLoadAverage, AvailableProcessors, Name, Arch
- java.lang:type=Runtime                              
    - Uptime
- java.lang:type=Threading                            
    - ThreadCount, PeakThreadCount, TotalStartedThreadCount
- java.nio:type=BufferPool                            
    - Count, TotalCapacity, MemoryUsed

### Jolokia module in metricbeat
This module collects metrics from Jolokia agents running on a target JMX server or dedicated proxy server. The default metricset is jmx. To collect metrics, Metricbeat communicates with a Jolokia HTTP/REST endpoint that exposes the JMX metrics over HTTP/REST/JSON.

Using jolokia module, we can read hundreds of attributes exposed by java run time. It can also be used to read custom application metrics. The module has to be configured to read specific metrics. All metrics from a single mapping will be POSTed to the defined host/port and sent to Elastic as a single event. To make it possible to differentiate between metrics from multiple similar applications running on the same host, you should configure multiple modules. When wildcards are used, an event is sent to Elastic for each matching MBean, and an mbean field is added to the event.
Refer to **metricbeat.yml** in this repository for metric beat configuration.
