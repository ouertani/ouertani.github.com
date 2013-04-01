---
layout: post
title: "Jaxws Header : Part 2 : Server Side"
date: 2012-06-28 03:19
comments: true
categories: [Scala, JEE, JAX-WS]
keywords: Scala, JEE, JAX-WS
description: JAX-Ws header on the server side
---
<h2>Purpose</h2>

Manipulating JAXWS header on the server Side like reading WSS username token, logging saop message and publish a specific header.

<h2>Introduction</h2>

On Telecom IT environment and specially middelware solution web service communications are heavy used between solutions. This tutorial aims to introduce using handler on server side by publishing specific header, reading WSS UserToken or logging the soap message on console.
<!-- more -->
This tutorial is Scala based, Java version can be easy translated.

<h2>Parse undeclared custom header</h2>

Let's consider We need to read WSS UserToken non published in our WSDL :
```xml
<soapenv:Header>
      <wsse:Security  xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd" xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd">
         <wsse:UsernameToken wsu:Id="UsernameToken-1">
            <wsse:Username>login</wsse:Username>
            <wsse:Password Type="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-username-token-profile-1.0#PasswordText">XXXX</wsse:Password>        
         </wsse:UsernameToken>
      </wsse:Security>
 </soapenv:Header>
```
To parse this header we need to extend a SoapHandler, ovveride handle message class on put header and password on context, below a constant and full class
```scala
object AuthenticationHandlerConstants {
 
  val REQUEST_USERID: String = "authn_userid";
  val REQUEST_PASSWORD: String = "authn_password";
  val AUTHN_PREFIX: String = "wsse";
  val AUTHN_LNAME: String = "Security";
  val AUTHN_URI: String = "http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd";
  val AUTHN_STAUTS: String = "authnStatus";
 
  val LOGIN: String = "login"
  val PWD: String = "passwd"
}
```
```scala
import java.io.PrintStream;
import java.util.HashSet;
import java.util.Iterator;
import java.util.Set;
import javax.xml.namespace.QName;
import javax.xml.soap.SOAPElement;
 
import javax.xml.soap.SOAPHeader;
import javax.xml.ws.handler.MessageContext;
import javax.xml.ws.handler.soap.SOAPHandler;
import javax.xml.ws.handler.soap.SOAPMessageContext;
import org.apache.log4j.Logger;
 
/**
 * @author Slim Ouertani
 */
class SecuritySOAPHandler extends SOAPHandler[SOAPMessageContext] {
 
  private val L = Logger.getLogger(classOf[SecuritySOAPHandler]);
 
  override def getHeaders(): Set[QName] = {
    val securityHeader = new QName("http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd", "Security", "wsse");
    val headers = new HashSet[QName]();
    headers.add(securityHeader);
    return headers;
  }
 
  override def handleMessage(soapMessageContext: SOAPMessageContext): Boolean = {
 
    try {
      val outMessageIndicator = soapMessageContext.get(MessageContext.MESSAGE_OUTBOUND_PROPERTY).asInstanceOf[Boolean];
      if (!outMessageIndicator) {
        val envelope = soapMessageContext.getMessage().getSOAPPart().getEnvelope()
        val header = envelope.getHeader()
 
        if (header == null) {
          L.warn("No headers found in the input SOAP request")
        } else {
          processSOAPHeader(header) match {
            case Some(_processSOAPHeader) => {
              soapMessageContext.put(AuthenticationHandlerConstants.AUTHN_STAUTS, java.lang.Boolean.TRUE);
              soapMessageContext.put(AuthenticationHandlerConstants.LOGIN, _processSOAPHeader._1);
              soapMessageContext.put(AuthenticationHandlerConstants.PWD, _processSOAPHeader._2);
            }
            case None => {
              soapMessageContext.put(AuthenticationHandlerConstants.AUTHN_STAUTS, java.lang.Boolean.FALSE);
              soapMessageContext.put(AuthenticationHandlerConstants.LOGIN, "");
              soapMessageContext.put(AuthenticationHandlerConstants.PWD, "");
            }
          }
        }
      }
    } catch {
      case ex: Exception => L.error(ex.getMessage(), ex);
    }
    soapMessageContext.setScope(AuthenticationHandlerConstants.AUTHN_STAUTS, MessageContext.Scope.APPLICATION);
    soapMessageContext.setScope(AuthenticationHandlerConstants.LOGIN, MessageContext.Scope.APPLICATION);
    soapMessageContext.setScope(AuthenticationHandlerConstants.PWD, MessageContext.Scope.APPLICATION);
    return true;
  }
 
  private def processSOAPHeaderInfo(e: SOAPElement): (String,String) = {
    var _id: String = null
    var _password: String = null
 
    val childElements = e.getChildElements(new QName(AuthenticationHandlerConstants.AUTHN_URI, "UsernameToken"));
    while (childElements.hasNext()) {
      val usernameToken = childElements.next();
      // loop through child elements
 
      usernameToken match {
 
        case child: SOAPElement => {
 
          val childElements1 = child.getChildElements(new QName(AuthenticationHandlerConstants.AUTHN_URI, "Username"));
          val childElements2 = child.getChildElements(new QName(AuthenticationHandlerConstants.AUTHN_URI, "Password"));
 
          while (childElements1.hasNext()) {
            val next = childElements1.next();
            next match {
              case l: SOAPElement => {
                val value = l.getValue();
 
                _id = value;
              }
            }
          }
          while (childElements2.hasNext()) {
            val next = childElements2.next();
            next match {
              case l: SOAPElement => {
                val value = l.getValue()
 
                _password = value;
              }
            }
          }
        }
      }
    }
    return (_id, _password);
  }
 
  private def processSOAPHeader(sh: SOAPHeader): Option[(String,String)] = {
    var authenticated: Option[(String,String)] = None
 
    val childElems = sh.getChildElements(new QName(
      AuthenticationHandlerConstants.AUTHN_URI,
      AuthenticationHandlerConstants.AUTHN_LNAME));
 
    // iterate through child elements
    while (childElems.hasNext()) {
 
      val child = childElems.next().asInstanceOf[SOAPElement]
      authenticated = Some(processSOAPHeaderInfo(child))
    }
    return authenticated
  }
 
  override def handleFault(soapMessageContext: SOAPMessageContext): Boolean = false
}
```
Unlike client side we need to add an xml file handler.xml to enable the previous handler
<h3>on web service declaration: </h3>
<ol>
	<li>Chain this handler : @HandlerChain(file = "handler.xml")</li>
	<li>Inject WebServiceContext : @Resource private var wsContext: WebServiceContext = _</li>
	<li>Retrieve a messageContext : val msgContext = wsContext.getMessageContext()</li>
	<li>Check stored valued : val authnStatus = msgContext.get(AuthenticationHandlerConstants.AUTHN_STAUTS)</li>
</ol>
``` scala
@WebService(serviceName = "WsService", targetNamespace = "http://slim.ouertani.me)
@HandlerChain(file = "handler.xml")
@Stateless()
@Interceptors(Array(classOf[TracingInterceptor]))
class WsService extends LogHelper{
   
  @Resource private var wsContext: WebServiceContext = _
 
  def authenticate() {
 
    val msgContext = wsContext.getMessageContext()
    val authnStatus = msgContext.get(AuthenticationHandlerConstants.AUTHN_STAUTS)
    if (authnStatus == null || !authnStatus.asInstanceOf[Boolean]) {
       L.warn("header authentification not found");
      throw AuthenticationHeaderNotFound.toFaultResponse
    }
 
    try {
      val login = msgContext.get("login").asInstanceOf[String]
      val passwd = msgContext.get("passwd").asInstanceOf[String]
      L.info("user " + login + "pwd" + passwd + "try this service")
    } catch {
      case ex: Exception =>
        L.warn(" exception to parse header authentification", ex);
        throw AuthenticationHeaderNotFound.toFaultResponse
    }
  }
}
```

<h2>Adding custom header</h2>

<ol>Publishing a custom header is easier than using the previous handler :

<li>First lets add a custom header class</li> 
``` scala
import scala.reflect.BeanProperty
import javax.xml.bind.annotation.{XmlType , XmlAccessType, XmlElement, XmlAccessorType};
/**
 * @author Slim Ouertani
 */
@XmlAccessorType(XmlAccessType.FIELD)
@XmlType(name = "Header", propOrder = Array(
    "username",
    "password"
  ))
class UsernameToken {
   @BeanProperty  @XmlElement(required = true, nillable = false ) var username : String= ""
   @BeanProperty  @XmlElement(required = true, nillable = false ) var password : String= ""
}
```

<li>publish this header using @WebParam (name = "usernameToken",header = true)</li>
``` scala
def execute (@WebParam(name = "request" ) request : Request, @WebParam (name = "usernameToken",header = true) header) {
  ....
 }
```
<li>retrieving user and password field are the same as accessing class properties :</li>
``` scala
val username = ""+header.getUsername()
val password = ""+header.getPassword()
```
<li>note that the preceding header is declared on WSDL method and produce this output on soapUI interface </li></ol>
``` xml
<soapenv:Header>
      <ws:usernameToken>
         <username>login</username>
         <password>XXXXX</password>
      </ws:usernameToken>
</soapenv:Header>
```
<h2>Conclusion</h2>


Publishing specific header using JAX-WS is easier than using handler. Otherwise, handlers on server side are helpful for other purpose like logging I/O message.