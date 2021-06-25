# Dev Cert
The dev cert is straightaway painful and - deeply ironically - requires local setup that might conflict between projects.

# Opening the solution in docker containers.
In this solution, the "web" project is the main one. I have therefore put it's devcontainer.json and Dockerfile in the root in order to prompt the "re-open in dev containers". This will also start the other containers so they are nice and ready, both dependencies and other dev containers. 

To open another container, open the comand palette (F1 or CTRL-SHIFT-P) and "Open folder in container", then browse to the folder, for example the "api" folder. This will open the container instance that is (probably) already running.

# Consider
- The "shutdownAction":"none" in the devcontainer.json files will leave the containers running even if you close the two VS Code windows -- which prevents you from accidentally shutting down both containers (and thus both windows stop working) by closing one. This part is optional. (consider adding this)
- Can we do something like "dotnet watch run" when spinning up the api container? If so, will editing still work?

# References
[Get started videos](https://channel9.msdn.com/Series/Beginners-Series-to-Dev-Containers)
## Multi container references
Ref: https://github.com/microsoft/vscode-remote-release/issues/254
https://code.visualstudio.com/docs/remote/containers-advanced#_connecting-to-multiple-containers-at-once
https://blog.johnnyreilly.com/2021/05/15/azurite-and-table-storage-dev-container/

# Azure functions
Creating the dev container *first* allows you to use an image with the functions cli already installed.
For .Net 5 and Functions, there is no pre-built image, but there is this handy dockerfile: https://github.com/Azure/azure-functions-docker/blob/dev/host/3.0/buster/amd64/dotnet/dotnet-isolated/dotnet-isolated-core-tools.Dockerfile


# Ports
If I add port mappings for the dev containers in docker-compose it doesn't work - so don't do that. Use the `forwardPorts` setting in devcontainer.json or just allow the automatic thing.

# Folders
By default, devcontainers goes up to the root of the git repository and maps that as the "workspace". This is referenced in docker-compose as a volume. This is usually what you want, especially if you are referencing shared dependencies. 
Each devcontainer.json file has a `workspaceFolder` property. This controls what folder vs code will open. In this example, I have set it to open at the root of the workspace, so all the dev containers can access all the source. That may not be right for you. If you have services that are truly separate then you may want to organise all your code for that in a folder and just open that. Incidentally, `workspaceFolder` only seems to work when combined with a docker-compose file.
*scrap that - workspaceFolder appears to always be ignored. when using docker compose, vs code always opens the root*