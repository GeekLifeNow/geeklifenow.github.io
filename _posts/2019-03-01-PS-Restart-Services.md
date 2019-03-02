---
layout: post
title: Restart (Remote) Services with PowerShell
subtitle: Easy script to remotely restart service(s).
tags: [PowerShell, automation, remote, services]
---

Have you ever had the need to quickly restart services on a remote Windows server and gotten tired of either RDPing to it or connecting a remote mmc in the GUI to restart services? Well, we have a few processes that sometime require the restart of services when they get hung up. Until the dev team finds a way to polish off their services to either report out and/or restart automatically, I'll just run a script to retstart the services in question!

There's not much to it. No fancy logging or error catching, but it does exactly what it needs to do. At the core, it's a matter of getting the remote service, adding it to a variable and then restarting the service using the Restart-Service cmdlet. Here's a few lines from the script:

~~~
$Service = Get-Service -ComputerName RemoteServer -Name "MyFlakyService"
Restart-Service -InputObject $Service -Verbose
~~~

_How do I know the service is back up?_

Great question! Head over to my rather humble [GitHub PowerShell Repository](https://github.com/GeekLifeNow/PowerShell-Automation){:target="_blank"} to get it for yourself and see how to check service status!

Restart **all** the services!

<p align="center">
  <img src="/img/restart-all-the-services.jpg">
</p>
