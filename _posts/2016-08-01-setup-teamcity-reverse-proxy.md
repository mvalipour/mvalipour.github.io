---
layout: post
title: "Setup TeamCity on a windows machine with reverse-proxy"
description: ""
category: "devops"
tags: [teamcity, devops, reverse-proxy, windows]
---
{% include JB/setup %}

In case you are installing TeamCity on a windows machine that already has another application listening to it's port 80 (in our case we have Octopus Deploy installed on the same machine), you would need to follow steps as below to get incoming port 80 traffic served by TeamCity.

<!--more-->

**Step 1 – Install TeamCity :** Go ahead and normally install a TeamCity server on your box.

**Step 2 – Server Configuration :**

- Stop TeamCity service (to then start again after the next step)
- Open `<TEAMCITY_DIR>/conf/server.xml` and add/change the following attributes on the `<Connector>` node.

```language-markup
<Connector
  port="8080"
  proxyName="teamcity.domain.com"
  proxyPort="80"
/>
```

It is very important to add the `proxyName` and `proxyPort`, otherwise part of the TeamCity server like the internal NuGet repository will produce invalid URLs.

**Step 3 – URL Rewiring / Reverse Proxy :**

- Install IIS and [Application Request Routing](http://www.iis.net/downloads/microsoft/application-request-routing)
- Add/Change default site to have http and/or https binding on port 80
- Select your site in IIS and go to *Url Rewrite*
- Add an Inbound Rule with the following details (If you prefer you can avoid the UI and directly add this to the `web.config` file under the physical path of your site)

```language-markup
<rule name="teamcity" enabled="true" patternSyntax="Wildcard" stopProcessing="true">
    <match url="*" />
    <action type="Rewrite" url="http://localhost:8080/{R:0}" />
</rule>
```

In case have other applications with the same need, you would need to provide `<condition>` for your rules and have one rule per application to control the traffic based on the `HTTP_HOST`

```language-markup
<rule name="teamcity" enabled="true" patternSyntax="Wildcard" stopProcessing="true">
    <match url="*" />
    <conditions>
        <add input="{HTTP_HOST}" pattern="^teamcity\.domain\.com$" />
    </conditions>
    <action type="Rewrite" url="http://localhost:8080/{R:0}" />
</rule>
<rule name="octopus" enabled="true" patternSyntax="Wildcard" stopProcessing="true">
    <match url="*" />
    <conditions>
        <add input="{HTTP_HOST}" pattern="^octopus\.domain\.com$" />
    </conditions>
    <action type="Rewrite" url="http://localhost:8081/{R:0}" />
</rule>
```
