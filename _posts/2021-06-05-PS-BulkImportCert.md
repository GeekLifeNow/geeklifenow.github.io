---
layout: post
title: 'Import PFX Certificate and Configure Binding on Remote IIS Servers'
subtitle: 'A way to import a pfx certificate and set the HTTPS binding on a list of remote servers.'
share-img: /img/posts/import_cert_0.png
tags: [PowerShell, PFX, IIS, SysAdmin, https, 443, bindings, web server, security, certificates]
---
Recently I was tasked with having to prepare a process to import a certificate and then set up an https (port 443) binding on about 100 web servers. I don’t know about you, but that seems like **way** too much clicking and manual steps! Luckily, I was fortunate enough to have the time to come up with a few scripts that will automate this process and make it so much easier to repeat in the future.

## PowerShell to the rescue…again!

As I have become more and more experienced with systems engineering, I tend to ask myself, “Can PowerShell make this easier for me?” It almost always starts a learning journey that only helps me to understand PowerShell even more. Thinking about this task, I broke it down into what I though were logical steps, which are basically the steps I would take in the GUI on a Windows Server:

1.	Import a .pfx certificate from a remote share
2.	Create a new binding for https in IIS
3.	Attach imported certificate to the https binding
4.	Check the ‘Require SSL’ box in IIS Manager

Most of these steps are straightforward clicks in the GUI, so why couldn’t all of this be done with a handful of PowerShell commands? Better yet, run against a list of multiple remote servers to do the same task?

I will post a link to my GitHub repo below that will have this full script for you to use in your environment, but for now I will break this up into the main parts to complete this task.
 
## Some quick preparations

A few variables are needed to get started. A working list of machines, and a network share where the .pfx cert file is located:

~~~
$Servers = Get-Content -Path C:\Scripts\Hosts\WebServers.txt
$CertFolder = '\\FileServer01\Shared\Certs’
~~~

If you will be running this against multiple machines, you will eventually want to be sure that you are working on machines that are online. I use a variant of _Test-Connection_ to ping each server at least once to make sure it is alive before moving on. If the server is online, I then copy the folder from the network share that contains the cert to the remote server’s _C:\Windows\Temp_ folder. This way the future command to import the certificate will not run into issues in copying from a network location. Here is that structure:

~~~
foreach ($Server in $Servers) {
    if (Test-Connection -ComputerName $Server -Quiet -Count 1) {
        Copy-Item -Path $CertFolder -Destination \\$Server\C$\Windows\Temp\CertAssets -Recurse -Force
        Write-Host "Assets successfully copied!"
     }
     else {
        Write-Host "$Server appears to be offline!"
      }
...
}
~~~
Notice I like to give feedback to the console with the _Write-Host_ lines as I run this. It would also be good to add to an existing log if you wrap your script with one. 

Now we know which servers are online and they have the assets copied locally. We are now ready to move forward.

## Import the certificate and create a new binding

This action is completed with the _Invoke-Command_ cmdlet which will run this action locally on each server. Depending on the level of security you wish in incorporate, you can do a few things with the password that is needed to import the PFX certificate. You can store it on the fly using _Get-Credential_ or you can just input the password in plain text inline with the command used for importing. I would suggest securing the password using _Get-Credential_ at a minimum. There are plenty of other ways to inject the password here, but that is out of the scope of this post. To harvest the password, you can use:

~~~
$MyPwd = Get-Credential -UserName 'Enter password below' -Message 'Enter password below'
~~~

<p align="center">
    <img src="/img/posts/import_cert_1.png">
</p>

This will store the password without having to have the password in plain test inside your script. We will pass this local variable into the remote command by utilizing the _$Using:_ component. Because _Invoke-Command_ and its accompanying script block run in a different scope (on the remote machine), it knows nothing of any local variables that are defined outside of the script block. This allows you to pass any local variables into the remote session and use accordingly.

We will be importing the certificate to the _Personal (My)_ certificate store of the machine account. The following is:

* The starting of the process on the remote server
* The import action using the provided password
* Create an https binding on port 443
* Adding the imported certificate to the https binding

*This still falls within the _foreach_ loop defined above:
~~~
...
Invoke-Command -ComputerName $Server -ScriptBlock {
    $SiteName = "Default Web Site"
    Import-PfxCertificate -Password $Using:MyPwd.Password -CertStoreLocation Cert:\LocalMachine\My -FilePath C:\Windows\Temp\CertAssets\GeekLifeNow.pfx
    Import-Module WebAdministration
    New-WebBinding -Name $SiteName -IP "*" -Port 443 -Protocol https
    $SSLCert = Get-ChildItem -Path Cert:\LocalMachine\My | Where-Object {$_.Subject.Contains("CertFriendlyName")}
    $Binding = Get-WebBinding -Name $SiteName -Protocol "https"
    $Binding.AddSslCertificate($SSLCert.GetCertHashString(), "My")
    Write-Host "Setup successful on: $env:COMPUTERNAME"
}
...
~~~

On top of any output you force to the console, there will also be some output for the certificate once it is imported successfully.

I like to cleanup my messes as well, so I add a few lines to remove the folder with the .pfx file in it that I copied to the machine. This will reside inside the foreach loop, preferably toward the end:

~~~
Remove-Item -Path "\\$Server\C$\Windows\temp\CertAssets" -Recurse -Force
Write-Host "Cleanup on $Server completed!"
~~~

## Require SSL on your site

This next bit of commands will be used however you roll out your deployment. If you want this to be required straight away, just throw it in with the rest of the code above and you are done. You may have a phased rollout in production, which may cause you to want to enable this on specific servers. Either way you craft this, the meat of the work will be these few lines:

~~~
Import-Module WebAdministration
Set-WebConfiguration -Location "Default Web Site" -Filter 'system.webserver/security/access' -Value "Ssl"
~~~

<p align="center">
    <img style="border:2px solid black"; src="/img/posts/import_cert_2.png">
</p>

Notice that the _-Location_ parameter is the site you want to require SSL for in IIS. This will now force all secure connections to your new binding with its associated certificate.

## Certified certificate certifier!

I’m hoping you found this post helpful in rolling out certificates to your environment, and I also hope it saves you a little bit more time in the future. I know this will save me time in the long run. Feel free explore the many other ways into import certificates. There are plenty of variables that will suit your environment, but this should give you a framework to shape your own script(s) to your environment. I left a lot of try/catch and logging out of my snippets, but to get the full-fledged version with error-handling and logging, head on over to my [Power-Shell-Automation GitHub repo](https://github.com/GeekLifeNow/PowerShell-Automation){:target="_blank"} and see what else may help you in your SysAdmin/IT Support endeavors! 
