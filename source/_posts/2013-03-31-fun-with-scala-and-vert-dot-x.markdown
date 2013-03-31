---
layout: post
title: "Fun with Scala and Vert.x"
date: 2013-02-12 18:43
comments: true
categories: [Scala, Vert.x]

keywords: Scala, Vert.x, WebSocket, Http Server, Http Client
description: Vert.x using Scala power
---

<a href="http://vertx.io/">Vert.x</a> is a polyglot event-driven application framework that runs on the Java Virtual Machine (JAVA 7 is the minimum supported version). Like Node.js, Vert.x is asynchronous and scalable, and lets developers build modern and effective web applications.

Being polyglot, Vert.x can be used in many flavors, among which : JavaScript,CoffeeScript, Ruby, Python,Groovy and Java. In order to enforce asynchronism and scalabity, Vert.x is built upon <a href="https://netty.io/">Netty</a>, leverage the <a href="http://en.wikipedia.org/wiki/Reactor_pattern">reactor pattern</a>, using a frightening number of handlers.

This article aims at showing the powerful combination of Scala and Vert.x - the Java counterpart being provided as comparison.

Note : the source code are hosted on GitHub as part of lang-scala <a href="https://github.com/ouertani/vert.x/tree/master/vertx-lang/vertx-lang-scala">https://github.com/ouertani/vert.x/tree/master/vertx-lang/vertx-lang-scala</a>

Vert.x supports many components :

<li><a href="http://en.wikipedia.org/wiki/WebSocket">WebSocket</a></li>
<li>HttpServer</li>
<li>Distributed Event Bus</li>
<li>TCP Server, SockJS ,... not presented here</li>
<h2>I - WebSocket</h2>

Web Socket are HTML 5 feature providing full-duplex communications. For old browsers that do not support WebSocket, Vert.x provides <a href="https://github.com/sockjs/sockjs-client">SockJS</a> as out-of-the-box component.

In order to run the following example check out : <a href="https://github.com/ouertani/vert.x/blob/master/vertx-examples/src/main/javascript/websockets/ws.html">https://github.com/ouertani/vert.x/blob/master/vertx-examples/src/main/javascript/websockets/ws.html</a> and save it into the compiled lib directory

<h4>Java</h4>
```	java SampleWebSocket.java
import org.vertx.java.core.Handler;
import org.vertx.java.core.buffer.Buffer;
import org.vertx.java.core.http.HttpServerRequest;
import org.vertx.java.core.http.ServerWebSocket;
import org.vertx.java.deploy.Verticle;
 
public class SampleWebSocket extends Verticle {
 
  public void start() {
    vertx.createHttpServer().websocketHandler(new Handler<ServerWebSocket>() {
      public void handle(final ServerWebSocket ws) {
        if (ws.path.equals("/myapp")) {
          ws.dataHandler(new Handler<Buffer>() {
            public void handle(Buffer data) {
              ws.writeTextFrame(data.toString()); // Echo it back
            }
          });
        } else {
          ws.reject();
        }
      }
    }).requestHandler(new Handler<HttpServerRequest>() {
      public void handle(HttpServerRequest req) {
        if (req.path.equals("/")) req.response.sendFile("ws.html"); // Serve the html
      }
    }).listen(8080);
  }
}
```
<h4>Scala</h4>
```	scala SampleWebSocket.scala
import org.vertx.java.core.buffer.Buffer;
import org.vertx.java.core.http.HttpServerRequest;
import org.vertx.java.core.http.ServerWebSocket;
import org.vertx.scala.deploy.Verticle
import org.vertx.scala.core._
 
class SampleWebSocket extends Verticle ( 
  _.getVertx().createHttpServer().websocketHandler{ 
     ws:ServerWebSocket => ws.path match {
        case "/myapp" => ws.dataHandler{data : Buffer =>   ws.writeTextFrame(data.toString())}
        case _ => ws.reject();
      }}
      .requestHandler{req : HttpServerRequest => req.path match {case ("/") => req.response.sendFile("ws.html")}}
      .listen(8080)
)()
```
<h2>II-HttpWebServer</h2>

Vert.x allows you to easily write full featured, highly performant and scalable HTTP and HTTPS servers.

The following example starts up an Http server, listening on port 8080, and logging all received requests.

<h4>Java</h4>
```	java SampleHttpWebServer.java
import org.vertx.java.core.Handler;
import org.vertx.java.core.http.HttpServer;
import org.vertx.java.core.http.HttpServerRequest;
import org.vertx.java.core.logging.Logger;
import org.vertx.java.deploy.Verticle;
 
 
public class SampleHttpWebServer extends Verticle {
 
    public void start() {
        HttpServer server = vertx.createHttpServer();
        final Logger log = getContainer().getLogger();
        server.requestHandler(new Handler<HttpServerRequest>() {
            public void handle(HttpServerRequest request) {
                log.info("A request has arrived on the server!");
            }
        }).listen(8080, "localhost");
    }
}
```
<h4>Scala</h4>
```	scala SampleWebServer.scala
import org.vertx.java.core.http. { HttpServerRequest => JHttpServerRequest}
import org.vertx.scala.deploy.Verticle
import org.vertx.scala.core._
 
class SampleWebServer extends Verticle ( x => 
  x.getVertx().createHttpServer().withRequestHandler{_ : JHttpServerRequest => 
     x.info("A request has arrived on the server!")}
.listen(8080, "localhost")
)()
```
<h2>III-HttpClient</h2>

Vert.x also provides an HttpClient API, so as to interact with the server part. The following samples create and send a GET request, then log the server's response.

<h4>Java</h4>
```	java SampleHttpClient.java
import org.vertx.java.core.Handler;
import org.vertx.java.core.http.HttpClient;
import org.vertx.java.core.http.HttpClientRequest;
import org.vertx.java.core.http.HttpClientResponse;
import org.vertx.java.core.logging.Logger;
import org.vertx.java.deploy.Verticle;
 
 
public class SampleHttpClient extends Verticle {
 
    @Override
    public void start() throws Exception {
        HttpClient client = vertx.createHttpClient().setHost("127.0.0.1");
        final Logger log = getContainer().getLogger();
 
        HttpClientRequest request = client.post("/some-path/", new Handler<HttpClientResponse>() {
            public void handle(HttpClientResponse resp) {
                log.info("Got a response: ");
            }
        });
 
        request.end();
    }
}
```
<h4>Scala</h4>
```	scala SampleWebClient.scala
import org.vertx.scala.deploy.Verticle
import org.vertx.scala.core._
 
class SampleWebClient extends Verticle (v =>
   v.getVertx.createHttpClient().setHost("127.0.0.1").setPort(8080)
   .andGetNow("/") {_ => v.info("Got a response: " )}
  )()
```
<h2>IV-EventBus</h2>

The event bus is like a vertebral spine, it can be used to connect distributed nodes, and to support interaction between different Verticles, even written in different languages.

<h4>Java</h4>
```	java SampleEventBus.java
import org.vertx.java.deploy.Verticle;
import org.vertx.java.core.eventbus.EventBus;
 
public class SampleEventBus extends Verticle {
   @Override
    public void start() throws Exception {
       EventBus eb = vertx.eventBus();
       eb.send("path", "ping1");
       eb.send("path", "ping2");
   }  
}
```
<h4>Scala</h4>
```	scala SampleEventBus.scala
import org.vertx.scala.deploy.Verticle
import org.vertx.scala.core._
class SampleEventBus extends Verticle ( x => { 
 val  point = x ! ("path")
 point >> ("ping 1") 
 point >> ("ping 2") 
} 
)()
```
<h2>Conclusion</h2>

This article introduced the basic of Vert.x using scala language and short examples. Full Scala language support will soon, hopefully, become available.

Stay tuned, a subsequent post will show you more about Vert.x with Scala.

source : http://blog.zenika.com/index.php?post/2013/02/11/fun-with-scala-and-vert-x