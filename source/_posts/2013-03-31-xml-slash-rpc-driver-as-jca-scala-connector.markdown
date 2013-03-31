---
layout: post
title: "XML/RPC driver as JCA Scala connector"
date: 2012-09-26 19:41
comments: true
categories: [JCA, JEE, SCALA, XML, UCIP, RPC]

keywords: JCA, JEE, SCALA, XML, UCIP, RPC
description: JCA JEE Connecter using UCIP Scala driver
---
Interoperability and reusability are key features of SOA architecture.  

The <a href="http://en.wikipedia.org/wiki/Java_EE_Connector_Architecture">Java EE Connector architecture</a> defines a standard architecture for connecting the Java EE platform to heterogeneous EISs. This article presents an XML/RPC  adapter using a Scala JCA outbound connector to an IN/AIR legacy system.

<h2>JCA and integration</h2>

{% blockquote JCA 1.6 p 35 %}
 "For enterprise application integration, bi-directional connectivity between enterprise applications and EIS is essential. The Java EE Connector architecture defines standard contracts that allow bi-directional connectivity between enterprise applications and EISs. It also formalizes the relationships, interactions, and the packaging of the integration layer, thus enabling enterprise application integration." 

{% endblockquote %}

The connector architecture defines a set of scalable, secure, and transactional mechanisms that enable the integration of EISs with application servers and enterprise application.

{% img center /images/jca.jpg 350 350 'JCA' 'images' %}
 
Using a UCIP JCA connector rather than using an <a href="http://java.dzone.com/articles/xml-rpc-using-scala">XML/RPC raw driver</a> lets you:
<ol>
<li>Hide connection complexity.</li>
<li>Use connection pooling and scalability.</li>
<li>Use a standard adapter that can be deployed with any JEE 6 server from an m x n integration problem to an m + n solution</li>
</ol>
<h2>How?</h2>
The use of a <a href="http://www.jcp.org/en/jsr/detail?id=322">JCA</a> resource adapter inside a JEE solution is the same as interacting with a database or queue: protocol communication and wire negotiation... are hidden to the final user.  

Two interfaces are presented to a customer:

<h3>1) A factory trait:</h3>
```scala
trait AirConnectorFactory extends Referenceable with Function0[AirConnector]
```
<h3>2) The connector trait:</h3>
```	scala
trait AirConnector {
def fire(elem : Elem) : Option[Elem]
}
```
As an outbound communication where the resource adapter allows an ESB or EE application server to connect to an IN/AIR node and perform work. All communication is initiated by the application. The Air connector factory should be injected as any resource and used like the following.
```	scala
@Resource(name="AIR")
var airConnectorFactory : AirConnectorFactory = _

val output = airConnectorFactory().fire(input)
```
{% blockquote %}
Source code is based on JCA 1.6 specification and hosted on <a href="https://github.com/ouertani/TelcoCX">github</a>.
{% endblockquote %}