---
layout: post
title: "JAX-WS Header : Part 1 the Client Side"
date: 2012-06-27 03:32
comments: true
categories: [JEE, SCALA, JAX-WS]
keywords: Scala, JEE, JAX-WS
description: JAX-Ws header on the client side
---
<h2>Purpose</h2>

Manipulating JAXWS header on the client Side like adding WSS username token or logging saop message.
<h2>Introduction</h2>

On Telecom IT environment and specially middelware solution, we will rarely do all the work but rather delegate some of business process to other tiers. Web service communications is heavy used between solutions. This tutorial aims to introduce using handler on client side by adding WSS UserToken or logging the soap message on console.
<!-- more -->
<h2>Adding undeclared custom header</h2>

Some Ws client needs to add a custom header which are not declared on WSDL. Adding WSS Username Token is like adding this XML snippet on header element:
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
Unlike <a href="http://docs.oracle.com/javaee/5/api/javax/xml/ws/handler/LogicalHandler.html">LogicalHandler</a> SOAPHandler have access to the entire Soap Message. Let's create an WSSUsernameTokenSecurityHandler Local Stateless Bean extending SOAPHandler to produce the previous header.
Below the WSSUsernameTokenSecurityHandler class
``` java WSSUsernameTokenSecurityHandler.java
package me.slim.ouertani;
 
import java.util.Set;
import java.util.TreeSet;
import javax.annotation.Resource;
import javax.ejb.LocalBean;
import javax.ejb.Stateless;
import javax.xml.namespace.QName;
import javax.xml.soap.*;
import javax.xml.ws.handler.MessageContext;
import javax.xml.ws.handler.soap.SOAPHandler;
import javax.xml.ws.handler.soap.SOAPMessageContext;
 
@Stateless
@LocalBean
public class WSSUsernameTokenSecurityHandler implements SOAPHandler<SOAPMessageContext> {
 
    @Resource(lookup = "login")
    private String login;
    @Resource(lookup = "pwd")
    private String pwd;
 
    public WSSUsernameTokenSecurityHandler() {
    }
 
    @Override
    public boolean handleMessage(SOAPMessageContext context) {
 
        Boolean outboundProperty =
                (Boolean) context.get(MessageContext.MESSAGE_OUTBOUND_PROPERTY);
        if (outboundProperty.booleanValue()) {
 
            try {
                SOAPEnvelope envelope = context.getMessage().getSOAPPart().getEnvelope();
                SOAPFactory factory = SOAPFactory.newInstance();
                String prefix = "wsse";
                String uri = "http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd";
                SOAPElement securityElem =
                        factory.createElement("Security", prefix, uri);
                SOAPElement tokenElem =
                        factory.createElement("UsernameToken", prefix, uri);
                tokenElem.addAttribute(QName.valueOf("wsu:Id"), "UsernameToken-2");
                tokenElem.addAttribute(QName.valueOf("xmlns:wsu"), "http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd");
                SOAPElement userElem =
                        factory.createElement("Username", prefix, uri);
                userElem.addTextNode(login);
                SOAPElement pwdElem =
                        factory.createElement("Password", prefix, uri);
                pwdElem.addTextNode(pwd);
                pwdElem.addAttribute(QName.valueOf("Type"), "http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-username-token-profile-1.0#PasswordText");
                tokenElem.addChildElement(userElem);
                tokenElem.addChildElement(pwdElem);
                securityElem.addChildElement(tokenElem);
                SOAPHeader header = envelope.addHeader();
                header.addChildElement(securityElem);
 
            } catch (Exception e) {
                e.printStackTrace();
            }
        } else {
            // inbound
        }
        return true;
    }
 
    @Override
    public Set<QName> getHeaders() {
        return new TreeSet();
    }
 
    @Override
    public boolean handleFault(SOAPMessageContext context) {
        return false;
    }
 
    @Override
    public void close(MessageContext context) {
        //
    }
}
```
Next, declare the StatelessBean and inject both Webservice via @WebServiceRef and WSSUsernameTokenSecurityHandler via @EJB. A callback init method will add a HandlerResolver to the service. Below the full implementation
``` java WebServiceClientBean.java
@Stateless
@LocalBean
public class WebServiceClientBean {
 
    @WebServiceRef()
    private WsService service;
    @EJB
    private WSSUsernameTokenSecurityHandler wSSUsernameTokenSecurityHandler;
 
    @PostConstruct
    private void init() {
        service.setHandlerResolver( new HandlerResolver() {
 
            @Override
            public List<Handler> getHandlerChain(PortInfo portInfo) {
                List<Handler> handlerList = new ArrayList<Handler>();
                handlerList.add(wSSUsernameTokenSecurityHandler);
                return handlerList;
            }
        });
    }
 
  public WsResponse getService(WsRequest wsRequest) {      
        WsPort port = service.getPort();
        return port.invoqueService(wsRequest);
    }
 
}
```
<h2>Logging SOAP messages</h2>

Logging exchange xml message can be also using handlers, below the scala snippet handler used to log both in and out messages:
``` scala RequestResponsePrinter.scala
class RequestResponsePrinter  extends SOAPHandler[SOAPMessageContext] with LogHelper {
 
override def getHeaders(): Set[QName] = {
    return new TreeSet();
  }
 
  override def handleMessage(context: SOAPMessageContext): Boolean = {
    val sb = new ToStringBuilder(this).append("Call operation", "handleMessage");
 
    L.debug(sb);
    val outboundProperty = context.get(MessageContext.MESSAGE_OUTBOUND_PROPERTY).asInstanceOf[Boolean]
  
 
      try {
            val msg = context.getMessage()
            val out = new ByteArrayOutputStream();
            msg.writeTo(out);
            val strMsg = new String(out.toByteArray());
            L.debug("outbound  : "+outboundProperty.booleanValue()+" [" + strMsg + "]");
 
      } catch {
        case e: Exception =>
          L.error(sb.append("EXCEPTION", e.getMessage()), e);
      }
   
    return true;
  }
 
  override def handleFault(context: SOAPMessageContext): Boolean = {
    return false;
  }
 
  override def close(context: MessageContext) {
    //
  }
}
```

``` scala scala version of Handler resolver HandlerResolver.scala
new HandlerResolver() {
 
      override def getHandlerChain(portInfo: PortInfo): java.util.List[javax.xml.ws.handler.Handler[_ <: javax.xml.ws.handler.MessageContext]] = {
        import scala.collection.mutable.ArrayBuffer
        val handlerList = ArrayBuffer[Handler[_ <: javax.xml.ws.handler.MessageContext]]()       
        handlerList += new RequestResponsePrinter()
        return handlerList
 
      }
    };
```
<h2>Conclusion</h2>

We have heavily used handler on client side. On the server side things can change a bit, to be continued ...
