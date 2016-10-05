---
layout: post
title: "Add a heartbeat end-point to an azure worker role"
category: "cloud"
tags: ["azure", "OWIN"]
---
{% include JB/setup %}

### Some background...

For one of my projects, I have a worker role on azure that performs some action in the background. In fact there are several of them that work in an orchestration together.

While building these workers, I noticed that I'm going to need one way to find out whether the worker is OK in the background and possibly get some very abstract information out of it. Having such a health monitorning is vital for a system which is composed of many background workers.

<!--more-->

So the main question is:

### How can we add a minimalistic http end-point to a worker

**Short answer:** OWIN

**Long answer:** OWIN provides the toolset to host a web server on a running process. Bingo!

First define your server behavior: this is a very simple action of building a response for incoming requests. In this case I want to output the version of the running worker.

```csharp
  public class Startup
  {
    public void Configuration(IAppBuilder app)
    {
      app.Run(context =>
      {
        var assembly = Assembly.GetExecutingAssembly();
        var fvi = FileVersionInfo.GetVersionInfo(assembly.Location);
        var version = fvi.FileVersion;

        context.Response.ContentType = "application/json";
        return context.Response.WriteAsync("{ \"version\": \"" + version + "\" }");
      });
    }
  }
```

then you need to write a code to start your server:

```csharp
  public static class Server
  {
    public static IDisposable Start()
    {
      var endpoint = RoleEnvironment.CurrentRoleInstance.InstanceEndpoints["Heartbeat"];
      var url = string.Format("{0}://{1}/health", endpoint.Protocol, endpoint.IPEndpoint);

      var options = new StartOptions(url)
      {
        ServerFactory = "Microsoft.Owin.Host.HttpListener"
      };

      return WebApp.Start<Startup>(options);
    }
  }
```

then all you need to do is to call this from `OnStart()` of your worker role class. Remember to store the return `IDisposable` in a local variable and `.Dispose()` it on `OnStop()`.

Finally add an end-point to your `ServiceDefinition.csdef` file to specify what port to run it on. -- in this case I use 3000

```markup
  <WorkerRole name="MyWorkerRole" vmsize="Small">
    <Endpoints>
      <InputEndpoint name="Heartbeat" port="3000" protocol="http" localPort="3000" />
    </Endpoints>
  </WorkerRole>
```

Now if your deploy to your cloud service, you if you open `http://<your-cloudservice-url>:3000/health` in your browser, you should get:

```bash
{ "version": "1.0.0.0" }
```
