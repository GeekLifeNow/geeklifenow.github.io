---
layout: post
title: 'Wheelhouse: BatchPatch'
subtitle: A tool that started out as a Windows patching solution that does WAY more than that! 
share-img: /img/posts/BP_logo.png
tags: [PowerShell, automation, Windows patching, WSUS, tools, BatchPatch]
---
A few years ago, I started with a Google search for something that could help me patch my critical servers rather than manually patching them one by one on the weekends, thus eliminating any kind of social life on said weekend. I have a life to live, ya know...and I'd rather not spend any more time watching a progress bar move across the screen than I already have.

Most of the servers in my environment I have set to update on their own and reboot when they need to. They grab updates from a WSUS server that runs on a Windows Server 2016 Core instance and after I approve updates, they are set on auto-pilot. The critical servers, though, need a little bit more babysitting only because the business wants the warm and fuzzies knowing that the servers have been patched and rebooted and they are ready for the dev teams to validate applications. My base list of critical servers runs about 40 hosts and patching them, with the help of BatchPatch, is as easy as sleeping in on Saturday morning...which is exactly what I do while this happens!

## Here's a quick run-down of how I implement critical server patching in my environment:

1. After the change has been approved, I import my list of servers from a static _servers.txt_ file. This adds my list of servers in the the host grid inside BatchPatch. *At this point I have already _"Approved for install"_ all the needed updates on the WSUS server.

<p align="center">
  <img src="/img/posts/wh-batchpatch_0.jpg">
</p>

2. Select all the servers on the grid by pressing **CTRL-A**. Then select **Actions > Windows updates > Download and install updates** Once This will force the hosts to:
   1. Check in with WSUS to see what updates are available.
   2. Download the updates. _(A handy progress bar for each host will appear to give an indication of where the overall download process is.)_
   3. Install the updates. _(Progress bar for the updates will appear for each host.)_
   4. Report the final status of the installations and whether or not a reboot is required.

<p align="center">
  <img src="/img/posts/wh-batchpatch_1.jpg">
</p>

<figure align="center">
  <img src="/img/posts/wh-batchpatch_2.png" />
  <figcaption>Image used from BatchPatch's site...</Figcaption>
</figure>

3. At this point, I may have maybe a half-dozen of hosts that actually do not need rebooted. So I'll select them and then remove them from the grid. Those are done!
4. The rest of the hosts that require a reboot I will schedule the reboot for a time of my choosing by selecting all the hosts in the grid with **CTRL-A** and then select **Actions > Task Scheduler > Create/modify scheduled task** _(I also need to make sure the task scheduler is enabled within BatchPatch. By default, it is set to off.)_
 
<p align="center">
  <img src="/img/posts/wh-batchpatch_3.jpg">
</p>

5. I set the _Task:_ to **Reboot (shutdown.exe /r /f /t 0)**, set the _Reference:_ time to execute, set _Recurrence:_ to **None** and then click **OK**.

<p align="center">
  <img src="/img/posts/wh-batchpatch_4.jpg">
</p>

6. The grid will then show you what and when the scheduled task will execute.

<p align="center">
  <img src="/img/posts/wh-batchpatch_5.jpg">
</p>

7. You could essentially walk away and let this ride, but like most decent sysadmins, I like to know that everything has completed when I wake up from sleeping in. I then add another "host" to the grid. It's not an actual machine, but a place holder...I name it _EMAIL_. I then select just the _EMAIL_ host and then select **Actions > Task Scheduler > Create/modify scheduled task** for just that entry in the grid. I set the _Task:_ to **Send email notification**, set the _Reference:_ time to execute (In this case, I want to send the email a good amount of time after reboots to give them enough time to apply updates on either side of the reboots.), set _Recurrence:_ to **None** and then click **OK**. _*BatchPatch has some flexibility as to what gets sent in the 'Send email notification', so check the settings for email notifications within BatchPatch._

<p align="center">
  <img src="/img/posts/wh-batchpatch_6.jpg">
</p>

8. Now the setup is completed! Sleep away and when I wake up, I check my email and find out the results. To this day, I have yet to have any issue with reboots not working. If there was a problem, this report would indicate that and I would investigate accordingly. I then can spend targeted time on any hosts that are not reporting as "online" Here is an example of a report. I have my email notifications to send an .html file of the entire grid:

| **Host**   | **Ping Reply**                     | **Remote Agent Log**           | **All Messages**               |
|------------|------------------------------------|--------------------------------|--------------------------------|
| GLN-DC01   | _Reply from 192.168.0.54 time=0ms_ | GLN-DC01 04/13/2019 02:07:11   | Sat-01:07:18 GLN-DC01 ONLINE   |
| GLN-DHCP01 | _Reply from 192.168.0.74 time=0ms_ | GLN-DHCP01 04/13/2019 02:08:05 | Sat-01:06:25 GLN-DHCP01 ONLINE |

At this point, I will do the necessary administrative tasks of completing this change in our change management system and email the local tech group so that they can begin validation on any applications.

##In Summary...

BatchPatch is a great tool! I started using it to just manage WSUS updates, but now I use it for lots of other automation tasks such as:

1. Deployment of applications (.exe, .msi, etc.)
2. Running remote .bat, .ps1, .reg, or other type files
3. Gathering information from servers or PCs like disk space usage, RAM utilization, OS version, logged in users, software that is installed, and a **lot** more
4. Setting up of what's called 'Job queues' which you can set a list of various actions and deployments to run on hosts, and then schedule them if needed

I even pushed out to a group of ~20 PCs a Windows 10 upgrade from 1709 to 1803! Now that was a time saver, and **super** easy!

Check out BatchPatch by going to their [website](https://batchpatch.com){:target="_blank"}. They have a trial version in which you can use to manage up to 4 hosts in a grid to see what you can do with it! After I started using it, we purchased it for our entire team to use at their respective sites. It's quite a handy tool! I now try to schedule and automate as much as I can with this.