---
layout: post
title: "Setting up Shibboleth SP on Windows Server 2012 with IIS"
tags:
  - IT
  - Windows
  - 
date: 2015-05-01
---

I was recently tasked with setting up Shibboleth SP on one of our Windows Server 2012 machines in order to facilitate SSO with one of our clients. This is a quick rundown of the steps I ended up needing to take in order to get this functioning properly.

Step number one was to setup Shibboleth on our Windows Server. I downloaded the package [here](http://shibboleth.net/downloads/service-provider/latest/win64/). At the time the file I grabbed was: shibboleth-sp-2.5.4-win64.msi but version numbers may change over time. I installed it and double checked all of the steps listed under manual installation [here](https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPWindowsIIS7Installer) were implemented properly.

I then started the configuration process in the default installation directory.

```
C:\opt\shibboleth-sp\etc\shibboleth\shibboleth2.xml 
```

This file can be a little bit dense at times but I'll try and cover the major parts but I found the abundant amount of commented out configuration to be overwhealming. I saved a copy and started building from scratch using the pieces I needed from the example.


This part should be placed in the first section after the initial SPConfig block and it's mapping Site ids to hostnames. This allows the Request Mapper to function below in the config file. In this example example.com is id #1 in IIS (Check this in the advanced settings of your site in IIS manager)

    <InProcess logger="native.logger">  
      <ISAPI normalizeRequest="true" safeHeaderNames="true">  
        <Site id="1" name="example.com"/>  
      </ISAPI>  
    </InProcess>  

This section tells Shibboleth which sections of the website should be protected behind the SSO. In our case example.com/secure will be protected but example.com will not. The name parameter is the subfolder that will be protected and requireSession states that they must be authenticated with shibboleth to see the contents.

    <RequestMapper type="Native">  
      <RequestMap applicationId="default">  
        <Host name="example.com"> 
          <Path name="secure" authType="shibboleth" requireSession="true"/>  
        </Host>  
      </RequestMap>  
    </RequestMapper>  

The entityID created here is what your site calls itself when communicating with the IDP.

    <ApplicationDefaults entityID="https://example.com/shibboleth">

The session section can be quite long so I've added comments below to explain each section and what it is doing.

    <Sessions lifetime="28800" timeout="3600" checkAddress="false" relayState="ss:mem" handlerSSL="true">

      <!--This section states the location of your IDP and what protocols it should support-->
      <SSO entityID="https://idp.client.com">
        SAML2 SAML1
      </SSO>

      <!--This section states what protocols to support in regards to logging out-->
      <Logout>SAML2 Local</Logout>

      <!--These handlers allow you to interact with Shibboleth itself by visiting a url (In our case example.com/Shibboleth.sso/Location)-->

      <!--This handler outputs an xml metadata file that you can use to find information about your shibboleth install and information you will be required to provide the IDP-->
      <Handler type="MetadataGenerator" Location="/Metadata" signing="false"/>

      <!-- This handler outputs information about your current logged in session -->
      <Handler type="Session" Location="/Session" showAttributeValues="true"/>

      <!-- JSON feed of discovery information. -->
      <Handler type="DiscoveryFeed" Location="/DiscoFeed"/>

    </Sessions>

This block simply states what will be shown on your error page if something goes wrong with Shibboleth.

    <Errors supportContact="root@localhost" logoLocation="/shibboleth-sp/logo.jpg" styleSheet="/shibboleth-sp/main.css"/>

This is the important part, this is the metadata provided by your IDP with "instructions" on how to connect. In this example our client gave us an xml file that I loaded locally but there are options that allow you to download a dynamic metadata file

    <MetadataProvider type="XML" file="partner-metadata.xml"/>

    <!-- Attribute and trust options you shouldn't need to change. -->
    <AttributeExtractor type="XML" validate="true" path="attribute-map.xml"/>
    <AttributeResolver type="Query" subjectMatch="true"/>
    <AttributeFilter type="XML" validate="true" path="attribute-policy.xml"/>

That was about it for configuring the shibboleth2.xml file. There are several other sections that have values that should not be removed but I didn't break them out. I've posted a complete "basic" configuration file [here](https://mruszczyk.github.io/assets/files/shibboleth2.xml) that includes those sections.

At this point I had thought I was done with configuring Shibboleth but I ended up needing to do some work in attribute-map.xml that allows metadata provided by the IDP to be read by our developer but that should not be relevent to all cases. If you have any questions or corrections feel free to shoot me an email.