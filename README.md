This is an example of using multiple different services in devcontainers for local debugging and having them talk to each other. There are a number of things to consider to make this a pleasant development experience.  
The focus here is on Dotnet and Azure, but most of it as applicable to other languages.

**WORK IN PROGRESS**
*Right now stuck on how to get the CosmosDB linux emulator working without changing the IP address to the local machine* Because
- More than one container needs to access CosmosDB
- More than one developer will work on this and the idea is to minimize the manual work for each dev.

# Using this example
TBD...
In this solution, the "web" project is the main one. I have therefore put it's devcontainer.json and Dockerfile in the root in order to prompt the "re-open in dev containers". This will also start the other containers so they are nice and ready, both dependencies and other dev containers. 

To open another container, open the comand palette (F1 or CTRL-SHIFT-P) and "Open folder in container", then browse to the folder, for example the "api" folder. This will open the container instance that is (probably) already running.

# Using Dev Conainers with multiple services
This repository looks at using devcontainers in a more complex scenario.  
We often have a setup with multiple services that you may all want to run and debug simultaneously. For example, a SPA, an API and an Azure Function (or equivalent). The SPA may be intended to be deployed to a CDN, so it makes sense to have a dev server running that edits and compiles that. Separately you then need to run the back-end API - and then there is the Azure Function that might have to pick up queue messages etc.

In addition to all that, you may have shared dependencies, such as Azure Storage, SQL, Cosmos etc. 

## The simplest
You *can* simply do the "open folder in devcontainer" for each of your different services. That will work reasonably well, but there will be no connection between the services and you may have clashes between dependencies, such as Azurite.

## Using docker-compose
With docker-compose, you can connect the services together and they can share the same dependencies. By default, when you start any one devcontainer, they will all start, which is probably a good thing. There are also clever things you can do with networking etc, though you probably won't need that.

## Folders
This is quite an intricate subject and is quite important for your scenario.
Firstly, whether you use docker-compose or not, VS code looks for the root of the current git repository and mounts that in the container. This is important because it means all the source code in the whole repo is available, meaning any references to shared libraries continue to work. There are ways to override that, if you really want.

However, where VS code opens is different.  
When you are not using docker-compose, VS Code will open the folder you asked to open in VS Code. I.e. if you added a devcontainer to this repository's /src/func and didn't use docker-compose, VS Code would open /src/func. You cannot override this - the `workspaceFolder` property in devcontainer.json is ignored in this scenario. 
When you use docker-compose, however, VS Code by default opens the root of the repository (or it may be where docker-compose is, I haven't checked). However, when you use docker-compose you *can* change which folder VS Code ends up opening, by using the `workspaceFolder` in devcontainer.json. 

Why should you care? 
On the one hand, getting VS code to open the root of repository means you have access to all the code, including shared libraries and unit tests. 
On the other hand, it's *much* easier to get Run -> Debug to work in VS Code if you have opened just the folder containing the service you want to run. For an example, look at the "func" in this project; it has a .vscode folder with auto-generated tasks and launch settings that make it trivial to debug the function. If, instead, I opened the root of the repository, I would have to move those settings to the root of the repository and I would have to chose which service to run each time I wanted to debug. 

I don't, yet, know which way I want to go. I suspect that for all the secondary or relatively-isolated services, i.e. the Azure Function and the static website, I will have them open just their folder in VS code because it will make it super easy to start it up and debug. This may have an impact on where you put your unit tests.

## A "root" devcontainer?
There is usually a "main" project where you do most of your work, often the API or the web server. I think it makes sense to have that open the root in VS Code, so you can edit all the code in there easily. In this project, I have put that .devcontainer folder in the root of the repository so the user will be prompted to open it as soon as they open the repository in code. It feels messy, but... 

Note that the code folder is *mounted* into each container, meaning changes are instantly available in all containers. So, you can do `dotnet watch run` in a service container; when you edit a shared library in the "main" container, the service will detect that and recompile. 

# Ports
Things get a bit murky so pay close attention. 
Dev containers can do a lot of magic to detect which ports your application are using and automatically exposing them to the host machine. But, we want the services to be able to talk *to each other* so we need to go through some extra hoops.

Firstly, understand that (at least in my experience) ASP.Net runs on *localhost*. Anything running on *localhost* will [not be exposed by docker to the outside world](https://github.com/docker/compose/issues/4799#issuecomment-623504144) at all so will never work. Except - the "devcontainers magic" *will* work with localhost, so you may see that when you start your service from a devcontainer, you can reach it from your local machine, but not from the other containers and it stops working if you try to use `docker-compose` to map the ports. So, the first thing to do is to add something like this to your `program.cs`:
```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
            webBuilder.UseUrls("http://0.0.0.0:6001", "https://0.0.0.0:6002");
        });
```
`0.0.0.0` means "all network interfaces".  
(NOTE: I'd love for someone to tell me this is wrong and I don't need this in program.cs).

## Using docker-compose
You then need to add something like this to your docker-compose file:
```yaml
    api:
        build: ./src/api/.devcontainer
        volumes: 
            - .:/workspace:cached
        command: /bin/sh -c "while sleep 1000; do :; done"
        ports: 
            - 6001:6001
            - 6002:6002
```

With that in place, make sure you don't use `forwardPorts` or `appPort` in your devcontainer.json as that will only confuse matters.

## Calling the API
You can now connect to the API service from your host machine on http://localhost:6001 and from other containers on http://api:6001. "localhost" will *not* work from other containers.

## Port mapping
You *can* do things like mapping 6001 externally to 5000 internally. But, when you start debugging, VS code in the container will try open the browser pointing to 5000 so that will just be annoying. And if you do use TLS (please don't) then the redirect will redirect you to the port known internally. So, best to set up the ports directly in code, your life will be easier.

## PS
- You can play around with using EXPOSE instead of PORTS if you fancy, but I guess that mostly you will want to be able to call your services from your host machine as well, for testing.
- Port 6000 is blocked by Chrome so don't use that.

# Dev Cert
The dev cert is straightaway painful and - deeply ironically - requires local setup that might conflict between projects.

# Consider
- "The "shutdownAction":"none" in the devcontainer.json files will leave the containers running even if you close the two VS Code windows -- which prevents you from accidentally shutting down both containers (and thus both windows stop working) by closing one. This part is optional." (consider adding this)
- Can we do something like "dotnet watch run" when spinning up the api container? If so, will editing still work?

# References
[Get started videos](https://channel9.msdn.com/Series/Beginners-Series-to-Dev-Containers)
## Multi container references
Ref: https://github.com/microsoft/vscode-remote-release/issues/254
https://code.visualstudio.com/docs/remote/containers-advanced#_connecting-to-multiple-containers-at-once
https://blog.johnnyreilly.com/2021/05/15/azurite-and-table-storage-dev-container/

# Azure functions
Creating the dev container *first* allows you to use an image with the functions cli already installed.
For .Net 5 and Functions, there is no pre-built image, but there is this handy dockerfile: https://github.com/Azure/azure-functions-docker/blob/dev/host/3.0/buster/amd64/dotnet/dotnet-isolated/dotnet-isolated-core-tools.Dockerfile. However, it turns out the core tools also require netcoreapp 3.1 so I have added that to the Dockerfile. 

# Cosmos DB
The Linux emulator here is in preview and is limited. We are back to the joy of certificates. I have avodied that (so far) in this code by setting some dangerous options to use Gateway mode and not checking SSL certs. That's not ideal and you may need to do the more complex things with certificates and IP addresses :( For the IP address, you will probably want to use docker-compose's network abilities to lock a specific IP address to that container, at least for the intra-container traffic, rather than doing it differently on each dev machine.

## CosmosDB Healthcheck
Cosmos may take a little while to start up, so there is a healthcheck on cosmos in docker-compose which just uses `curl` to check if it is up. Services that rely on cosmos being up uses `depends_on` like this:
```yml
depends_on: 
    cosmos:
        condition: service_healthy
```
At the time of writing, `curl` is not installed on the emulator image, so I am using a Dockerfile to install it. I have kept everything else in docker-compose in the expectation that curl will be included in the emulator image in the future and thus allow the removal of the Dockerfile.cosmos.

## IP Address
I was advised that it is important to give the Cosmos container a static IP, so I have done that in docker-compose. I am not convinced this is strictly true, though - it may only be to do with certificates. To be tested.

## Certificates

## Performance
When I include the Cosmos image in my setup, my laptop fan comes on a fair bit and it does seem to consume a fair bit of memory and CPU, even when I am not talking to it at all. It is only in preview and your mileage may vary.

## Persistent storage
There are things you can do to map volumes and persist data etc - see the docs if that is relevant to you.


## Feedback to Azure team
Curl is not installed by default, making the healthcheck a tad more complex to set up. I have solved it with a dockerfile to install it, but it may be worth adding curl to the image.

I would *like* to access cosmos from my other containers using the docker-compose networking, i.e. "https://cosmos:8081/....". However, if I set AZURE_COSMOS_EMULATOR_IP_ADDRESS_OVERRIDE=cosmos then cosmos fails to start (the healtcheck reports "connection refused").
If I set AZURE_COSMOS_EMULATOR_IP_ADDRESS_OVERRIDE=172.18.0.100 then it comes up okay, but the certificate is still issued to "localhost" so it is unlikely to work from other containers?