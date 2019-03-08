---
layout: post
title: Create Scheduled Tasks
subtitle: Create a scheduled task to run PowerShell script via GUI or with PowerShell.
tags: [PowerShell, automation, task scheduler]
---
In my environment we have a good number of scheduled tasks that run on various Windows servers. Don’t ask me what they do, that’s for the dev team to know and for me to setup! I try my best to push using PowerShell and it is starting to gain adoption, which is great. That being said, how exactly would one go about running a PowerShell script (.ps1) from a scheduled task?

I’ll show you two ways, one from the plain GUI (yes, I know…gross) and then the cool-kids way, using PowerShell…to create a scheduled task that runs PowerShell.
<p align="center">
  <img src="/img/posts/sched_task_tut_0.jpg">
</p>
## Setup a scheduled task to run a PowerShell script:

1. First, open the Task Scheduler. I am using the Windows 10 GUI for the sake of this demonstration.   There’s not much difference in the interface from the more recent versions of Windows Server. Click **Action > Create Task…**:
   
     ![Create Task](/img/posts/sched_task_tut_1.jpg)

2. Give your task a name and a description and any other specific options that fit your environment:
    
    ![Name task and other settings](/img/posts/sched_task_tut_2.png)

3. Go to the **Triggers** tab and select **New…** and then select your options for when you want the task to run. I want my task to trigger daily starting on May the 4th (be with you) @ 12:00 AM and run every 20 minutes for 24 hours. This essentially will run indefinitely. Set any other options and then click **OK**.
    
    ![Task trigger settings](/img/posts/sched_task_tut_3.jpg)

4. Go to the **Actions** tab and select **New…**. In order for the task to run a PowerShell script, you’ll need to set the Action: to **Start a program**. Then in the Program/script: field, enter **_powershell.exe_**, as this will start the PowerShell session.

    The Add arguments (optional): field is where you will define which PowerShell file to run, in this case I am running **_-File    
    C:\Scripts\YourScript.ps1_**. Be sure to input the _-File_ parameter before you enter the location of the script, as this is what 
    tells PowerShell to run a script from a file. Click **OK** to confirm this section.
   
   ![Task Action settings](/img/posts/sched_task_tut_4.jpg)

5. At this point, you have enough setup that this can be finished by clicking *OK*. Feel free to set any other settings to your liking in the *Conditions* or *Settings* tabs.

Now you have a task that will run C:\Scripts\YourScript.ps1 on your given schedule!

_*One thing you will need to have in place is to allow your machine to run PowerShell scripts. By default, Windows will not run PowerShell scripts which is intentional on behalf of security. You will need to set the desired Set-ExecutionPolicy to fit your environment. More on that [here]( https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/set-executionpolicy?view=powershell-6){:target="_blank"}_

## Setup a scheduled task to run a PowerShell script…via PowerShell!

Here is a portion of a PowerShell script that will set up this same task:

Here we set the variables needed to build the task:
~~~
$STName = "Some Task"
$STDescription = "A task that will live in infamy!"
$STAction = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-File C:\Scripts\YourScript.ps1"
$STTrigger = New-ScheduledTaskTrigger -Daily -At 12am
$STSettings = New-ScheduledTaskSettingsSet
$STUserName = "user_account"
~~~
This will allow you to securely input the password for the account that will run the task. This is _much better_ than just plopping the password in plain text inside the script. When you run this build script, you will be prompted by the OS to input username/password. Your InfoSec team will thank you!
~~~
$STCredentials = Get-Credential -UserName $STUserName -Message "Enter password"
$STPassword = $STCredentials.GetNetworkCredential().Password
~~~
This part builds the task.  Using the Register-ScheduledTask cmdlet:
~~~
Register-ScheduledTask -TaskName $STName -Description $STDescription -Action $STAction -Trigger $STTrigger -User $STUserName -Password $STPassword -RunLevel Highest -Settings $STSettings
Start-Sleep -Seconds 3
~~~
Because we are using the _-Daily_ parameter for the task, we have to set some of the schedule settings separately from the building sequence. This will fetch the task and set the needed options for this to run every 20 minutes ($STModify.Triggers.repetition.Interval = 'PT20M') for a period of 24 hours ($STModify.Triggers.repetition.Duration = 'P1D'):
~~~
$STModify = Get-ScheduledTask -TaskName $STName
$STModify.Triggers.repetition.Duration = 'P1D'
$STModify.Triggers.repetition.Interval = 'PT20M'
$STModify | Set-ScheduledTask -User $STUserName -Password $STPassword
~~~
This has been very helpful lately, as we continue to deploy core versions of Windows Server 2016/2019 in the environment. Yes, you can get to a GUI of the Task Scheduler, but only remotely using an mmc snap-in. I’m sure this script will come in handy as we continue to automate in the environment!

Check out my [PowerShell GitHub repo](https://github.com/GeekLifeNow/PowerShell-Automation){:target="_blank"} for this and other scripts that may help you!
