---
layout: post
title: "Sending an Email using the JavaMail Session and Glassfish"
date: 2012-06-28 02:18
comments: true
categories: [JEE, MAIL, Glassfish]

keywords: JEE, MAIL, Glassfish, CDI
description: JEE Java Mail Api Configuration
---

<h2>Purpose</h2>

This tutorial is a supplement to the article of oracle published <a href="http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/javamail/javamail.html">here</a>.
After reading this later, I decided to share some tips about this EE environment JavaMail api configuration.
<h2>Introduction</h2>

Two years ago, we decided to move to JEE 6 for our enterprise solutions and take a lot of fun with new EJB 3.1 features and annotations. Glassfish has made our lives easier, it was very easy to declare variables using it's administration console.
Less complicated than using global resource environment for properties It is also the continuation to this <a href="http://www.jroller.com/articles/jee-6-environmental-enterprise">tricks</a> this tutorial delegate to Glassfish EE server managing mail session.
<!-- more -->
<h2>Prerequisites</h2>

<ol>Before starting this tutorial, you should:
<li>Fully understand of <a href="http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/javamail/javamail.html">http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/javamail/javamail.html</a></li>
<li>Have access to an SMTP server. You must know the host name, port number, and security settings for your SMTP server. Web mail providers may offer SMTP access, view your email account settings or help to find further information. Be aware that your user name is often your full email address and not just the name that comes before the @ symbol.</li>
<li>Have basic familiarity with Servlets and CDIs (helpful but not required)</li>
</ol>

<h2>Create a Java mail session resource :</h2>

<ol>
To create and JavaMail Session :
<li>start galssfish server</li>
<li>open admin console ( default localhost:4848)</li>
<li>Go to Resources -> JavaMail Sessions -> click new to add new javaMail resource</li>
<li>make sure the following field are filled in :</li>
</ol>
Jndi name : EMailME for example, will be used later on lookup resource :
```java
@Resource(lookup = "EMailME")
Mail host
```
<ol start="5">
<li>Default user</li>
<li>Default sender will be used later on mailSession.getProperty("mail.from")</li>
<li>For secure Mail add this advanced setting :

<ul>
<li>mail.smtp.password : email password</li>
<li>mail.smtp.port : email port</li>
<li>mail.smtp.auth : true </li>
</ul>
</li>
</ol>
This screenshot summarizes all these parameters :

![Mail EE](/images/mailee.jpg)

<h2>II - Using email Service</h2>


This tutorial is a supplement to oracle one. We will use CDI with the same example.

<ol>
	<li>MailSessionBean Will be a CDI bean with RequestScope</li>
	<li>Inject the full JavaMail Session as resource rather than create it</li>
</ol>
```java
@Named
@RequestScoped
public class EmailSessionBean {

    @Resource(lookup = "EMailME")
    private Session mailSession;

    public void sendEmail(String to, String subject, String body) {
        MimeMessage message = new MimeMessage(mailSession);
        try {

            message.setFrom(new InternetAddress(mailSession.getProperty("mail.from")));
            InternetAddress[] address = {new InternetAddress(to)};
            message.setRecipients(Message.RecipientType.TO, address);
            message.setSubject(subject);
            message.setSentDate(new Date());
            message.setText(body);

            Transport.send(message);
        } catch (MessagingException ex) {
            ex.printStackTrace();
        }
    }
}
```
Look short !

<ol start="3"><li>EmailServlet remind the same except using @Inject rather than @EJB</li></ol>
```java
@WebServlet(name = "EmailServlet", urlPatterns = {"/EmailServlet"})
public class EmailServlet extends HttpServlet {

    @Inject
    private EmailSessionBean emailBean;

    protected void processRequest(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
 .........
   }
 ......
}
```
<h2>Conclusion :</h2>

Let's EE Server manager MailSession for us, keep configuration on server and thanks to Glassfish administration console managing JavaMail Session became easier.
