---
layout: post
title: 'PowerShell: Change Shortcut Target'
subtitle: A PowerShell script to change the target of a .url/.lnk file on multiple PCs.
share-img: /img/posts/shortcut_target_0.png
tags: [PowerShell, automation, desktop management, shortcuts]
---
<p align="center">
  <img style="border:2px solid black" src="/img/posts/shortcut_target_0.png">
</p>

Recently, as a part of retiring legacy Windows Server OS(s), I had to find a quick and dirty way to change the target location for a certain file in the Public desktop folder on all the machines in my environment. Yes, I know...any shortcuts should be deployed via Group Policy...I get it. At the time, this was the most efficient way to deploy given the access and timelines when these were deployed. I am slowly but surely utilizing GPOs to manage shared printers, mapped drives, and desktop shortcuts and settings.

So here was the task at hand:

_Search through a list of PCs and modify the target address on a specific web shortcut that pointed to an intranet page that resides on the C:\Users\Public\Desktop._

Seems simple enough, right? _grin._ Of course, the more I dug into the tasl, the more I wanted to add. Which has definitely helped with challenging myself in creating custom scripts for my environment. I have recently set out to dig in with PowerShell, and though I do watch some videos and read my go-to book, [Learn Windows PowerShell in a Month of Lunches](https://www.manning.com/books/learn-windows-powershell-in-a-month-of-lunches-second-edition){:target="_blank"}, there's honestly been no better learning than just starting with a simple, mundane task and work at automating it!

## Mass-changing the target address for a .url/.lnk on a list of PCs:

The beginning variables are short and sweet:

~~~
$LogFile = "C:\Path2Logfile\Log.txt"
$Computers = Get-Content -Path C:\Scripts\computers.csv
$Shell = New-Object -ComObject WScript.Shell
$NewTarget = "https://geeklifenow.com/"
~~~

I won't cover logging in depth, but you should use your preferred method of logging to get detailed feedback on this task. I have a pre-made .csv file with a list of my targeted computers inside. The _$Shell_ variable is the key to getting into manipulate the shortcut file. Finally, _$NewTarget_ is the address we want to set on the shortcut.

I place the workload inside a foreach loop and the task is as follows:

1. Check to see if the PC is online using the _[Test-Connection cmdlet](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/test-connection?view=powershell-6){:target="_blank"}_. This will return either a _True_ or _False_ result.)

2. If it is online, check for the shortcut file in the _C:\Users\Public\Desktop_ folder. I am using two conditions to make sure I grab the appropriate shortcut: The file ends with .url **AND** the file has the string 
"web" in the name. This is where I know my environment enough that I know I will grab the right file. You will need to use the necessary conditions to grab the file you want in your environment.

3. _$ShortcutToChange_ will be populated with either the object information or it will be $null (It will not error if it doesn't find anything!), so then I check to see if I have anything by simply looking to see if the length of the variable is not 0: ***if ($ShortcutToChange.Length -ne 0)***

4. Then the magic happens! Which, when you see it, it's quite anti-climactic in the fact that it's a few lines to make the change.

5. I have been working a lot lately with the try/catch structure to get a better handle on error handling and logging, so I'm trying to catch any errors that may arise. I'm just logging any error that comes up, rather than catching a specific one. Catching specific errors will be something that I would like to work toward.

~~~
foreach ($Computer in $Computers) {
    $Online = Test-Connection -BufferSize 16 -Count 2 -ComputerName $Computer -Quiet
    try {
        if ($Online -eq $true) {
            try {
                $ShortcutToChange = Get-ChildItem -Path \\$Computer\C$\Users\Public\Desktop -ErrorAction Stop | Where-Object { $_.Extension -eq ".url" -and $_.Name -like '*web*' }
            }
            catch {
                $ErrorMessage = $_.Exception.Message
                Write-Log -Text "WARNING: $Computer is online, but error getting to the Public\Desktop! Error: $ErrorMessage"
                Continue
            }
            if ($ShortcutToChange.Length -ne 0) {
                try {
                    $url = $shell.CreateShortcut($ShortcutToChange.FullName)
                    $url.TargetPath = $NewTarget
                    $url.Save()
                    Write-Log -Text "SUCCESS: $Computer's GeekLifeNow link was successfully modified!"
                }
                catch {
                    $ErrorMessage = $_.Exception.Message
                    Write-Log -Text "WARNING: Error saving new GeekLifeNow link: $ErrorMessage"
                }
            }
            else {
                Write-Log -Text "$Computer does NOT have a GeekLifeNow link. Nothing to see here..."
            }
        }
        else {
            Write-Log -Text "WARNING: $Computer is offline."
        }
        }
    catch {
        $ErrorMessage = $_.Exception.Message
        Write-Log -Text "ERROR: Error connecting to $Computer : $ErrorMessage"
    }
}
~~~

So I tried to catch anywhere in which there would be an issue, and log the error. I also built in logging if the computer was offline. This is good in case you want to run another pass to get any PCs that were sleeping or powered off in the future. I'm also playing around with the text in logging with the SUCCESS/WARNING/ERROR designations on each line.

**BOOM!** I now have _x_ number of shortcuts modified so that they are no longer pointing to our legacy webserver without having to trot to each machine and have 3 meaningless conversations in passing, 12 extra fly-by IT requests, and a big waste of my life doing such a mundane task. My current environment consists of ~450 desktops, so I surely will be using GPOs to place these types of things on the desktop fleet, but it was nice to learn this task as it has given me a better understanding of PowerShell.

You can see this script in its entirety in my [GitHub](https://github.com/GeekLifeNow/PowerShell-Automation){:target="_blank"} repo.