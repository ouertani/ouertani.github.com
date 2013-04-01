---
layout: post
title: "XML-RPC using scala"
date: 2012-06-28 02:03
comments: true
categories: [Scala, Ucip, XML/RPC] 

keywords: SCALA, XML, UCIP, RPC
description: UCIP Scala driver using XML/RPC
---
<h2>Purpose</h2>

Calling remote procedure using XML-RPC and Scala.
<h2>Introduction

On Telecom IT environment and specially middelware solution, we will rarely do all the work but rather delegate some of business process to other tiers. Web service communications is heavy used between solutions. However, many IT node continues to support older protocols like XML-RPC. Ericsson Intelligent Network (IN) and its subsystem uses AIR Integration Protocol user communication(UCIP) based on a variant of XM-RPC.

<h2>Why UCIP ?</h2>

UCIP is intended for user self services such as Adjustments, Account Refill, and Account Inquiries and to extract account details. UCIP is an IP-based protocol used for integration towards the AIR server from the external application.
<h2>Why Scala ?

UCIP is an XML over HTTP based protocol, which makes it easy to integrate with a central integration point within a network. In addition, Scala can be used to easily create, parse, and process XML documents. XML data can be represented in Scala either by using a generic data representation, or with a data-specific data representation.
<h2>Scala & XML-RPC</h2>

Rather than using an precompiled version of XML-RPC library, we will do our home solution using httpclient library from apache :
```xml
 <dependency>
 <groupId>org.apache.httpcomponents</groupId>
 <artifactId>httpclient</artifactId>
 <version>4.1</version>
</dependency>
```
<h3>UPDATEBALANCEANDDATE</h3>


The message UpdateBalanceAndDate is used to adjust balances and expiry dates on the main account and the dedicated accounts. The sample I chooses here is only to update main balance :

Let's start with some helpful class :

1- InParameters class : contains In specific configuration like ip and port
```scala
case class InParameters(ip:String,port:Int,user:String,pwd:String,agent:String,url:String)
```
2- XMLRPCVo abstract class : groups all XML request message like the transaction generated Id and date
```scala
object XmlRpcVo {
 val f = new SimpleDateFormat("yyyyMMdd'T'HH:mm:ssZ");
}
abstract class XmlRpcVo {
 def originTransactionID = System currentTimeMillis
 def dateTime:String= XmlRpcVo.f.format(new Date())
}
```
3- UpdateBalanceAndDateParameter class with extend our XMLRPCVo and contains all data needed by Update main balance operation like subscriber number
```scala
case class UpdateBalanceAndDateParameter(subscriberNumber:String,
 originNodeType:String,
 originHostName:String,
 transactionCurrency:String,
 amount: Int )
extends XmlRpcVo
```
4-UCIP suggests to add custom header on XML-RPC message. XmlRpcHttpClient class dialogue with IN nodes using generic execute method 
```scala
class XmlRpcHttpClient(inp :InParameters) extends LogHelper{
 
  private val httpclient:DefaultHttpClient = client 
 
  private [this] def client() =
  {
    val httpclient = new DefaultHttpClient();
       
    httpclient.getCredentialsProvider().setCredentials(
      new AuthScope(inp.ip, inp.port),
      new UsernamePasswordCredentials(inp.user, inp.pwd)) 
 
 
    httpclient.getParams().setParameter(CoreProtocolPNames.USER_AGENT, inp.agent) 
    httpclient
  }
  private [this] def poster(elem:Elem) ={
    val httppost = new HttpPost(inp.url)
    httppost.addHeader("Content-Type", "text/xml")
    httppost.addHeader("Content-Disposition", "form-data; name=\"fname\"; filename=\"request.xml\"")
 
    val comment = new StringBody(""+elem)
 
    val reqEntity = new MultipartEntity()
    reqEntity.addPart("fname", comment)
    httppost.setEntity(reqEntity)
    httppost
  }
 
  def execute(requestVo:scala.xml.Elem):HttpResponse = {
    try     { client.execute(poster(requestVo))}
    catch   { case e =>  throw new connectionExecption(e) }
    finally {  client.getConnectionManager().shutdown() }
  }
}
```
XML-RPC is a Remote Procedure Calling protocol and the body of an XML-RPC request is formatted using XML. A procedure executes on the AIR server and the value it returns is also formatted in XML. The interface to the AIR server interface uses XML-RPC over HTTP. Sending an XML to update the main balance will be using this class

5- UpdateBalanceAndDate class : generates an UpdateBalanceAndDate XML request message
```scala
class UpdateBalanceAndDate (ubdp :UpdateBalanceAndDateParameter)  {
    def request=
      <methodCall>
        <methodName>UpdateBalanceAndDate</methodName>
        <params>
          <param>
            <value>
              <struct>
                <member>
                  <name>originNodeType</name>
                  <value>
                    <string>{ubdp.originNodeType}</string>
                  </value>
                </member>
                <member>
                  <name>originHostName</name>
                  <value>
                    <string>{ubdp.originHostName}</string>
                  </value>
                </member>
                <member>
                  <name>originTransactionID</name>
                  <value>
                    <string>{ubdp.originTransactionID}</string>
                  </value>
                </member>
                <member>
                  <name>originTimeStamp</name>
                  <value>
                    <dateTime.iso8601>{ubdp.dateTime}</dateTime.iso8601>
                  </value>
                </member>
                <member>
                  <name>subscriberNumber</name>
                  <value>
                    <string>{ubdp.subscriberNumber}</string>
                  </value>
                </member>
                <member>
                  <name>transactionCurrency</name>
                  <value>
                    <string>{ubdp.transactionCurrency}</string>
                  </value>
                </member>
                <member>
                  <name>adjustmentAmountRelative</name>
                  <value>
                    <string>{ubdp.amount}</string>
                  </value>
                </member>                
              </struct>
            </value>
          </param>
        </params>
      </methodCall>
  }
```
6- UpdateBalanceAndDate companion object is used to parse response and validate the transaction
```scala
object UpdateBalanceAndDate {
  
  @throws (classOf[ClientProtocolException])
  @throws (classOf[IOException])
  def  getResponse(response:HttpResponse ):Int=  {
     
    try {
      val entity = response.getEntity();
      if (entity != null) {      
        val s = EntityUtils.toString(entity)
        val elem =  XML.loadString(s)      
     
        val tuple=   for {
          x <- (elem \\ "member")
          name = (x \ "name" ) .text
          code = ( x  \ "value"  \ "i4" ).text  if(name == "responseCode")
            } yield (code)
         
          if(tuple.size == 1)  {
            return tuple(0).toInt 
          }
          return RESPONSE_PARSE_EXCEPTION
        }
        else { return NULL_RESPONSE  }
      }
      catch {
        case _ =>  return  UNEXCEPTED_ERROR_WHILE_PARSE_RESPONSE         
      }
    }
  }
```
7- UpdateBalanceAndDateController controlleur class orchestrate requests and utilities class
```scala
class UpdateBalanceAndDateController {
 
  def update(inp: InParameters, ubdp : UpdateBalanceAndDateParameter ):Int={ 
    try {
      val c = new XmlRpcHttpClient(inp)
      val ubd = new  UpdateBalanceAndDate (ubdp)
      val rep=  c execute(ubd.request)
      return UpdateBalanceAndDate.getResponse(rep ) 
    }catch {
      case e:connectionExecption => return AIR_EXCEPTION  
      case e =>  return UNEXPECTED_CONNECTION__EXCEPTION 
    }
  }
}
```
<h2>Conclusion</h2>

Create XML message and parse it using scala is like a baby toy and makes using UCIP protocol easier and simpler. This sample can be generalized to the other UCIP messages like GetBalanceAndDate or GetAccountDetails and any other XML-RPC communication.