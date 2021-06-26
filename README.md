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

# Use this example with devcontainers.
In this solution, the "web" project is the main one. I have therefore put it's devcontainer.json and Dockerfile in the root in order to prompt the "re-open in dev containers". This will also start the other containers so they are nice and ready, both dependencies and other dev containers. 

To open another container, open the comand palette (F1 or CTRL-SHIFT-P) and "Open folder in container", then browse to the folder, for example the "api" folder. This will open the container instance that is (probably) already running.

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


# Ports
If I add port mappings for the dev containers in docker-compose it doesn't work - so don't do that. Use the `forwardPorts` setting in devcontainer.json or just allow the automatic thing.

