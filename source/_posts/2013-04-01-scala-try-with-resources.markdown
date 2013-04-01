---
layout: post
title: "Scala try-with-resources"
date: 2010-12-06 04:29
comments: true
categories: [SCALA]


keywords: SCALA, try-with-ressource, JAVA 7
description: step by step to implement try-with ressource java 7 features using scala
---
Have you ever had this duplication of code in java?
<!-- more -->
```java
public void doQuery(String q) throws SQLException {
       PreparedStatement preparedStatement = null;
       ResultSet resultSet = null;     
       Connection conn = null;
       try {
           conn = getConnection();
           preparedStatement = conn.prepareStatement(q);        
           resultSet = preparedStatement.executeQuery();
           
 
       } finally {
           close(resultSet);
           close(preparedStatement);
           close(conn);           
       }
   }
 
      private static void close(Connection connection) {
       try {
           if (connection != null) {
               connection.close();
           }
       } catch (Exception e) {
          error( e);
       }
   }
 
   private static void close(PreparedStatement ps) {
       try {
           if (ps != null) {
               ps.close();
           }
       } catch (Exception e) {
          error( e);
       }
   }
 
   private static void close(ResultSet rs) {
       try {
           if (rs != null) {
               rs.close();
           }
       } catch (Exception e) {
          error( e);
       }
   }
```

we have duplicated the close method (using cascading try or not ) because Connection, PreparedStatement and ResultSet does not share the same interface (may be the future AutoCloseable interface). 

Next java version will introduce try-with ressource wich. We try to give the same behavior using scala futures.

<h2>I- BASIC ENHANCEMENT : ONE CLOSE METHOD</h2>
```scala
@throws(classOf[SQLException])
def doQuery( q:String)   {
  var  preparedStatement:PreparedStatement = null;
  var resultSet:ResultSet = null;
  var conn:Connection = null;
  try {
    conn = getConnection()
    preparedStatement = conn.prepareStatement(q)
    resultSet = preparedStatement.executeQuery()
  } finally {
    close(resultSet)
    close(preparedStatement)
    close(conn)
    
  }
}
def getConnection():Connection=null
 
def close(x : {def close()}){
  try {
          if (x != null) {
              x.close();
          }
      } catch  {
         case  e:Exception => error(e);
      }
}
```
Scala is a full object ( Java is not). x : {def close()} means any class that has this method as signature. 
The best point here is that scala rather than other dynamic language do compilation check, the worst is the performance.

<h2>II- GENERIC CLOSE</h2>

We can use var args close version :
```scala
def close(x : {def close()}*){
  x.foreach (close)    
}
```
and
```scala
close(resultSet)
close(preparedStatement)
close(conn)
```
will be :
```scala
close(resultSet,preparedStatement,conn)
```

<h2>III- OPEN CLASS</h2>

What if we need to add close method to theses class even final ones? 
Let's open class by creation rich one:
```scala
class RichConnection (c:Connection){
  def <>(){
    close(c)
  }
}
class RichPreparedStatement(p :PreparedStatement){
  def <>(){
    close(p)
  }
}
class RichResultSet(r:ResultSet){
  def  <>(){
    close(r)
  }
}
```
and introduce implicit conversions :
```scala
implicit def toRichConnection(c:Connection):RichConnection= new RichConnection(c)
implicit def toRichPreparedStatement(p:PreparedStatement):RichPreparedStatement= new RichPreparedStatement(p)
implicit def toRichResultSet(r:ResultSet):RichResultSet= new RichResultSet (r)
```
and as the result the call will be :
```scala
finally {
      resultSet.<>
      preparedStatement.<>
      conn.<>
    }
```
<h2>IV- USING TYPES</h2>

define type :
```scala
type AutoCloseable={def close()}
```
and close definition will be 
```scala
def close(x : AutoCloseable){
     try {
           if (x != null) {
               x.close();
           }
       } catch  {
          case  e:Exception => error(e);
       }
  }
 def close(x : AutoCloseable*){
   x.foreach (close)    
 }
```
<h2>V-CHAINED TRY-WITH</h2>
we introduce a curring tryWith method as
```scala
def tryWith[A<%AutoCloseable](ac :A)(f: A=> Unit)
{
  try {
    f
  } finally {
   close(ac)
  }
}
```
tryWith enable us to use doQuery as :
```scala
def doQuery( q:String)   {
   tryWith (getConnection()) { conn =>
     tryWith ( conn.prepareStatement(q)) {ps =>
       tryWith (ps.executeQuery()){rs =>
         }
     }     
  } 
}
```