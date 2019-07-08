---
layout: post
title: 'Bulk Change Home Drive Path in Active Directory'
subtitle: A quick and easy way to update the home drive paths for multiple users.
share-img: /img/posts/bulk_change_homedirectory_path_0.png
tags: [PowerShell, AD, Active Directory, home directory, bulk changes]
---
This will be a quick walk-through but will also make quick work should you need to do something similar with this process! We recently went through an Active Directory migration and in the process, our local DFS infrastructure became inconsistent. Any shares residing in the DFS namespace were becoming unstable and users could not consistently connect to them. Kind of a problem when all the user home drives are set with DFS paths...

## The task at hand:

Replace the DFS share locations with the literal UNC path for all the user home drives at my site.

For example: Silence Dogood's current home drive points to _\\company.com\dfs-site\home directories\sdogood_ and it needs to be changed to _\\fs01\siteusers\sdogood._

...for Silence and about 200 of his co-workers!

## The solution:

Just like anything else in the world of sysadmins, there's plenty of different ways to do the same thing. Time was of the essence for me on this, so I had to move fast. The service desk tickets were rolling in for this issue at a growing rate, so I did it quick 'n' dirty!

Because all of my site's users reside in on OU, I grabbed all the users in the OU, filtered for any user that has anything in the _HomeDrive_  location and grabbed some other identifying information and spit it out in a .csv file:

~~~
Get-ADUser -Filter 'HomeDrive -ne "$Null"' -SearchBase "OU=Users,OU=localsite,OU=Sites,DC=company,DC=com" -Property SamAccountName, HomeDirectory | Export-Csv -Path C:\Temp\HomeDrives.csv -encoding ascii -NoTypeInformation
~~~

This gave me a humble .csv file that looked like this:

_SamAccountName	HomeDirectory_
silence.dogood	\\fs01\siteusers\sdogood

I then opened Excel and performed a Find & Replace to swap out the paths in the HomeDirectory column. Save...duh!

I then created a PowerShell script to take the new paths and replace the existing ones for the users. Because I pulled the SamAccountNames and they are in the .csv, I can just use Set-ADUser rather than retrieving them again. Here's what it looks like:

~~~
Import-Csv -Path C:\Temp\HomeDrives.csv | foreach {
    $sam = $_.SamAccountName
    $path = $_.HomeDirectory
    Write-Verbose -Message "HomeDirectory path for $sam has been modified to $path" -Verbose
    Set-ADUser -Identity $sam -HomeDirectory $path
}
~~~

This basic process is:

1. Importing the .csv data
2. Finding the user object using the $sam variable which is the current SamAccountName in the foreach loop
3. Changing the HomeDrive path by dropping in the $path variable which is the current HomeDirectory value in the foreach loop via Set-ADUser

Done! I threw in the verbose message so I could send something pretty for the team to see that this was completed.

Like I said...quick 'n' dirty! No error handling or logging on this. I may take time to do it, but it got the job done. For those sysadmins out there, who just need stuff done quick so you can go put out the next fire, you'll hopefully find it in your heart to forgive the lack of 'beautification' of this process.

## Up the geek-points by using Regular Expressions!

One way I started to think about this was just parsing the HomeDrive path and replacing everything up to the last backslash in the path with my new path. This was way too time consuming in the moment, but I know it can be done...essentially a find/replace on the fly. Take a look at this [post](https://powershellexplained.com/2017-07-31-Powershell-regex-regular-expression/){:target="_blank"} to get some more insight on how you can leverage regex in getting things done in PowerShell!