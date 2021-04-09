---
layout: post
title: "Basic WS with Scala"
date: 2011-09-10 03:55
comments: true
categories: [JEE, JAX-WS, SCALA, Sbt]
keywords: Scala, JEE, JAX-WS
description: JAX-WS using scala and sbt
---
I recently had a chance to work on building SOA platforme using JEE 6 and Scala language. I will share here some snippets my be helpfull to start publishing web services using Scala, Sbt and JAX-WS.
Consider the example to calculate the quotient and remainder using euclidean algorithm. Not very complexe if the inputs are positives numbers.
<!-- more -->
## I- Sbt

Let’s create an sbt projet and add required dependencies
<blockquote>
$ mkdir sws
$ cd sws
$ touch build.sbt
$ edit build.sbt as following :
</blockquote>
```scala
name := "Euclide"

version := "1.0"

organization := "me.ouertani"

scalaVersion := "2.9.1"

scalaSource in Compile <<= baseDirectory(_ / "src")

scalaSource in Test <<= baseDirectory(_ / "test")

libraryDependencies += "javax" % "javaee-api" % "6.0" % "provided"

libraryDependencies += "log4j" % "log4j" % "1.2.16" % "provided"

libraryDependencies += "junit" % "junit" % "4.8.2" % "test"
```
<blockquote>
$ mkdir src
$ touch src/Euclide.scala
</blockquote>

That’a all for project configuration.

## II- Create the WS

We will use a single Euclide.scala file for all class this will be easier for visibility

- Create a Request Class and don’t forget :
	- default constructor
	- field access annotation

```scala
@XmlAccessorType(XmlAccessType.FIELD)
case class Request(a : Int, b : Int ){ def this(){this(0,0)}}
```
- Create a Response Class
```scala
@XmlAccessorType(XmlAccessType.FIELD)
@XmlType(name = "output")
case class Response (c: Int, d: Int){ def this(){this(0,0)}}
```
- Why not to create an exception class to handle exceptions like b is equal to 0
```scala
@XmlAccessorType(XmlAccessType.FIELD)
case class ResponseException(why : String){  def this(){this("")}}

@XmlAccessorType(XmlAccessType.FIELD)
case class FaultResponse(@BeanProperty faultInfo : ResponseException) extends Exception { def this(){this(ResponseException(""))}}
```
Don’t forget to add BeanProperty annotation to faultInfo input !

	- Let’s add an interceptor to log all ws method call

```scala
object TracingInterceptor{
  import org.apache.log4j.Logger
  val L = Logger.getLogger(classOf[TracingInterceptor])
}

import TracingInterceptor._
class TracingInterceptor {

  @AroundInvoke
  def  log( context:InvocationContext):Object ={
    try {
      if (L.isDebugEnabled()) {
        val clazz = context.getMethod().getDeclaringClass().getName()
        val method = context.getMethod().getName()        
        L.debug(clazz + " : Is invoking method: " + method + " With Parameters : " + context.getParameters().mkString("[", ",","]"))
      }
    } catch {
      case  ex :Exception =>    L.warn("Unable to Intercept Method Call", ex)
    }
    context.proceed()
  }
}
```
	- The WS class and it’s divide method using intercpetor and stateless EJB 3.1
```scala
@WebService(serviceName = "Euclide", targetNamespace = "http://slim.ouertani.me/")
@Stateless()
@Interceptors(Array(classOf[TracingInterceptor]))
class Euclide {
  @throws(classOf[FaultResponse])
  def divide( @WebParam(name="input" )  @XmlElement(required = true, nillable = false)
    req :Request):Response =  req match {
      case Request(_, 0 ) => throw FaultResponse(ResponseException("Can't divide by ZERO!"))
      case Request(a, b ) =>  Response(a / b ,a % b)
    }
}
```
	- Don’t forget package , imports at the head of file and it’s all:

```scala
package me.ouertani.scala.ws

import javax.interceptor.Interceptors
import javax.xml.bind.annotation.XmlElement
import javax.interceptor.{ AroundInvoke , InvocationContext }
import javax.xml.bind.annotation. { XmlAccessType , XmlAccessorType, XmlElement, XmlType}
import javax.ejb. { EJB ,Stateless }
import javax.jws. {WebService , WebParam }
import scala.reflect.BeanProperty
```
<h2>III- packaging and run WS</h2>
<ol>
	<li>Let’s start compiling our projet</li>
$ xsbt compile
	<li>package the jar</li>
$ xsbt package
	<li>deploy it to your JEE 6 container ( mine is Glassfish 3.1.1 )</li>
</ol>
<h2>IV – test it</h2>
1- After depoyed let’s test it using soapUI tool
Request 1
```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:slim="http://slim.ouertani.me/">
   <soapenv:Header/>
   <soapenv:Body>
      <slim:divide>
         <input>
            <a>6</a>
            <b>4</b>
         </input>
      </slim:divide>
   </soapenv:Body>
</soapenv:Envelope>
```

Response 1 :
```xml
<S:Envelope xmlns:S="http://schemas.xmlsoap.org/soap/envelope/">
   <S:Body>
      <ns2:divideResponse xmlns:ns2="http://slim.ouertani.me/">
         <return>
            <c>3</c>
            <d>0</d>
         </return>
      </ns2:divideResponse>
   </S:Body>
</S:Envelope>
```
Request 2 :
```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:slim="http://slim.ouertani.me/">
   <soapenv:Header/>
   <soapenv:Body>
      <slim:divide>
         <input>
            <a>6</a>
            <b>0</b>
         </input>
      </slim:divide>
   </soapenv:Body>
</soapenv:Envelope>
```
Response 2 :
```xml
<S:Envelope xmlns:S="http://schemas.xmlsoap.org/soap/envelope/">
   <S:Body>
      <S:Fault xmlns:ns4="http://www.w3.org/2003/05/soap-envelope">
         <faultcode>S:Server</faultcode>
         <faultstring>me.ouertani.scala.ws.FaultResponse</faultstring>
         <detail>
            <ns2:FaultResponse xmlns:ns2="http://slim.ouertani.me/">
               <faultInfo>
                  <why>Can't divide by ZERO!</why>
               </faultInfo>
            </ns2:FaultResponse>
         </detail>
      </S:Fault>
   </S:Body>
</S:Envelope>
```
Fine!
