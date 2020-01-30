# Creating a containerized ASP.NET Core / React web application

The fully-baked end product of this guide is available in the [src](/src) sibdirectory. Read the instructions below to build the application from scratch:

## Install prerequisite software

This guide assumes you're using either Ubuntu 18.04 or Windows 10.

For Windows users, make sure you have the following software installed:

1. Docker for Windows
1. Windows Subsystem for Linux (WSL) with Ubuntu 18.04
1. Visual Studio Code
1. C# extension for Visual Studio Code
1. Chrome JavaScript Debugger extension for Visual Studio Code

I strongly recommend following the setup guide for Windows Subsystem for Linux at https://github.com/erik1066/windows-wsl-ubuntu-setup. The guide includes a [script](https://github.com/erik1066/windows-wsl-ubuntu-setup/blob/master/scripts.sh) that will install all the necessary software in Ubuntu, such as the .NET Core SDK, NodeJS, and Docker Compose.

## Create the React/.NET Core project

Once WSL is installed and you have a Bash terminal open, navigate to a folder of your choice under which the ASP.NET Core web service will reside. For example:

```bash
cd /mnt/c/Users/yourusername/source
```

> `/mnt/c/` in WSL points to the C: drive on the Windows file system

Once you're in the desired folder, run the following commands:

```bash
mkdir dotnet-react-example
cd dotnet-react-example
dotnet new react
```

The `dotnet new react` line uses .NET Core's built-in React template. Unfortunately, this template seems perpetually out-of-date. Let's fix this:

```bash
rm -rf ClientApp
npx create-react-app clientapp
```

> To use TypeScript instead of JavaScript, instead run `npx create-react-app clientapp --template typescript`.

Lower-casing is important for `clientapp` on the 2nd line since `npx` disallows anything but lower-case project names. Of course, .NET Core expects the pascal-cased `ClientApp`. We can fix this with two `mv` commands:

```bash
mv clientapp clientapp2
mv clientapp2 ClientApp
```

You may alternatively keep `clientapp` lower-cased and instead make the following source-code level changes in the .NET Core codebase:

1. Open `dotnet-react-example.csproj` and change `<SpaRoot>ClientApp</SpaRoot>`
1. Open `Startup.cs` and change `configuration.RootPath = "ClientApp/build";`
1. Open `Startup.cs` and change `spa.Options.SourcePath = "ClientApp";`

You can find-and-replace in Bash like this:

```bash
find ./Startup.cs -type f -exec sed -i 's/ClientApp/clientapp/g' {} \;
find ./dotnet-react-example.csproj -type f -exec sed -i 's/ClientApp/clientapp/g' {} \;
```

The `dotnet build` command is normally used to build .NET Core projects. However, we're building both the ASP.NET Core backend and the React front-end. The .NET Core `react` template incorporates `npm` commands into the `.csproj` file so that the React project can be built alongside the .NET Core code, but this only works when you run `dotnet publish`. Let's run this now:

```bash
dotnet publish
```

Now let's run the thing:

```bash
dotnet run
```

Visit `https://localhost:5001/` and you should see the React app running successfully.

## Add a Dockerfile

Add a Dockerfile to the root folder:

```bash
touch Dockerfile
```

We'll use a two-stage build. The first stage builds the image with the .NET Core SDK, while the second stage will contain only the .NET Core runtime and is what will be executed by Docker. This keeps the image size very small. Of course, we need to include NPM in the first stage, which adds a bit of complexity.

The Dockerfile ought to look like this:

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/core/sdk:3.1-alpine as build

# installs NodeJS and NPM
RUN apt-get update -yq && apt-get upgrade -yq && apt-get install -yq curl git nano
RUN curl -sL https://deb.nodesource.com/setup_12.x | bash - && apt-get install -yq nodejs build-essential

# copy the files from the file system so they can built
COPY ./ /src
WORKDIR /src

# install node
RUN npm install -g npm
RUN npm --version

# Opt out of .NET Core's telemetry collection
ENV DOTNET_CLI_TELEMETRY_OPTOUT 1

# set node to production
ENV NODE_ENV production

# run the publish command, which also runs the required NPM commands to build the React front-end
RUN dotnet publish -c Release

# Run stage
FROM mcr.microsoft.com/dotnet/core-nightly/aspnet:3.1-alpine as run

ENV DOTNET_CLI_TELEMETRY_OPTOUT 1

RUN apk update && apk upgrade --no-cache

EXPOSE 9015/tcp
ENV ASPNETCORE_URLS http://*:9015

ARG MONGO_CONNECTION_STRING
ARG MONGO_USE_SSL

ENV MONGO_CONNECTION_STRING ${MONGO_CONNECTION_STRING}
ENV MONGO_USE_SSL ${MONGO_USE_SSL}

COPY --from=build /src/bin/Release/netcoreapp3.1/publish /app
WORKDIR /app

RUN mkdir -p /ASP.NET/DataProtection-Keys
RUN chown -R 1001:0 /ASP.NET/DataProtection-Keys

# don't run as root user - this is a good security practice, and also required for running this container in OpenShift
RUN chown 1001:0 dotnet-react-example.dll
RUN chmod g+rwx dotnet-react-example.dll
USER 1001

ENTRYPOINT ["dotnet", "dotnet-react-example.dll"]
```

You should now be able to build the Docker image:

```bash
docker build \
    -t dotnet-react-example \
    --rm \
    --force-rm \
.
```

Notice we've specified two environment variables: `MONGO_CONNECTION_STRING` and `MONGO_USE_SSL`. These can be supplied as part of the `docker build` command, but preferably will be used when running the image.

To run the container with the two environment variables set:

```bash
docker run -d \
    -p 9015:9015 \
    -e MONGO_CONNECTION_STRING=mongodb://mongo:27017 \
    -e MONGO_USE_SSL=false \
    dotnet-react-example
```

Check that your container is running by visiting `http://localhost:9015`.

You can stop the container like such, changing `container_id` with the ID returned from the `docker ps` command:

```bash
docker kill container_id
```

## Getting environment variables in ASP.NET Core

Now that these environment variables have been set in the container image, how do we access them from .NET Core?

In `Startup.cs`, these variables are accessed via the `Configuration` object, which is defined as a publicly-accessible property:

```cs
public IConfiguration Configuration { get; }
```

Thus, from anywhere in the Startup class, you can do something like this:

```cs
string conn = Configuration["MONGO_CONNECTION_STRING"];
```

Observe that all items you can return will be of type `string`. In the case of `MONGO_USE_SSL`, you should convert the string value to a `bool`.

The `Configuration` object isn't visible outside of `Startup.cs` and that's generally okay, as `Startup.cs` is where dependency injection (DI) is generally done and thus where the ENV vars would be used.

To see an example of .NET Core dependency injection in action with Mongo, where the Mongo connection string is passed into the container via environment variables, see the [ASP.NET Core Mongo CRUD microservice](https://github.com/erik1066/fdns-ms-dotnet-object) on GitHub.

## Debugging in Visual Studio Code

Create a `.vscode` folder:

```bash
mkdir .vscode
```

We're now going to create a compound launch file that runs two debuggers: One for React that runs in Chrome and one for .NET Core.

Add a `launch.json` file like such:

```json
{
  "version": "0.2.0",
  "compounds": [
    {
      "name": "[localhost] ASP.NET Core and Chromium",
      "configurations": [
        "[localhost] .NET Core Launch (web)",
        "Launch Chromium"
      ]
    }
  ],
  "configurations": [
    {
      "name": "[localhost] .NET Core Launch (web)",
      "type": "coreclr",
      "request": "launch",
      "preLaunchTask": "build",
      "program": "${workspaceFolder}/bin/Debug/netcoreapp3.1/dotnet-react-example.dll",
      "args": [],
      "cwd": "${workspaceFolder}",
      "stopAtEntry": false,
      "internalConsoleOptions": "openOnSessionStart",
      "launchBrowser": {
        "enabled": false, // normally true, but set to 'false' for compound debugging
        "args": "${auto-detect-url}",
        "webRoot": "${workspaceFolder}",
        "windows": {
          "command": "cmd.exe",
          "args": "/C start ${auto-detect-url}"
        },
        "osx": {
          "command": "open"
        },
        "linux": {
          // "command": "xdg-open"
          "command": "/usr/bin/chromium-browser" // for Linux
        }
      }
    },
    {
      "name": "Launch Chromium",
      "type": "chrome",
      "request": "launch",
      "url": "http://localhost:5000",
      "webRoot": "${workspaceRoot}",
      "runtimeExecutable": "/usr/bin/chromium-browser" // for Linux
    },
    {
      "name": ".NET Core Attach",
      "type": "coreclr",
      "request": "attach",
      "processId": "${command:pickProcess}"
    }
  ]
}
```

Note that on Windows 10, you may need to change the `runtimeExecutable": "/usr/bin/chromium-browser"` line to point to the location of Chrome. The location specified above is specific to Chromium on Ubuntu 18.04.

Next, add a `tasks.json` file:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "build",
      "command": "dotnet",
      "type": "process",
      "args": ["build", "${workspaceFolder}/dotnet-react-example.csproj"],
      "problemMatcher": "$msCompile"
    }
  ]
}
```

In Visual Studio Code, in the **Debugging** pane, select the `[localhost] ASP.NET Core and Chromium` option from the debug mode selector and press the green **Debug** button. Two debuggers will launch: One for the .NET Core backend and one for the React frontend in Chrome. (Be sure you've installed the JavaScript Debugger for Chrome extension in Visual Studio Code.)
