---
layout: post
title: "EE 6 Environmental Enterprise Entries and Glassfish"
date: 2012-06-24 02:55
comments: true
categories: [JEE, JNDI]
keywords: JEE, JNDI
description: JCA JEE Connecter using UCIP Scala driver
---
<h2>PURPOSE</h2>
This tutorial is a supplement to the article of oracle published <a href="http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/env_entry/env_entry.html.">here</a>. After reading <a href="http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/env_entry/env_entry.html.">this later</a>, I decided to share some tips about this EE environment configuration.
<h2>INTRODUCTION</h2>

Two years ago, we decided to move to JEE 6 for our enterprise solutions and take a lot of fun with new EJB 3.1 features and annotations. Glassfish has made our lives easier, it was very easy to declare variables using it's administration console.
<!-- more -->
Less complicated than using <b>ejb-jar.xml</b> are two methods using Glassfish server and resources :
<h2>I- OLD FASHION WITH RESOURCE PER PARAM (SIMILAR TO ORACLE TUTORIAL) </h2>
<ol>
<li>Declare an EJB with toBeInjected resource :</li>
```java
import javax.annotation.Resource;
import javax.ejb.Stateless;
import javax.ejb.LocalBean;

/**
 *
 * @author slim
 */
@Stateless
@LocalBean
public class MyEjbBean {
  @Resource(lookup="resName")
  protected String toBeInjected;


  ////

}
```
<li>Open admin console ( localhost:4848 by default ) -> Resources -> Custom Resources -> New -> and fill the informations as below</li>
</ol>
![JNDI EE](/images/jndiee1.jpg)

At first glance, it looks simple and fast forward but problems start after just after a first project.

<h3>Problems ! Why ?</h3>
We achieved more than 20 variables and some resource names begin to confuse them. I was very hard for our operations team to catch the name of the correct setting and update its value.
It was necessary to group parameters resources into a single resource by project.

<h2>NEXT FASHION : GROUPING RESOURCES </h2>

In this section I will use scala (similar class can be translated easy to java).
<ol>
<li>First of all, we need to create our single resource class as simple bean</li>

```java
package me.slim.ouertani
class Ressource {
  var param1 : String = _
  var parma2 : String = _
}
```

<li>Next, let's create a resource factory class :</li>
</ol>
```java
package me.slim.ouertani
class RessourceFactory extends ObjectFactory {

  override def getObjectInstance(obj: Object, name: javax.naming.Name, nameCtx: javax.naming.Context, environment: Hashtable[_, _]): Object = {
    val ressource = new Ressource();
    val reference = obj.asInstanceOf[Reference];
    val attributes = reference.getAll();
    while (attributes.hasMoreElements()) {
      val refAddr = attributes.nextElement().asInstanceOf[RefAddr];
      init(ressource, refAddr.getType(), refAddr.getContent());
    }
    return ressource;
  }

  private[this] def init(ressource: Ressource, tipe: String, content: Object) {
    tipe match {  

      case "param1" => ressource.param1 = content.toString()
      case "param2" => ressource.param2 = content.toString()

    }
  }
}
```
The purpose of object factory is to read injectable parameters and populate a resource class

<li>Back to Glassfish admininstration console, create a single resource ( see screenshot below) :</li>
<ol>
	<li>Jndi Name => the resource name to be used by project for example
	<li>Resource Type => check second radio button, and fill the resource full name ( with package)
	<li>Factory class => Our RessourceFactory class full name ( with package)
	<li>Add two (or more) properties to be injected in our resource class.</ol>
![JNDI EE](/images/jndiee2.jpg)

<li>At the end we inject our resource in EJB class :</li>

```java
@Stateless
@LocalBean
class MyEjbBean {
  @Resource(lookup="globalRes")
  protected var res:Ressource= _

  /// res.param1
}
```
<h2>CONCLUSION</h2>

Having a single resource per project facilitates business management and thanks to Glassfish administration console managing JNDI became easier.
