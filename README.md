# helloworld-mdb

This project shows how-to:

* Install AMQ Artemis Broker
* Install and configure WildFly to connect to the AMQ Artemis Broker
* Use a servlet to send messages to a queue in the AMQ Artemis Broker
* Use a Message Driven Bean to read those messages back from the queue in the AMQ Artemis Broker

> NOTE: "destination" in the linked documentation is used as synonym to queue or topic

> NOTE: in the AMQ Artemis Broker a queue is implemented as an "address" containing a single queue
 

# Install and configure ActiveMQ Artemis:

Download [ActiveMQ Artemis|https://activemq.apache.org/components/artemis/download/] and then:

```

unzip apache-artemis-2.18.0-bin.zip
# now you should have a folder named "apache-artemis-2.18.0"
export ARTEMIS_BASE_PATH=$PWD/apache-artemis-2.18.0
cd $ARTEMIS_BASE_PATH
mkdir $ARTEMIS_BASE_PATH/data
cd $ARTEMIS_BASE_PATH/data
# use admin/admin for the default username and password and allow anonymous access
$ARTEMIS_BASE_PATH/bin/artemis create mybroker
```

start ActiveMQ Artemis:

```
$ARTEMIS_BASE_PATH/data/mybroker/bin/artemis run
```


# Install and configure WildFly:

Download [WildFly|https://www.wildfly.org/downloads/], unzip it in `wildfly-24.0.1.Final`, and configure it as described in the [documnetation|https://docs.wildfly.org/24/Admin_Guide.html#Messaging_Connect_a_pooled-connection-factory_to_a_Remote_Artemis_Server]:

```
unzip wildfly-24.0.1.Final.zip
# now you should have a folder named "wildfly-24.0.1.Final"
export WILDFLY_BASE_PATH=$PWD/wildfly-24.0.1.Final

$WILDFLY_BASE_PATH/bin/jboss-cli.sh 
[disconnected /] embed-server --server-config=standalone-full.xml

[standalone@embedded /] /socket-binding-group=standard-sockets/remote-destination-outbound-socket-binding=remote-artemis:add(host=127.0.0.1, port=61616)
{"outcome" => "success"}

[standalone@embedded /] /subsystem=messaging-activemq/remote-connector=remote-artemis:add(socket-binding=remote-artemis)
{"outcome" => "success"}

[standalone@embedded /] /subsystem=messaging-activemq/pooled-connection-factory=remote-artemis:add(connectors=[remote-artemis], entries=[java:/jms/remoteCF])
{"outcome" => "success"}

[standalone@embedded /] /subsystem=messaging-activemq/pooled-connection-factory=remote-artemis:write-attribute(name="enable-amq1-prefix", value="true")
{"outcome" => "success"}

[standalone@embedded /] /subsystem=messaging-activemq/external-jms-queue=testQueueRemoteArtemis:add(entries=[java:/queue/testQueueRemoteArtemis])
{"outcome" => "success"}
```

you end up with having something like the following in `standalone-full.xml` (note the `enable-amq1-prefix="false"` telling WF not to add the legacy prefixes when talking to Artemis):

```
<subsystem xmlns="urn:jboss:domain:messaging-activemq:13.0">
    <remote-connector name="remote-artemis" socket-binding="remote-artemis"/>
    <pooled-connection-factory name="remote-artemis" entries="java:/jms/remoteCF" connectors="remote-artemis" enable-amq1-prefix="true"/>
    <external-jms-queue name="testQueueRemoteArtemis" entries="java:/queue/testQueueRemoteArtemis"/>
...            
```  

finally start the server:

```
$WILDFLY_BASE_PATH/bin/standalone.sh -c standalone-full.xml
```

> NOTE: if you don't define the queue in the configuration you get the following error when later deploying the application:
> ```text
>   16:10:01,089 ERROR [org.jboss.as.controller.management-operation] (DeploymentScanner-threads - 1) WFLYCTL0013: Operation ("deploy") failed - address: ([("deployment" => "helloworld-mdb.war")]) - failure description: {
>   "WFLYCTL0412: Required services that are not installed:" => ["jboss.naming.context.java.queue.testQueueRemoteArtemis"],
>   "WFLYCTL0180: Services with missing/unavailable dependencies" => ["jboss.naming.context.java.module.helloworld-mdb.helloworld-mdb.env.\"org.jboss.as.quickstarts.servlet.HelloWorldMDBServletClient\".queue is missing [jboss.naming.context.java.queue.testQueueRemoteArtemis]"]
>   }
> ```


# Test Application

For the test application you can use an adapted version of the `helloworld-mdb` module in `git@github.com:wildfly/quickstart.git`;
You can find everything you need at `https://github.com/tommaso-borgato/helloworld-mdb`:

```
git clone git@github.com:tommaso-borgato/helloworld-mdb.git
cd helloworld-mdb
mvn install -Denforcer.skip -DskipTests
```


# enable-amq1-prefix="true"

> This is already what you have if you followed the instructions so far;

With the following configuration in `standalone-full.xml` (note `enable-amq1-prefix="true"`):

```
    <subsystem xmlns="urn:jboss:domain:messaging-activemq:13.0">
        <remote-connector name="remote-artemis" socket-binding="remote-artemis"/>
        <pooled-connection-factory name="remote-artemis" entries="java:/jms/remoteCF" connectors="remote-artemis" enable-amq1-prefix="true"/>      
        <external-jms-queue name="testQueueRemoteArtemis" entries="java:/queue/testQueueRemoteArtemis"/>
        ...
    </subsystem>

    <socket-binding-group name="standard-sockets" default-interface="public" port-offset="${jboss.socket.binding.port-offset:0}">
        ...
        <outbound-socket-binding name="remote-artemis">
            <remote-destination host="127.0.0.1" port="61616"/>
        </outbound-socket-binding>
    </socket-binding-group>
```

And the following Servlet and MDB (note NO `jms.queue.` prefix is present anywhere - not in the servlet nor in the MDB):

```
@WebServlet("/HelloWorldMDBServletClient")
public class HelloWorldMDBServletClient extends HttpServlet {

    @Inject
    @JMSConnectionFactory("java:/jms/remoteCF")
    private JMSContext jmsContext;

    @Resource(lookup = "java:/queue/testQueueRemoteArtemis")
    private Queue queue;
    ...
```

```
@MessageDriven(name = "HelloWorldQueueMDB", activationConfig = {
        @ActivationConfigProperty(propertyName = "destinationLookup", propertyValue = "testQueueRemoteArtemis"),
        @ActivationConfigProperty(propertyName = "destinationType", propertyValue = "javax.jms.Queue"),
        @ActivationConfigProperty(propertyName = "useJNDI", propertyValue = "false"),
        @ActivationConfigProperty(propertyName = "acknowledgeMode", propertyValue = "Auto-acknowledge") })
@ResourceAdapter("remote-artemis")
public class HelloWorldQueueMDB implements MessageListener {
    ...
```

> NOTE: the `@ResourceAdapter("remote-artemis")` references `<pooled-connection-factory name="remote-artemis" ... >` the in the WildFly configuration

When you invoke `http://127.0.0.1:8080/helloworld-mdb/HelloWorldMDBServletClient`:

* the queue (and address) `jms.queue.testQueueRemoteArtemis` is created in Artemis
* the servlet sends messages to Artemis `jms.queue.testQueueRemoteArtemis`
* and the MDB reads messages also from Artemis `jms.queue.testQueueRemoteArtemis`


# enable-amq1-prefix="false"

> To get this configuration you have to change `standalone-full.xml` either manually or via the cli like in the following:
> ```
> [standalone@embedded /] /subsystem=messaging-activemq/pooled-connection-factory=remote-artemis:write-attribute(name="enable-amq1-prefix", value="false")
> {"outcome" => "success"}
> ```
> And you also need to add the `jms.queue.` prefix on the `destinationLookup` in the `HelloWorldQueueMDB` MDB configuration (see later);

With the following configuration in `standalone-full.xml` (note `enable-amq1-prefix="false"`):

```
    <subsystem xmlns="urn:jboss:domain:messaging-activemq:13.0">
        <remote-connector name="remote-artemis" socket-binding="remote-artemis"/>
        <pooled-connection-factory name="remote-artemis" entries="java:/jms/remoteCF" connectors="remote-artemis" enable-amq1-prefix="false"/>      
        <external-jms-queue name="testQueueRemoteArtemis" entries="java:/queue/testQueueRemoteArtemis"/>
        ...
    </subsystem>

    <socket-binding-group name="standard-sockets" default-interface="public" port-offset="${jboss.socket.binding.port-offset:0}">
        ...
        <outbound-socket-binding name="remote-artemis">
            <remote-destination host="127.0.0.1" port="61616"/>
        </outbound-socket-binding>
    </socket-binding-group>
```

And the following Servlet and MDB (note the `jms.queue.` prefix is present only in the MDB configuration):

```
@WebServlet("/HelloWorldMDBServletClient")
public class HelloWorldMDBServletClient extends HttpServlet {

    @Inject
    @JMSConnectionFactory("java:/jms/remoteCF")
    private JMSContext jmsContext;

    @Resource(lookup = "java:/queue/testQueueRemoteArtemis")
    private Queue queue;
    ...
```

```
@MessageDriven(name = "HelloWorldQueueMDB", activationConfig = {
        @ActivationConfigProperty(propertyName = "destinationLookup", propertyValue = "jms.queue.testQueueRemoteArtemis"),
        @ActivationConfigProperty(propertyName = "destinationType", propertyValue = "javax.jms.Queue"),
        @ActivationConfigProperty(propertyName = "useJNDI", propertyValue = "false"),
        @ActivationConfigProperty(propertyName = "acknowledgeMode", propertyValue = "Auto-acknowledge") })
@ResourceAdapter("remote-artemis")
public class HelloWorldQueueMDB implements MessageListener {
    ...
```

> NOTE: the `@ResourceAdapter("remote-artemis")` references `<pooled-connection-factory name="remote-artemis" ... >` the in the WildFly configuration

When you invoke `http://127.0.0.1:8080/helloworld-mdb/HelloWorldMDBServletClient`:

* the queue (and address) `jms.queue.testQueueRemoteArtemis` is created in Artemis
* the servlet sends messages to Artemis `jms.queue.testQueueRemoteArtemis`
* and the MDB reads messages also from Artemis `jms.queue.testQueueRemoteArtemis`
