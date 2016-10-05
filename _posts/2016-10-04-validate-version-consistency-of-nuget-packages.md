---
layout: post
title: "Validate version consistency of NuGet packages in a solution (in PowerShell)"
description: ""
category: "back-end"
tags: [nuget, powershell]
---
{% include JB/setup %}

If you use NuGet you have will realise that there are times where same of your package may have different versions installed in different parts of your solution.

This could result in run-time inconsistencies and is something you would want to avoid.

<!--more-->

Visual studio has a tool that will alert you about this:

![](http://i.imgur.com/IqFSGYa.png)


But what about the build server? How do we make sure that the changes someone has proposed isn't breaking the consistency of versions?

I was hoping NuGet command line would provide me with a command to validate (or at least retrieve this).

But nevertheless, that's why I wrote this little PowerShell script to do that.

```language-powershell
Get-ChildItem -Recurse -Depth 5 -Path "." |
    ? { $_.Name -eq "packages.config" } |
    % { ([xml](Get-Content -Path $_.FullName)).packages.package | select id, version } |
    Group-Object -Property id |
    select Name, @{ Name="Versions"; Expression= { $_.Group | select -ExpandProperty version -Unique }} |
    ? { $_.Versions.Count -gt 1 } |
    Sort-Object -Property Name |
    % { Write-Error "Package '$($_.Name)' has multiple versions used across the solution! $($_.Versions)" }
```

### Further work

Note that the script reads through all `packages.config` files in the solution directory. This may include the projects that live in the same directory but not referenced in the solution. So if that's what you need, you would need to read through your `solution.sln` file to extract the list of projects + path first.
