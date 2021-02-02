---
layout: post
title: 'How to add AD Users/Groups to Local Groups'
subtitle: 'A time saver when it comes to adding access to AD users & groups to a PC or server's local groups.'
share-img: /img/posts/bulk_change_homedirectory_path_0.png
tags: [PowerShell, AD, Active Directory, local groups, bulk changes, permissions]
---
This is another quick post that might save some of my fellow SysAdmins and IT support people some time when needing to make some similar changes. I recently needed to add access to some Hyper-V hosts that are running Hyper-V Server 2016. This OS has the same look and feel as a Server 'Core' installation, so managing it is a bit different without the warm and fuzzy GUI that we all love. I needed to add a specific Active Directory security group to the local Hyper-V Administrators group that natively resides on this OS. Rather that using the Microsoft Management Console (MMC) to add this along with lots of licking, I figured I would try to build a PowerShell script to do this in bulk. I ended up testing with a single-interactive method and then just structured out the bulk part when I knew the single version worked.

## Adding an AD group or user to a local group on one host interactively:

~~~
$Domain = "geeklifenow.com"
$ServerName = Read-Host "Server"
$LocalGroup = "Hyper-V Administrators"
$DomainGroup = "GLN-HyperVMgmt"
([ADSI]"WinNT://$ServerName/$LocalGroup,group").add(([ADSI]"WinNT://$Domain/$DomainGroup").path)
Write-Host "$DomainGroup has been added to $LocalGroup on $ServerName!"
~~~

This code will prompt for a host (server) name and will add the AD security group named _GLN-HyperVMgmt_ to the local group _Hyper-V Administrators_ on the server provided. The beauty of this tiny script is that you can customize by simply changing the variables _$LocalGroup_ and _$DomainGroup_ to prompt for input by just changing the variable declarations with thge cmdlet Read-Host:

~~~
$LocalGroup = Read-Host "What local group do you want to add to?"
$DomainGroup = Read-Host "What AD group do you want to add?"
~~~

Want to just add a single AD user and not a group?

~~~
$Domain = "geeklifenow.com"
$ServerName = Read-Host "Server"
$LocalGroup = "Hyper-V Administrators"
$DomainUser = "din.djarin"
([ADSI]"WinNT://$ServerName/$LocalGroup,group").add(([ADSI]"WinNT://$Domain/$DomainUser").path)
Write-Host "$DomainUser has been added to $LocalGroup on $ServerName!"
~~~

## Adding an AD group or user to a local group on a bulk list of hosts:

This is where we make our money...in bulk...automation that is! I'm still waiting to make money in bulk...but then again, aren't we all?

To do this, we just need to feed PowerShell a listing of hosts via a .txt file and wrap the commands inside a __foreach__ loop:

~~~
$Domain = "geeklifenow.com"
$ServerNames = Get-Content -Path C:\HostListings\Hosts.txt
$LocalGroup = "Hyper-V Administrators"
$DomainGroup = "GLN-HyperVMgmt"

foreach ($ServerName in $ServerNames) {
    ([ADSI]"WinNT://$ServerName/$LocalGroup,group").add(([ADSI]"WinNT://$Domain/$DomainGroup").path)
    Write-Host "$DomainGroup has been added to $LocalGroup on $ServerName!"
}
~~~

Just as in the last section, feel free to customize the input by utilizing Read-Host for various needs.

## Wrap-Up

As you can see just a few lines of PowerShell can save you lots of time if you find yourself in a similar situation. This was my first exposure to the _ADSI_ query component and it is a very useful component. As like with anything else, their may be twelve different ways to do this without using ADSI but this was the fix for me in my environment.

You can check out my [GitHub repo](https://github.com/GeekLifeNow/PowerShell-Automation){:target="_blank"} for this and other scripts that may help you in the day to day management of your environment.
