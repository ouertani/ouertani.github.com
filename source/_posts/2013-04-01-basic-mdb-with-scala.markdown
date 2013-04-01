---
layout: post
title: "Basic MDB with Scala"
date: 2011-09-11 03:41
comments: true
categories: [JEE, MDB, SCALA]
keywords: JEE, SCALA, MDB
description: MDB JEE Messaging using scala
---
While I am Akka fun, Typesafe follower, in this post I will continue presenting scala on JEE 6 environment and introduce basic message driven bean (MDB ) using scala language.

As an example let's take a simple one MDB that will consome operation command like add and substract and my be asked to replay by the result
<!-- more -->
<h2>I- Exchange API</h2>
```scala
sealed trait TOperation
case class Add(x:Int) extends  TOperation
case class Sub(x:Int) extends TOperation
case class Sum extends TOperation
case class Count(x:Int) extends TOperation
```
<h2>II- The listener</h2>

The object
``` scala
object MDbListener {
  var count = 0
  val L = Logger.getLogger(classOf[MDbListener])
}
```
The class
``` scala
@MessageDriven(
  messageListenerInterface=classOf[MessageListener],
  mappedName = "jms/toMdb",
  activationConfig = Array(
    new ActivationConfigProperty(propertyName = "acknowledgeMode", propertyValue = "Auto-acknowledge"),
    new ActivationConfigProperty(propertyName = "destinationType", propertyValue = "javax.jms.Queue")))
class MDbListener extends MessageListener {
 
  import MDbListener._
 
  override def onMessage(message: Message) {
    L.debug("Message Received")
 
    message match {
      case o: ObjectMessage => {
          try {
            val obj = o.getObject()
            obj match {
              case action: TOperation => handle(action)
              case _ => L.warn("Unkownen msg")
            }
          } catch {
            case ex: Exception => L.error(">>> Exception", ex)
          }
 
        }
      case _ => L.warn("unknown msg")
    }
  }
 
  def handle(op: TOperation ) = op match {/* TODO */
  }
}
```
Nothing especial here but be careful to not forget adding messageListenerInterface=classOf[MessageListener] on MessageDriven annotation this is because ScalaObject super class and that's all for the listener.

<h2>III- The sender</h2>

The handle method will do mathematical operation and can be requested to send the sum to another JMS Queue.
``` scala
def handle(op: TOperation ) = op match {
    case   Add(x) => count +=x
    case   Sub(x) => count -=x
    case   s:Sum =>  sendJMSMessageToTt( Count (count))
    case   c:Count => L.warn ("Wrong message destination!")
  }
```
Very easy with pattern matching!

The sendJmsMessageToIt method can be done like the following

``` scala
@Resource(mappedName = "jms/Rep")
 var  queue :Queue=_
 @Resource(mappedName = "jms/RepFactory")
 var  factory:ConnectionFactory=_
 
 private [ws ] def sendJMSMessageToTt( messageData:Count)  {
   def  createJMSMessageForjmsTt( session:Session,  messageData:Count):Message={
     val om = session.createObjectMessage()
     om.setObject(messageData)
     om
   }
 
   def tryWith[A<%{ def close()}](ac :A)(f: A=> Unit)
   {
     try {f} finally { ac.close }
   }
 
   tryWith(factory.createConnection()){
     connection => tryWith(connection.createSession(false, Session.AUTO_ACKNOWLEDGE)) {
        session =>  tryWith( session createProducer queue ){
                messageProducer => messageProducer.send(createJMSMessageForjmsTt(session, messageData))
       }
     }
   }
 }
```
Here we used an old trick to manage resources : <a href="http://www.jroller.com/ouertani/entry/scala_try_with_resources">http://www.jroller.com/ouertani/entry/scala_try_with_resources</a>