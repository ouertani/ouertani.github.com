---
layout: post
title: "Rest with Scala and Vert.x"
date: 2013-03-09 00:24
comments: true
categories: [Scala, Vert.x, Rest]

keywords: Rest, Scala, Vert.x, String interpolation, Resticle, Web Service
description: Rest Web Service with Vert.x using Scala
---

A previous post introduced some features of using Scala with <a href="http://vertx.io/">Vert.x</a>. This post covers how to publish Rest web services in an elegant and simple fashion.

As in the previous post, Examples in Java and Scala are presented, source code been hosted on GitHub as part of lang-scala <a href="https://github.com/ouertani/vert.x/tree/master/vertx-lang/vertx-lang-scala">https://github.com/ouertani/vert.x/tree/master/vertx-lang/vertx-lang-scala</a>
<!-- more -->

<h2>I- Resticle</h2>

Resticle is a unit of deployment dedicated to Rest. It provides a new method (“handles”) which can be used by Vert.x to start Rest services. “handles” returns a sequence of rest handlers :

<ul>
<li>RestHandler : Action => Response</li>
<li>Action :</li><ul>
<li>GET(pattern : String)</li>
<li>POST (pattern : String)</li>
<li>PUT (pattern : String)</li>
<li>DELETE(pattern : String)</li>
<li>HEAD (pattern : String)</li>
<li>ALL(pattern : String)</li>
</ul>
<li>Response : HttpServerRequest => Any</li>
<li>OK, Unauthorized,....</li>
</ul>
In this tutorial we will be working with the SampleResticle class, for both scala and java.

<h2>II - Hello World</h2>

Vert.x provides a powerful routing and <a href="http://vertx.io/core_manual_java.html#routing-http-requests-with-pattern-matching">route matching</a> mechanism, which simplifies the routing of HTTP requests to different handlers based on pattern matching on the request path.

In Hello World snippet, let us publish a static GET service :

<li>GET : /hello → code : 200 , body : world</li>
<h4>Scala</h4>

```scala	SampleResticle.scala

class SampleResticle extends Resticle
{
  override def handles =
         { GET("/hello")      :>  OK( _ => "world ") }
}
```
<h4>Java</h4>
```	java	SampleResticle.java			
public class SampleResticle extends Verticle {

    public void start() {
        HttpServer server = vertx.createHttpServer();
        RouteMatcher routeMatcher = new RouteMatcher();
        routeMatcher.get("/hello", new Handler<HttpServerRequest>() {
            public void handle(HttpServerRequest req) {
                req.response.statusCode =200;
                req.response.end("world");
            }
        });
        server.requestHandler(routeMatcher).listen(8080, "localhost");
    }
}
```
<h2>III- Chaining Handlers</h2>

Using Resticle we can chain handlers quit easily. The following snippets create static GET and DELETE services :

<li>GET : /hello → code : 200 , body : world</li>
<li>DELETE : /posts → code : 401 , body : Not allowed user</li>
<h4>Scala</h4>

```	scala	SampleResticle.scala				
class SampleResticle extends Resticle
{
  override def handles =
         { GET("/hello")      :>  OK( _ => "world ") } ++
         { DELETE("/posts")   :>  Unauthorized {_ => "Not allowed user" }}
}
```
<h4>java</h4>

```	java	SampleResticle.java				
public class SampleResticle extends Verticle {

    public void start() {
        HttpServer server = vertx.createHttpServer();
        RouteMatcher routeMatcher = new RouteMatcher();
        routeMatcher.get("/hello", new Handler<HttpServerRequest>() {
            public void handle(HttpServerRequest req) {
                req.response.statusCode =200;
                req.response.end("world");
            }
        });
         routeMatcher.delete("/posts", new Handler<HttpServerRequest>() {
            public void handle(HttpServerRequest req) {
                req.response.statusCode =401;
                req.response.end("Not allowed user");
            }
        });
        server.requestHandler(routeMatcher).listen(8080, "localhost");
    }
}
```
<h2>IV - Value path</h2>

Vert.x <a href="http://vertx.io/core_manual_java.html#routing-http-requests-with-pattern-matching">pattern matching</a> lets you extract values from the path and use them as parameters in the request.

<li>GET : /hello → code : 200 , body : world</li>
<li>DELETE : /posts → code : 401 , body : Not allowed user</li>
<li>POST : /:blogname → code : 200 , body : post {blogname} received !</li>


<h4>Scala : ( Using <a href="http://docs.scala-lang.org/overviews/core/string-interpolation.html">String interpolation</a>)</h4>

```	scala	SampleResticle.scala				
class SampleResticle extends Resticle
{
  override def handles =
         { GET("/hello")      :>  OK( _ => "world ") } ++
         { DELETE("/posts")   :>  Unauthorized {_ => "Not allowed user" }} ++
         { POST("/:blogname") :>  OK {req  => val param = req.params().get("blogname") ; s"post $param received !" } }
}
```
<h4>Java</h4>

```	java	SampleResticle.java				
public class SampleResticle extends Verticle {

    public void start() {
        HttpServer server = vertx.createHttpServer();
        RouteMatcher routeMatcher = new RouteMatcher();
        routeMatcher.get("/hello", new Handler<HttpServerRequest>() {
            public void handle(HttpServerRequest req) {
                req.response.statusCode =200;
                req.response.end("world");
            }
        });
         routeMatcher.delete("/posts", new Handler<HttpServerRequest>() {
            public void handle(HttpServerRequest req) {
                req.response.statusCode =401;
                req.response.end("Not allowed user");
            }
        });

         routeMatcher.post("/:blogname", new Handler<HttpServerRequest>() {
            public void handle(HttpServerRequest req) {
                req.response.statusCode =200;
                String blogName = req.params().get("blogname");
                req.response.end("post "+blogName+ " received !");
            }
        });

        server.requestHandler(routeMatcher).listen(8080, "localhost");
    }
}
```
<h2>V- Json</h2>

Let’s assume we have a Blog class with two String fields title and content :


case class Blog (title :String , content : String)
The java equivalent has been relocated to the end of the document due to its verbosity ;).

Publishing an object using Resticle is simple and transparent due to implicit convertor : T => Buffer.

<h4>Scala ( Type Class)</h4>
```	scala	Blog.scala
object Blog {
  implicit def toBuffer(blog : Blog):Buffer = JsonObject.withString("title" -> blog.title).withString("content" -> blog.content)
}
```
<h4>Java ( with explicit convertor )</h4>

```	java	Convertor.java				
public class Convertor {
    public static  JsonObject toJson(Blog blog) {
       return new JsonObject().putString("title", blog.getTitle()).putString("content", blog.getContent());
    }
}
```
<li>GET : /hello → code : 200 , body : world</li>
<li>DELETE : /posts → code : 401 , body : Not allowed user</li>
<li>POST : /:blogname → code : 200 , body : post {blogname} received !</li>
<li>GET : /:id → code : 200 , body : {"title":"rest","content":"scala & vertx"}</li>
<h4>Scala</h4>

```	scala	SampleResticle.scala				
class SampleResticle extends Resticle
{
  override def handles =
         { GET("/hello")      :>  OK( _ => "world ") } ++
         { DELETE("/posts")   :>  Unauthorized {_ => "Not allowed user" }} ++
         { POST("/:blogname") :>  OK {req  => val param = req.params().get("blogname") ; s"post $param received !" } } ++
         { GET("/:id")        :>  OK ( _ => Blog("rest","scala & vertx"))}
}
```
<h4>Java</h4>

```	java	SampleResticle.java				
public class SampleResticle extends Verticle {

    public void start() {
        HttpServer server = vertx.createHttpServer();
        RouteMatcher routeMatcher = new RouteMatcher();
        routeMatcher.get("/hello", new Handler<HttpServerRequest>() {
            public void handle(HttpServerRequest req) {
                req.response.statusCode =200;
                req.response.end("world");
            }
        });
         routeMatcher.delete("/posts", new Handler<HttpServerRequest>() {
            public void handle(HttpServerRequest req) {
                req.response.statusCode =401;
                req.response.end("Not allowed user");
            }
        });

         routeMatcher.post("/:blogname", new Handler<HttpServerRequest>() {
            public void handle(HttpServerRequest req) {
                req.response.statusCode =200;
                String blogName = req.params().get("blogname");
                req.response.end("post "+blogName+ " received !");
            }
        });

        routeMatcher.get("/:id", new Handler<HttpServerRequest>() {
            public void handle(HttpServerRequest req) {
                req.response.statusCode =200;
                Blog blog =  new Blog("rest","scala & vertx");
                JsonObject obj = Convertor.toJson(blog);
                req.response.end(obj.encode());
            }
        });

        server.requestHandler(routeMatcher).listen(8080, "localhost");
    }
}
```
Java Blog

```	java	Blog.java				
public class Blog {

    private final  String title;
    private final String content;

    public Blog(String title, String content) {
        this.title = title;
        this.content = content;
    }

    public String getTitle() {
        return title;
    }

    public String getContent() {
        return content;
    }
  }
```
<h2>Conclusion</h2>

Despite the fact that Resticle is in first development step, Rest support is by far simpler and elegant in scala than in java. As described in first tutorial Vert.x java version is burdened with a frightening number of handlers. Will Vert.x 2.0 address this point using <a href="https://github.com/vert-x/vert.x/wiki/Vert.x-2.0-plan">Promises/Deferred APIs </a>?
