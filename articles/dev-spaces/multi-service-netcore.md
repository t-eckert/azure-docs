---
title: "Running multiple dependent services using .NET Core and VS Code"
titleSuffix: Azure Dev Spaces
services: azure-dev-spaces
ms.service: azure-dev-spaces
author: DrEsteban
ms.author: stevenry
ms.date: "11/21/2018"
ms.topic: "tutorial"
description: "Rapid Kubernetes development with containers and microservices on Azure"
keywords: "Docker, Kubernetes, Azure, AKS, Azure Kubernetes Service, containers, Helm, service mesh, service mesh routing, kubectl, k8s "
---
# Multi-service development with Azure Dev Spaces

In this tutorial, you'll learn how to develop multi-service applications using Azure Dev Spaces, along with some of the added benefits that Dev Spaces provides.

## Call a service running in a separate container

In this section, you will create a second service, `mywebapi`, and have `webfrontend` call it. Each service will run in separate containers. You'll then debug across both containers.

![Multiple containers](media/common/multi-container.png)

### Download sample code for *mywebapi*
For the sake of time, let's download sample code from a GitHub repository. Go to https://github.com/Azure/dev-spaces and select **Clone or Download** to download the GitHub repository. The code for this section is in `samples/dotnetcore/getting-started/mywebapi`.

### Run *mywebapi*
1. Open the folder `mywebapi` in a *separate VS Code window*.
1. Open the **Command Palette** (using the **View | Command Palette** menu), and use auto-complete to type and select this command: `Azure Dev Spaces: Prepare configuration files for Azure Dev Spaces`. This command is not to be confused with the `azds prep` command, which configures the project for deployment.
1. Hit F5, and wait for the service to build and deploy. You'll know it's ready when the *Application started. Press Ctrl+C to shut down.* message appears in the debug console.
1. The endpoint URL will look something like http://localhost:\<portnumber\>. **Tip: The VS Code status bar will display a clickable URL.** It might seem like the container is running locally, but actually it is running in our dev space in Azure. The reason for the localhost address is because `mywebapi` has not defined any public endpoints and can only be accessed from within the Kubernetes instance. For your convenience, and to facilitate interacting with the private service from your local machine, Azure Dev Spaces creates a temporary SSH tunnel to the container running in Azure.
1. When `mywebapi` is ready, open your browser to the localhost address. Append `/api/values` to the URL to invoke the default GET API for the `ValuesController`.
1. If all the steps were successful, you should be able to see a response from the `mywebapi` service.

### Make a request from *webfrontend* to *mywebapi*
Let's now write code in `webfrontend` that makes a request to `mywebapi`.
1. Switch to the VS Code window for `webfrontend`.
1. *Replace* the code for the About method in `HomeController.cs`:

    ```csharp
    public async Task<IActionResult> About()
    {
        ViewData["Message"] = "Hello from webfrontend";
        
        using (var client = new System.Net.Http.HttpClient())
            {
                // Call *mywebapi*, and display its response in the page
                var request = new System.Net.Http.HttpRequestMessage();
                request.RequestUri = new Uri("http://mywebapi/api/values/1");
                if (this.Request.Headers.ContainsKey("azds-route-as"))
                {
                    // Propagate the dev space routing header
                    request.Headers.Add("azds-route-as", this.Request.Headers["azds-route-as"] as IEnumerable<string>);
                }
                var response = await client.SendAsync(request);
                ViewData["Message"] += " and " + await response.Content.ReadAsStringAsync();
            }

        return View();
    }
    ```

The preceding code example forwards the `azds-route-as` header from the incoming request to the outgoing request. You'll see later how this helps teams with [collaborative development](team-development-netcore.md).

### Debug across multiple services
1. At this point, `mywebapi` should still be running with the debugger attached. If it is not, hit F5 in the `mywebapi` project.
1. Set a breakpoint in the `Get(int id)` method that handles `api/values/{id}` GET requests.
1. In the `webfrontend` project, set a breakpoint just before it sends a GET request to `mywebapi/api/values`.
1. Hit F5 in the `webfrontend` project.
1. Invoke the web app, and step through code in both services.
1. In the web app, the About page will display a message concatenated by the two services: "Hello from webfrontend and Hello from mywebapi."

### Automatic tracing for HTTP messages
You may have noticed that, although *webfrontend* does not contain any special code to print out the HTTP call it makes to *mywebapi*, you can see HTTP traces messages in the output window:
```
// The request from your browser
default.webfrontend.856bb3af715744c6810b.eus.azds.io --gyk-> webfrontend:
   GET /Home/About HTTP/1.1

// *webfrontend* reaching out to *mywebapi*
webfrontend-668b7ddb9f-n5rhj --pu5-> mywebapi:
   GET /api/values/1 HTTP/1.1

// Response from *mywebapi*
webfrontend-668b7ddb9f-n5rhj <-pu5-- mywebapi:
   HTTP/1.1 200 OK
   Hello from mywebapi

// Response from *webfrontend* to your browser
default.webfrontend.856bb3af715744c6810b.eus.azds.io <-gyk-- webfrontend:
   HTTP/1.1 200 OK
   <!DOCTYPE html>
   <html>
   <head>
       <meta charset="utf-8" />
       <meta name="viewport" content="width=device-width, initial-sc...<[TRUNCATED]>
```
This is one of the "free" benefits you get from Dev Spaces instrumentation. We insert components that track HTTP requests as they go through the system to make it easier for you to track complex multi-service calls during development.


### Well done!
You now have a multi-container application where each container can be developed and deployed separately.


## Next steps

> [!div class="nextstepaction"]
> [Learn about team development in Dev Spaces](team-development-netcore.md)