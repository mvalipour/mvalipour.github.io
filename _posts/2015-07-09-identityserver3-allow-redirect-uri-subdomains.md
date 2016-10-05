---
layout: post
title: "Allow redirect URI subdomains in IdentityServer3"
description: ""
category: "back-end"
tags: [identityserver, openid, security]
---
{% include JB/setup %}

[IdentityServer3](https://github.com/IdentityServer/IdentityServer3) is an awesome library that allows you to build [in no time] an single sign-on server using .NET and OWIN/Katana.

When client applications send a user to the server, they must
provide a `redirect_uri` as a parameter which is used to send the user
back to the application with the token.

One of the security measures build into IdentityServer is restricting
the list of URIs allowed (for each client).


```csharp
var client = new Client
{
    ClientName = "JS Client",
    ClientId = "implicitclient",
    RedirectUris = new []
    {
        "https://www.myapp.com/callback.html",
    }
}  
```

The default behavior of IdentityServer is to do an exact match on these
(case insensitive).

But what if we have [an unpredictable list of] subdomains in our app? Exact matches won't work in this case.

<!--more-->

What we wish here is to have this:

```csharp
    RedirectUris = new []
    {
        "https://*.myapp.com/callback.html",
    }
```

Luckily IdentityServer3 have support for overriding the default validator,
and what we need to do is to write one that does match using wildcards (as suggested above):

```csharp
public class WildcardRedirectUriValidator : DefaultRedirectUriValidator
{
    public override Task<bool> IsRedirectUriValidAsync(string requestedUri, Client client)
    {
        return MatchUriAsync(requestedUri, client.RedirectUris);
    }

    public override Task<bool> IsPostLogoutRedirectUriValidAsync(string requestedUri, Client client)
    {
        return MatchUriAsync(requestedUri, client.PostLogoutRedirectUris);
    }

    private Task<bool> MatchUriAsync(string requestedUri, List<string> allowedUris)
    {
        var rules = allowedUris.Select(ConvertToRegex).ToList();
        var res = rules.Any(r => Regex.IsMatch(uri, r, RegexOptions.IgnoreCase));
        return Task.FromResult(res);
    }

    private const string WildcardCharacter = @"[a-zA-Z0-9\-]";

    private static string ConvertToRegex(string rule)
    {
        if (rule == null)
        {
            throw new ArgumentNullException(nameof(rule));
        }

        return Regex.Escape(rule)
                    .Replace(@"\*", WildcardCharacter + "*")
                    .Replace(@"\?", WildcardCharacter);
    }
}
```

Above code defines a new validator class that inherits the default validator
and only overrides the URI matching methods.

The new matcher converts the pattern URI to a regular expression by
replacing `*` with `[a-zA-Z0-9\-]+` (on or more characters) and `?` with
`[a-zA-Z0-9\-]` (You can choose a different set of characters to match) and
then match it against the URI.

Note that here we also change validation of `PostLogoutRedirectUri` which is
another parameter used for logout process. (see [documentation](https://identityserver.github.io/Documentation/docs/configuration/clients.html)).

Finally you would need to configure this validator when setting up your identity server.

```csharp
public void Configuration(IAppBuilder app)
{
    var options = new IdentityServerOptions
    {
        //...
        Factory = new IdentityServerServiceFactory
        {
            RedirectUriValidator = new Registration<IRedirectUriValidator>(typeof(WildcardRedirectUriValidator));
        }
    };

    app.UseIdentityServer(options);
}
```
