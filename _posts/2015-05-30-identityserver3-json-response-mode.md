---
layout: post
title: "Adding JSON response mode to IdentityServer3"
description: ""
category: "back-end"
tags: [identityserver, openid, security, owin]
---
{% include JB/setup %}

We recently started building a solution composed of many websites. For authentication, we use [IdentityServer3](https://github.com/IdentityServer/IdentityServer3) to build single sign-on using [OpenID connect](http://openid.net/connect/) protocol.

This is generally how a normal login scenario works:

1. You open a page on `site-a.com`
2. Because you are not logged in, `site-a` will send you to `my-idserver.com/connect/authorize` via a `302` response
3. You login on `my-idserver.com` (or may well be already logged in), and your browser does a `POST` to `site-a.com` (with token parameters)
4. `site-a.com` will handle the posted token and log you in.

This scenario fits most of our needs; However, we noticed we have need that is not covered by in IdentityServer3.

> We need to call into `my-idserver.com` (from a javascript) and retrieve the token for the already logged in user.

<!--more-->

In other words:

- You visit a secured page on `site-a.com` (at this point we know you must be logged in on `my-idserver.com`)
- a script loads from `site-b.com` on a page on `site-a.com`
- the script needs a token...

**Note** that a solution in which the script receives a token from `site-a.com` is not acceptable because `site-a.com` maybe any website outside the scope of our control. (still using our identity server).

In this article I'm going to explain how we created an OWIN middleware to add support for `json` response to `IdentityServer3`.

## Solution

IdentityServer uses `response_mode` parameter to determine the type of response send back to the requester. Possible options for this parameter are:

- `form_post`: this is the value used in the standard scenario, where idServer returns an html response that contains a `<form>` with all of the response parameters as hidden fields.
- `fragment`: in this case idServer response is a `302` status code with location header set to `redirect_uri` plus the response parameters set as the hash/fragment part.
- `query`: identical to `fragment` except in this case response parameters are set as query strings.

What we do want from a solution, is a new type of `response_mode` called `json`.

The only thing that is different in this mode (in any other mode in fact), is the way response is passed to the requester.  
So what we will be adding to our identity server is a proxy middleware to sit before the IdentityServer middleware.

The first thing we need to setup is the middleware pipeline:

```language-csharp
  public void Configuration(IAppBuilder app)
  {
      app.Use<JsonHandlerMiddleware>();

      var idsrvOptions = new IdentityServerOptions { ... };
      app.UseIdentityServer(idsrvOptions);
  }
```

This ensures that for incoming requests, the `JsonHandlerMiddleware` is hit first, giving us the oportunity to intercept.

```language-csharp
    public class JsonHandlerMiddleware : OwinMiddleware
    {
        public JsonHandlerMiddleware(OwinMiddleware next) : base(next)
        {
        }
    }
```

The proxy works like this:

```language-csharp
    public override Task Invoke(IOwinContext context)
    {
        var mode = context.Request.Query["response_mode"];
        if (mode != "json")
        {
            return this.Next.Invoke(context);
        }

        var innerContext = CloneContext(context);
        innerContext.Response.OnSendingHeaders(state =>
        {
            if (!HandleJsonResponse(context, innerContext))
            {
                context.Response.StatusCode = 403;
            }
        }, null);

        return this.Next.Invoke(innerContext);
    }
```

- If the `response_mode` is not `json` we simply carry on with the next middleware (identity server), without any alteration.
- If it is:
  - Clone the context
  - Call into the identity server middleware
  - Before it finishes the response, intercept and write the json token to the output of the original context.
  - And if for any reason we couldn't finish, we simply return a `403` status code -- forbidden.

The reason why we clone the context, is to keep the original context untouched. Identity server changes response headers and body, so we need to make sure the cloned context has a different response header and body set to it so that its changes don't leak into/affect our `json` output.

Apart from that, we need to set `response_mode` to `fragment` to get the identity server function in that mode.

And set `response_uri` to the `origin` header of the request. This is to make sure that `json` model only works if the request comes from a safe origin. IdentityServer3 by default validates the `redirect_uri` (in this case the requester Origin) against a white-list of domains.

```language-csharp
    private static IOwinContext CloneContext(IOwinContext context)
    {
        var clonedQuery = context.Request.Query.ToDictionary(a => a.Key, a => a.Value.First());
        clonedQuery["response_mode"] = "fragment";
        clonedQuery["redirect_uri"] = context.Request.Headers["Origin"];

        var clonedEnv = context.Environment.ToDictionary(e => e.Key, e => e.Value);
        clonedEnv["owin.ResponseHeaders"] = new HeaderDictionary(new Dictionary<string, string[]>());
        clonedEnv["owin.ResponseBody"] = new MemoryStream();
        clonedEnv["owin.RequestQueryString"] = string.Join("&", clonedQuery.Select(a => $"{a.Key}={a.Value}").ToArray());

        return new OwinContext(clonedEnv);
    }
```

**IMPORTANT Note**: It is very crucial to use SSL (https) for all domains to avoid any man-in-the-middle attack -- altering the Origin header. It is not possible for any malicious javascript to alter `origin` header. However to make things even safer, we use CORS headers to allow only certain domains on `Access-Control-Allow-Origin` -- which is outside of the scopes of this article.

Finally we handle the json response:

```language-csharp
    private bool HandleJsonResponse(IOwinContext context, IOwinContext innerContext)
    {
        if (innerContext.Response.StatusCode != 302)
        {
            return false;
        }

        var clientId = context.Request.Query["client_id"];
        var client = Clients.FindById(clientId);
        if (client == null)
        {
            return false;
        }

        if (!client.JsonResponseAllowed)
        {
            return false;
        }

        return RewriteResponseAsJson(client, context, innerContext);
    }
```

First line checks the IdentityServer reponse status code and if was anything other than a `302` (redirection), we reject the request.

Then we are reading the client and making sure this client has json response enabled on it.

**Note 1.** this is a completely optional thing to implement and is only necessary if you have multiple clients and wish to allow `json` mode only for some of them.

**Note 2.** `JsonResponseAllowed` is not a property of standard IdentityServer `Client` class. You would have to add your own `MyClient` class that extends the `Client` -- I have skipped the code here for simplicity.

And here is how the `Rewrite` function works:

```language-csharp
    private const string AntiHijackingPrefix = "while(1);";

    private bool RewriteResponseAsJson(EmpactisSystem client, IOwinContext context, IOwinContext innerContext)
    {
        var location = innerContext.Response.Headers["Location"];
        if (string.IsNullOrWhiteSpace(location))
        {
            return false;
        }

        var uri = new UriBuilder(location);
        var queries = uri.Fragment?.Trim('#');
        if (string.IsNullOrWhiteSpace(queries))
        {
            return false;
        }

        var parts = queries.Split(new[] { '&' }, StringSplitOptions.RemoveEmptyEntries)
                           .Select(a => a.Split('='))
                           .Where(a => a.Length == 2)
                           .Aggregate(new NameValueCollection(),
                               (seed, current) =>
                               {
                                   seed.Add(current[0], current[1]);
                                   return seed;
                               });
        var token = parts["id_token"];
        if (string.IsNullOrWhiteSpace(token))
        {
            return false;
        }

        var obj = new
        {
            token,
            lifetime = client.IdentityTokenLifetime,
            nonce = context.Request.Query["nonce"]
        };
        var json = JsonConvert.SerializeObject(obj);

        context.Response.StatusCode = 200;
        context.Response.ContentType = "application/json";
        context.Response.Write(AntiHijackingPrefix + json);

        return true;
    }
```

In summary:

- Read `location` header
- Parse its query strings into a `NameValueCollection` -- to avoid case-sensitivity
- create a response object
- serialize it and write to the output

**Bonus feature** (optional) We prefix our json with a `while(1);` to [stop JSON Hijacking attack](http://stackoverflow.com/questions/2669690/why-does-google-prepend-while1-to-their-json-responses).

## Request a token as JSON

Now all we have to do to request our token as JSON is to construct a URL with the correct parameters and send a `GET` request to the identity server. (using your preferred way of sending ajax request -- e.g. `$http` in angular or `$.ajax` in jquery).

```language-javascript
  var nonce = Date.now() + "" + Math.random();
  var url =
      idServerUrl + "/connect/authorize?" +
      "client_id=" + encodeURI(opts.client_id) + "&" +
      "response_type=" + encodeURI(opts.response_type) + "&" +
      "scope=" + encodeURI(opts.scope) + "&" +
      "response_mode=json&" +
      "nonce=" + encodeURI(nonce);
```

**Note:** Remember to validate the nonce with the `nonce` field in the response.
