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

        OncRpcSvc service = new OncRpcSvcBuilder()
                .withTCP()
                .withAutoPublish()
                .withPort(port)
                .withSameThreadIoStrategy()
                .withRpcService(new OncRpcProgram(PROG_NUMBER, PROG_VERS), dummy)
                .build();
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
        <property name="rpcServices">
            <map>
                <entry key-ref="my-rpc" value-ref="my-rpc-svc"/>
            </map>
        </property>

    </bean>

    <bean id="oncrpcsvc" class="org.dcache.xdr.OncRpcSvc" init-method="start" destroy-method="stop">
        <description>My RPC service</description>
        <constructor-arg ref="rpcsvc-builder"/>
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
                .withRpcService(new OncRpcProgram(strlen.STRLEN, strlen.STRLENVERS), new StrlenSvcImpl())
                .build();
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
    <version>2.5.0</version>
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

Accessing client subject inside RPC service
-------------------------------------------
In some situation, OncRpcSvc can internally call other services which require client subject to be set in the
context of the current thread. We use standard Java's Subject.doAs() mechanism to inject user subject
into processing thread. As a result, the user subject can be extracted from AccessControlContext.

```java
// SomeService.java
import javax.security.auth.Subject;
import java.security.AccessController;

public class SomeService {
    public void doSomeTask() {
        Subject subject = Subject.getSubject(AccessController.getContext());
        // start using subject
    }
}

// SubjectAvareSvcImpl.java
public class SubjectAvareSvcImpl implements RpcDispatchable {
    @Override
    public void dispatchOncRpcCall(RpcCall call)
                throws OncRpcException, IOException {
        externalService.doSomeTask();
        call.reply(XdrVoid.XDR_VOID);
    }
}
```

To avoid unnecessary overhead, subject propagation is not enabled by default:
```java
OncRpcSvc service = new OncRpcSvcBuilder()
        .withTCP()
        .withAutoPublish()
        .withSameThreadIoStrategy()
        .withRpcService(... , new SubjectAvareSvcImpl())
        .withSubjectPropagation()
        .build();
```

How to contribute
=================


**oncrpc4j** uses the linux kernel model of using git not only a source
repository, but also as a way to track contributions and copyrights.

Each submitted patch must have a "Signed-off-by" line.  Patches without
this line will not be accepted.

The sign-off is a simple line at the end of the explanation for the
patch, which certifies that you wrote it or otherwise have the right to
pass it on as an open-source patch.  The rules are pretty simple: if you
can certify the below:
```

        Developer's Certificate of Origin 1.1

        By making a contribution to this project, I certify that:

        (a) The contribution was created in whole or in part by me and I
            have the right to submit it under the open source license
            indicated in the file; or

        (b) The contribution is based upon previous work that, to the best
            of my knowledge, is covered under an appropriate open source
            license and I have the right under that license to submit that
            work with modifications, whether created in whole or in part
            by me, under the same open source license (unless I am
            permitted to submit under a different license), as indicated
            in the file; or

        (c) The contribution was provided directly to me by some other
            person who certified (a), (b) or (c) and I have not modified
            it.

	(d) I understand and agree that this project and the contribution
	    are public and that a record of the contribution (including all
	    personal information I submit with it, including my sign-off) is
	    maintained indefinitely and may be redistributed consistent with
	    this project or the open source license(s) involved.

```
then you just add a line saying ( git commit -s )

	Signed-off-by: Random J Developer <random@developer.example.org>

using your real name (sorry, no pseudonyms or anonymous contributions.)

