---
layout: post
title: "WCF Data Services and maxReceivedMessageSize"
date: 2011-05-09 15:49:14 +0000
categories: ["Programming"]
original_url: https://nivlam.wordpress.com/2011/05/09/wcf-data-services-and-maxreceivedmessagesize/
---

We have a WCF Data Service (Astoria) that uses most of the default configuration settings. Every once in a while, a client would receive the following exception:

{% raw %}
```text
System.Data.Services.Client.DataServiceRequestException: An error occurred while processing this request. ---> System.Data.Services.Client.DataServiceClientException: BadRequest
```
{% endraw %}

This exception only occurs when the client sends a large message. The default message size in WCF is 65536. Since we're using a WCF Data Service and not a normal WCF Service, adding a WCF Data Service (.svc) to the project does not auto-generate the service/binding information in the config. To increase the message size, I needed to manually enter the service/binding information:

{% raw %}
```xml
<system.serviceModel>
  <services>
    <service name="MyNamespace.MyService">
      <endpoint bindingConfiguration="msgSize" address="" binding="webHttpBinding" contract="System.Data.Services.IRequestHandler" />
    </service>
  </services>
  <bindings>
    <webHttpBinding>
      <!-- 2097152 = 2 MB-->
      <binding name="msgSize" maxReceivedMessageSize="2097152" maxBufferSize="2097152" />
    </webHttpBinding>
  </bindings>
</system.serviceModel>
```
{% endraw %}

The service name is the class (.svc) that inherits from `System.Data.Services.DataService<T>`.
