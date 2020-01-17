---
layout: post
title: 'Mass Update VMware Tools on Guest VMs'
subtitle: 'Use PowerCLI to update VMware Tools on multiple VMs in your cluster.'
share-img: /img/posts/vmware_tools_update_0.png
tags: [VMware, PowerShell, PowerCLI, virtualization, VMware Tools, maintenance]
---
If your environment is anything like mine, you are noticing that your physical server footprint is decreasing while your virtual server footprint is increasing. This is a good sign your org is at least moving in the right direction with maximizing resources by virtualizing workloads, applications, and infrastructure. The one thing that shifts is going to be your focus on how you keep up maintenance on these little VM clones you have running all over your vSphere cluster! 

OK, I'm a tad OCD when I am browsing through my vSphere environment, and I see that annoying little notification:

_A newer version of VMware Tools is available for this virtual machine._

I upgrade it...then I see another...upgrade...now I've forgotten what I came to do in the first place!

## Enter PowerCLI

I won't pretend to be a PowerCLI guru by any means. In fact, I literally just installed it before I wrote this post to do a pretty cool task that is going to save me lots of time in the future. The fact that within 15 minutes, I can save a boatload of time in the future, I'm guessing I need to start digging more into PowerCLI in the coming months!

What I like right off the bat with PowerCLI is that it feels so much like PowerShell. In fact, it is a specific Powershell module that is designed to work with Microsoft PowerShell. It has over 400 cmdlets that can be used to automate and manage your VMware environment. 

<p align="center">
    <img style="border:2px solid black" src="/img/posts/vmware_tools_update_1.png" border="2">
</p>

## The task at hand:

I want to be able to mass-update VMware Tools on multiple VMs in my environment without have to click/wait/click/wait, etc. We'll do this by:

1. Installing PowerCLI
2. Grabbing a list of VMs that need VMware Tools upgraded
3. Updating VMware Tools on a targeted list of VMs

## Installing PowerCLI

If you have any older PowerCLI versions (6.x and earlier), they will need to be removed from the traditional Programs and Features area on your Windows machine. Be sure to be connected to the Internet, as the module needs to be pulled down from an online repository. Also, make sure Windows Management Framework 5.1 is installed if you are working on a Windows 7/8/Server 2012 R2 or earlier OS.

Open a PowerShell window (as and administrator) and enter:

~~~
Install-Module VMware.PowerCLI
~~~

You may get prompted to install a dependency called NuGet as well as getting asked to install from an untrusted repository. I would confirm with a 'Y'  for these.

You may also want to change your station's local script policy:
(Feel free to make this setting as you need for your environment.)

~~~
Set-ExecutionPolicy RemoteSigned
~~~

This command will opt-out of the Customer Experience Improvement Program and will ignore any certficate warnings when using PowerCLI. I don't use custom certs for my purpose, so I'll decline on all the warnings:

~~~
Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Confirm:$false -ParticipateInCeip $false
~~~

If you have installed the PowerCLI module previously, but think it may need updated, you can check the version and/or update it by using:

~~~
Get-Module -Name VMware.PowerCLI -ListAvailable
Update-Module -Name VMware.PowerCLI
~~~

## Connecting to your vSphere cluster and updating VMware Tools on VMs

The first thing you need to do is connect to your vSphere cluster:

~~~
Connect-VIServer -Server "g1.geeklifenow.com"
~~~

You then will be prompted to login with the credentials needed to interact with vSPhere:

<p align="center">
    <img style="border:2px solid black" src="/img/posts/vmware_tools_update_2.png" border="2">
</p>

There are plenty of other options to use when connecting to your cluster, like passing the username/password, using stored credentials, etc. Check out the ways to connect [here.](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/e7c1a32c-a3c6-4d7c-91bb-18a86a38daf7/12353298-ce6e-4d3f-bd8d-ab9f5ab044cc/doc/Connect-VIServer.html){:target="_blank"} Remember, because this is all PowerShell, you can set variables for the connection information if you wish, for further scriped automation.

To get a quick listing of running VMs and the version of VMware Tools that is installed:

~~~
Get-VM | Get-VMguest | Where {$_.State -eq 'Running'} | Select VmName, ToolsVersion
~~~

To get a quick listing of running VMs that don't have a specific version on VMware Tools installed, let's say 10.3.5:

~~~
Get-VM | Get-VMguest | Where-Object {$_.State -eq 'Running' -and $_.ToolsVersion -notlike '10.3.5'} | Select VmName, ToolsVersion
~~~

To get a quick listing of running VMs that are showing they need updated:

~~~
Get-VM | Get-VMguest | where-object {$_.State -eq 'Running' -and $_.ExtensionData.ToolsversionStatus -eq 'GuestToolsNeedUpgrade'} | Select VmName
~~~

Now, if your cluster has mulitple Datacenters, like mine, you may want to just drill down to just your site. These previous commands will pull all the VMs from the entire cluster. To query just your datacenter site, prepend the commands with _Get-Datacenter -Name MydatacenterSite._ So the command to find all the needed updates for your datacenter site would be:

~~~
Get-Datacenter -Name MyDatacenterSite | Get-VM | Get-VMguest | where-object {$_.State -eq 'Running' -and $_.ExtensionData.ToolsversionStatus -eq 'GuestToolsNeedUpgrade'} | Select VmName
~~~

To update the VMs, one way to do this is to save all the VM names that need updated into a .csv file:
_(From here on out, I will be assuming that we are looking into the MyDatacenterSite.)_

~~~
Get-Datacenter -Name MyDatacenterSite | Get-VM | Get-VMguest | where-object {$_.State -eq 'Running' -and $_.ExtensionData.ToolsversionStatus -eq 'GuestToolsNeedUpgrade'} | Select VmName | export-csv C:\Temp\vms2update.csv -NoTypeInformation
~~~

Feel free to use comparisons to target specific guest OS versions with this query. For instance, because VMware Tools no longer is supported on Windows XP, I would need to exclude any VMs that are running it to use for updating. For this I would use:

~~~
Get-Datacenter -Name MyDatacenterSite | Get-VM | Get-VMguest | where-object {$_.State -eq 'Running' -and $_.OSFullName -ne 'Microsoft Windows XP Professional (32-bit)' -and $_.ExtensionData.ToolsversionStatus -eq 'GuestToolsNeedUpgrade'} | Select VmName | export-csv C:\Temp\vms2update.csv -NoTypeInformation
~~~

This will give you the .csv you need to run updates against using:

~~~
$VMs = Import-Csv C:\Temp\vms2update.csv
$VMs | % { Get-VM -Name $_.VmName | Update-Tools -NoReboot}
~~~

This will start the update process on each VM. The -NoReboot parameter is using the the following switch parameters for the installer:

_/s /v /qn REBOOT=ReallySuppress_

This then allows you to reboot your VMs inside your regular maintenance windows. According to the VMware [documentation for this cmdlet](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/e7c1a32c-a3c6-4d7c-91bb-18a86a38daf7/12353298-ce6e-4d3f-bd8d-ab9f5ab044cc/doc/index.html#linkb2c6f419e7481f8df0437dbcd81516fde6ac1b08;Update-Tools.html){:target="_blank"}, this may not keep the guest from rebooting:

_"However, the virtual machine might still reboot after updating VMware Tools, depending on the currently installed VMware Tools version, the VMware Tools version to which you want to upgrade, and the vCenter Center/ESX versions."_

...although if you are fairly current on all versions, I don't think you will have any issues with surprise reboots. I would caution doing this on critical production VMs during business hours, just to be safe.

So now the VMware Tools will be updated one at a time on each VM in your .csv, at least from my observation. There is a progress bar, but for me it sits at 0% and the quickly jumps to 99% then blips to 0 again. At least for now, I wouldn't count on an accurate % for progress, but if you watch, you can see that it is working through the list.

<p align="center">
    <img style="border:2px solid black" src="/img/posts/vmware_tools_update_3.png" border="2">
</p>

## Wrap-Up

Keeping your environment updated in all facets is quite a daunting task the larger your environment is. When you come across ways to speed it up or perform tasks in bulk, that is always worth the time to iron out, document, and streamline in order to save precious time and to give your clicking finger a break. This simple automation can be built out in even more complex ways to further automate updates. I could easily see this as a script that runs on a schedule to perform updates within a maintenance window. It would also be nice to see if there are any error catching options that can be used to document any errors that may come up. 

I also like the fact that this utilizes the 'cluster-provided' VMware Tools installer. I started to use BatchPatch to schedule and install updates using the .exe installer and switches, but then you must start managing different versions of installers.

I can't take full credit for this information. In my Googling, I came across this [blog post](https://www.ivobeerens.nl/2019/09/09/vmware-tools-installation-and-upgrade-tips-and-tricks/){:target="_blank"} by Ivo Bereens that walked me through most of the process. You can also browse the [PowerCLI Cmdlets Reference](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/e7c1a32c-a3c6-4d7c-91bb-18a86a38daf7/12353298-ce6e-4d3f-bd8d-ab9f5ab044cc/doc/index.html#linkca3978754e0753392fc9e7f5c20d9ffea739948e;Overview.html){:target="_blank"} to see what else this powerful tool can do for your VMware environment.

I sure hope this was helpful to you fellow geeks, techies, and sysadmins! 
