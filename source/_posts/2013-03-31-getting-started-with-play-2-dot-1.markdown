---
layout: post
title: "Play 2.1 Scala 2.10 and String Interpolation"
date: 2012-10-18 19:00
comments: true
categories: [Play, Scala]

keywords: Play 2.1, Scala 2.10, String interpolation
description: Play 2.1 using Scala 2.10 String interpolation
---

Play Roadmap has been <a href="https://docs.google.com/document/d/1OEt6gZ3a-daSkNXqXGAM4jBs5LtuDkLZIzsWN9aeM1g/preview?sle=true">published</a>. The major feature will be the support of <a href="http://www.scala-lang.org/">scala 2.10</a> version.


By using <a href="http://www.playframework.org/">Play 2.1</a> we can start profiting with <a href="https://docs.google.com/document/d/1NdxNxZYodPA-c4MLr33KzwzKFkzm9iW9POexT9PkJsU/edit">String interpolation</a>



<h2>STRING INTERPOLATION: QUICK EXAMPLE ? </h2>

```	scala
val n = 20
 
"Bob is "+n+" years old"
```
can be replaced by :
```	scala
s"Bob is $n years old"
```
The benefit of using String interpolation are : 

<li>one or two characters in less</li>
<li>compile time check</li>
<li>compile time transformation ( thanks to <a href="http://scalamacros.org/">macros</a>)</li>

<h2>PLAY 2.1</h2>

On Play framework an sample action can be :
```	scala
def index = Action {
  val version = 2.10
  Ok("hello scala "+ version)
}
```
And by String interpolation we can rewrite it as :
```	scala
def index = Action {
  val version = 2.10
  Ok(s"hello scala $version")
}
```
<h2>BETTER ! </h2>


Let's do better and remove parenthesizes and 's' characters

First create an <a href="http://docs.scala-lang.org/sips/pending/implicit-classes.html">implicit class</a> (scala 2.10 features) to wrap a call to Ok object and define an ok object :
```	scala
implicit class HTTPInterpolation(s: StringContext) {
   object ok {
     def apply(exprs: Any*) = {
       Ok(s.s(exprs: _*))
     }
   }
 }
```
Now we can use it as 
```	scala
def index = Action {
    val version = 2.10
    ok"hello scala $version"
}
```
Short and concise ! 

<h2>REFERENCES :</h2>

<li>Play RaodMap: <a href="https://docs.google.com/document/d/1OEt6gZ3a-daSkNXqXGAM4jBs5LtuDkLZIzsWN9aeM1g/preview?sle=true.">https://docs.google.com/document/d/1OEt6gZ3a-daSkNXqXGAM4jBs5LtuDkLZIzsWN9aeM1g/preview?sle=true.</a></li>
<li>String interpolation : <a href="https://docs.google.com/document/d/1NdxNxZYodPA-c4MLr33KzwzKFkzm9iW9POexT9PkJsU/edit">https://docs.google.com/document/d/1NdxNxZYodPA-c4MLr33KzwzKFkzm9iW9POexT9PkJsU/edit</a></li>
<li>Implicit Class : <a href="http://docs.scala-lang.org/sips/pending/implicit-classes.html">http://docs.scala-lang.org/sips/pending/implicit-classes.html</a>
<li>Macro : <a href="http://scalamacros.org/">http://scalamacros.org/</a></li>
<li>String interpolation with concise samples : <a href="http://www.blog.project13.pl/index.php/coding/1540/scala-2-10-0-hello-string-interpolation/">http://www.blog.project13.pl/index.php/coding/1540/scala-2-10-0-hello-string-interpolation/</a></li>