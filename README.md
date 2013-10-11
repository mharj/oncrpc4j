ONCRPC4J
========

This is a part of dCache.ORG's NFSv4.1 work.

Technically, this is not a fork of Remote Tea RPC library, but formally it is as
we was inspired by [Remote Tea RPC](http://remotetea.sourceforge.net/) and took
lot of ideas from it including xdr language parser. The goal to be able to use
stubs generated by Remote Tea (no need to rewrite existing RPC servers) with
minimal changes.

The library supports *IPv6*, *RPCSEC_GSS* and compliant with [rfc1831](http://www.ietf.org/rfc/rfc1831.txt) and [rfc2203](http://www.ietf.org/rfc/rfc2203.txt).


### There are several options how to use *ONCRPC4J* in your application:###

Embedding service into an application
-------------------------------------

```java
package me.mypackage;

import org.dcache.xdr.RpcDispatchable;
import org.dcache.xdr.RpcCall;
import org.dcache.xdr.XdrVoid;
import org.dcache.xdr.OncRpcException;

public class Svcd {

    private static final int DEFAULT_PORT = 1717;
    private static final int PROG_NUMBER = 111017;
    private static final int PROG_VERS = 1;

    public static void main(String[] args) throws Exception {

        int port = DEFAULT_PORT;

        RpcDispatchable dummy = new RpcDispatchable() {

            @Override
            public void dispatchOncRpcCall(RpcCall call)
                          throws OncRpcException, IOException {
                call.reply(XdrVoid.XDR_VOID);
            }
        };

        OncRpcSvc service = new OncRpcSvc(port);
        service.register(new OncRpcProgram(PROG_NUMBER, PROG_VERS), dummy);
        service.start();
    }
}
```

###or with builder###

```java
package me.mypackage;

import org.dcache.xdr.RpcDispatchable;
import org.dcache.xdr.RpcCall;
import org.dcache.xdr.XdrVoid;
import org.dcache.xdr.OncRpcException;

public class Svcd {

    private static final int DEFAULT_PORT = 1717;
    private static final int PROG_NUMBER = 111017;
    private static final int PROG_VERS = 1;

    public static void main(String[] args) throws Exception {

        int port = DEFAULT_PORT;

        RpcDispatchable dummy = new RpcDispatchable() {

            @Override
            public void dispatchOncRpcCall(RpcCall call)
                          throws OncRpcException, IOException {
                call.reply(XdrVoid.XDR_VOID);
            }
        };

        OncRpcSvc service = new OncRpcSvcBuilder()
                .withTCP()
                .withAutoPublish()
                .withPort(port)
                .withSameThreadIoStrategy()
                .build();
        service.register(new OncRpcProgram(PROG_NUMBER, PROG_VERS), dummy);
        service.start();
    }
}
```

###or as a spring bean###

```java
package me.mypackage;

import org.dcache.xdr.RpcDispatchable;
import org.dcache.xdr.RpcCall;
import org.dcache.xdr.XdrVoid;
import org.dcache.xdr.OncRpcException;
import java.io.IOException;

public class Svcd implements RpcDispatchable {

    @Override
    public void dispatchOncRpcCall(RpcCall call)
                throws OncRpcException, IOException {
        call.reply(XdrVoid.XDR_VOID);
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-2.5.xsd">

    <bean id="my-rpc-svc" class="me.mypackage.Svcd">
        <description>My RPC service</description>
    </bean>

     <bean id="my-rpc" class="org.dcache.xdr.OncRpcProgram">
        <description>My RPC program number</description>
        <constructor-arg index="0" value="1110001" />
        <constructor-arg index="1" value="1" />
    </bean>

    <bean id="rpcsvc-builder" class="org.dcache.xdr.OncRpcSvcFactoryBean">
        <description>Onc RPC service builder</description>
        <property name="port" value="1717"/>
        <property name="useTCP" value="true"/>
    </bean>

    <bean id="oncrpcsvc" class="org.dcache.xdr.OncRpcSvc" init-method="start" destroy-method="stop">
        <description>My RPC service</description>
        <constructor-arg ref="rpcsvc-builder"/>
        <property name="programs">
            <map>
                <entry key-ref="my-rpc" value-ref="my-rpc-svc"/>
            </map>
        </property>
    </bean>
</beans>
```

Notice, that included *SpringRunner* will try to instantiate and run bean with id __oncrpcsvc__.


```sh
$ java -cp $CLASSPATH org.dcache.xdr.SpringRunner svc.xml
```

Using RPCGEN to generate client and server stubs
------------------------------------------------

Assume a service which calculates a the length of a string. It provides a single remote call *strlen* which takes a string as an argument ans returns it's length. Let describe that procedure according XDR language specification:

```c
/* strlen.x */
program STRLEN {
    version STRLENVERS {
        int strlen(string) = 1;
    } = 1;
} = 117;
```
Here we define *STRLEN* program number to be *117* and version number *1*. Now we can generate stub files for client and server:

```sh
$ java -jar oncrpc4j-rpcgen.jar -c StrlenClient strlen.x 
```
Simply extend this class and implement abstract methods:

```java
//StrlenSvcImpl.java
import org.dcache.xdr.*;

public class StrlenSvcImpl extends strlenServerStub {

   public int strlen_1(RpcCall call$, String arg) {
       return arg.length();
   }
}
```

Now it's ready to be complied and deployed as standalone or Spring application:

```java
// StrlenServerApp.java
// standalone application example
import org.dcache.xdr.*;

public class StrlenServerApp {

    static final int DEFAULT_PORT = 1717;

    public static void main(String[] args) throws Exception {

        OncRpcSvc service = new OncRpcSvcBuilder()
                .withTCP()
                .withAutoPublish()
                .withPort(DEFAULT_PORT)
                .withSameThreadIoStrategy()
                .build();
        service.register(new OncRpcProgram(strlen.STRLEN, strlen.STRLENVERS), new StrlenSvcImpl());
        service.start();
        System.in.read();
    }
}
```

In addition, a client will be generated as well which can be used as:

```java
// StrlenClientApp.java
import java.net.InetAddress;
import org.dcache.xdr.*;

public class StrlenClientApp {
  
    static final int DEFAULT_PORT = 1717;
    
    public static void main(String[] args) throws Exception {
        InetAddress address = InetAddress.getByName(args[0]);

        StrlenClient client = new StrlenClient(address, DEFAULT_PORT,
                strlen.STRLEN,
                strlen.STRLENVERS,
                IpProtocolType.TCP);
        System.out.println("Length of " + args[1] + " = " + client.strlen_1(args[1]));
        client.shutdown();
    }
}
```

Your RPC client and server are ready!

Use ONCRPC4J in your project
==========================

As maven dependency
------------------

```xml
<dependency>
    <groupId>org.dcache</groupId>
    <artifactId>oncrpc4j-core</artifactId>
    <version>2.1.0</version>
</dependency>

<repositories>
    <repository>
        <id>dcache-snapshots</id>
        <name>dCache.ORG maven repository</name>
        <url>https://download.dcache.org/nexus/content/repositories/releases</url>
        <layout>default</layout>
    </repository>
</repositories>
```
