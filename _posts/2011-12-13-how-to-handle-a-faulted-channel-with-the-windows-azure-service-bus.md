---
author: admin
comments: true
date: 2011-12-13 04:52:51+00:00
layout: post
slug: how-to-handle-a-faulted-channel-with-the-windows-azure-service-bus
title: How to Handle a Faulted Channel with the Windows Azure Service Bus
wordpress_id: 1736
categories:
- Service Bus
- WCF
- Windows Azure
---

Recently I wrote a WCF service that fronted an on-premises SQL Server database for an MVC application running in a [Window Azure Web Role](http://www.windowsazure.com/en-us/home/tour/compute/). There are a number of ways I could have approached this scenario; I decided to use the [netTcpRelayBinding](http://msdn.microsoft.com/en-us/library/windowsazure/microsoft.servicebus.nettcprelaybinding.aspx) via the [Windows Azure Service Bus](http://www.windowsazure.com/en-us/home/tour/service-bus/) for the following reasons:

 

  
  * I needed optimal performance (i.e. TCP over HTTP). 
   
  * I wanted to reuse existing connections (rather than opening many connections). 
   
  * I didn’t want to open up ports in my firewall (inbound or outbound). 
 

Based on these requirements, the Service Bus is almost a no brainer. The Service Bus provides both messaging and connectivity capabilities, the latter of which provide a nice way to build loosely coupled applications in hybrid scenarios (i.e. cloud + on-premises).

 

I dove right in and wrote the following code:

 
    
    <span style="color: blue">public static </span><span style="color: #2b91af">IEnumerable</span><<span style="color: #2b91af">Customer</span>> GetCustomers()
    {
        <span style="color: #2b91af">Uri </span>serviceUri = <span style="color: #2b91af">ServiceBusEnvironment</span>.CreateServiceUri(<span style="color: #a31515">"sb"</span>, 
            <span style="color: #a31515">"MYSERVICENAMESPACE"</span>, <span style="color: #a31515">"Customer"</span>);
        <span style="color: blue">var </span>customersChannelFactory = <span style="color: blue">new </span><span style="color: #2b91af">ChannelFactory</span><<span style="color: #2b91af">ICustomerChannel</span>>(
            <span style="color: #a31515">"RelayEndpoint"</span>, <span style="color: blue">new </span><span style="color: #2b91af">EndpointAddress</span>(serviceUri));
        <span style="color: blue">var </span>customersChannel = customersChannelFactory.CreateChannel();
        customersChannel.Open();
    
        <span style="color: blue">return </span>customersChannel.GetCustomers();
    }






(I put much of the binding, client, and endpoint behaviors in the Web.Config.)





This code works but has problems. The channel to the Service Bus will open every time the service is called, resulting in suboptimal performance. To make this more efficient I decided to make the ChannelFactory and Channel variables statics so that I could reuse the channel if it had already been opened.




    
    <span style="color: blue">static </span><span style="color: #2b91af">ChannelFactory</span><<span style="color: #2b91af">ICustomerChannel</span>> customersChannelFactory;
    <span style="color: blue">static </span><span style="color: #2b91af">ICustomerChannel </span>customersChannel;
    
    <span style="color: blue">public static </span><span style="color: #2b91af">IEnumerable</span><<span style="color: #2b91af">Customer</span>> GetCustomers()
    {
        <span style="color: blue">if </span>(customersChannelFactory == <span style="color: blue">null </span>|| customersChannel == <span style="color: blue">null</span>)
        {
            <span style="color: #2b91af">Uri </span>serviceUri = <span style="color: #2b91af">ServiceBusEnvironment</span>.CreateServiceUri(<span style="color: #a31515">"sb"</span>, 
                <span style="color: #a31515">"MYSERVICENAMESPACE"</span>, <span style="color: #a31515">"Customer"</span>);
            customersChannelFactory = <span style="color: blue">new </span><span style="color: #2b91af">ChannelFactory</span><<span style="color: #2b91af">ICustomerChannel</span>>(
                <span style="color: #a31515">"RelayEndpoint"</span>, <span style="color: blue">new </span><span style="color: #2b91af">EndpointAddress</span>(serviceUri));
            customersChannel = customersChannelFactory.CreateChannel();
            customersChannel.Open();
        }
    
        <span style="color: blue">return </span>customersChannel.GetCustomers();
    }






This updated does a good job of improving the performance of the service call but has it’s own problems. When you create a Channel through the ChannelFactory your channel can enter a faulted state – with the Service Bus there are a number of ways that this can occur (in my testing it was because I often stopped/started the service host). When this happens the WCF communication static must be reset by recreating the client channel. 





Finding an efficient – and somewhat elegant – way of handling the faulted state proved to be a fun challenge. Fortunately, I received a lot of help from folks on the Service Bus team.





In the end I went with the following code:




    
    <span style="color: blue">static </span><span style="color: #2b91af">ChannelFactory</span><<span style="color: #2b91af">ICustomerChannel</span>> customersChannelFactory;
    <span style="color: blue">static </span><span style="color: #2b91af">ICustomerChannel </span>customersChannel;
    
    <span style="color: blue">public static </span><span style="color: #2b91af">IEnumerable</span><<span style="color: #2b91af">Customer</span>> GetCustomers()
    {
        <span style="color: #2b91af">List</span><<span style="color: #2b91af">Customer</span>> customers = <span style="color: blue">null</span>;
    
        <span style="color: blue">if </span>(customersChannelFactory == <span style="color: blue">null</span>)
        {
            <span style="color: #2b91af">Uri </span>serviceUri = <span style="color: #2b91af">ServiceBusEnvironment</span>.CreateServiceUri(<span style="color: #a31515">"sb"</span>, 
                <span style="color: #a31515">"MYSERVICENAMESPACE"</span>, <span style="color: #a31515">"Customer"</span>);
            customersChannelFactory = <span style="color: blue">new </span><span style="color: #2b91af">ChannelFactory</span><<span style="color: #2b91af">ICustomerChannel</span>>(
                <span style="color: #a31515">"RelayEndpoint"</span>, <span style="color: blue">new </span><span style="color: #2b91af">EndpointAddress</span>(serviceUri));
        }
    
        <span style="color: blue">int </span>tries = 0;
        <span style="color: blue">while </span>(tries++ < 3)
        {
            <span style="color: blue">try
            </span>{
                <span style="color: blue">if </span>(customersChannel == <span style="color: blue">null</span>)
                {
                    customersChannel = customersChannelFactory.CreateChannel();
                    customersChannel.Open();
                }
    
                <span style="color: blue">return </span>customersChannel.GetCustomers();
            }
            <span style="color: blue">catch </span>(<span style="color: #2b91af">CommunicationException</span>)
            {
                customersChannel.Abort();
                customersChannel = <span style="color: blue">null</span>;
            }
        }
    
        <span style="color: blue">return </span>customers;
    }






More verbose but much, much better.





The key is correctly handling the CommunicationException, aborting the channel, and setting the channel to null. Since we have retry logic via the while loop it will try again, determine that the channel is null, and re-open the channel.





There are certainly other ways to do this correctly. For example, you could decide to create a [Faulted event handler](http://msdn.microsoft.com/en-us/library/bb350915.aspx).





I hope this helps!
