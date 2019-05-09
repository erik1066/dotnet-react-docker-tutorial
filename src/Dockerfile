# Build stage
FROM mcr.microsoft.com/dotnet/core/sdk:2.2.203 as build

# installs NodeJS and NPM
RUN apt-get update -yq && apt-get upgrade -yq && apt-get install -yq curl git nano
RUN curl -sL https://deb.nodesource.com/setup_10.x | bash - && apt-get install -yq nodejs build-essential

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
FROM mcr.microsoft.com/dotnet/core/aspnet:2.2.4-alpine3.9 as run

ENV DOTNET_CLI_TELEMETRY_OPTOUT 1

RUN apk update && apk upgrade --no-cache

EXPOSE 9015/tcp
ENV ASPNETCORE_URLS http://*:9015

ARG MONGO_CONNECTION_STRING
ARG MONGO_USE_SSL

ENV MONGO_CONNECTION_STRING ${MONGO_CONNECTION_STRING}
ENV MONGO_USE_SSL ${MONGO_USE_SSL}

COPY --from=build /src/bin/Release/netcoreapp2.2/publish /app
WORKDIR /app

RUN mkdir -p /ASP.NET/DataProtection-Keys
RUN chown -R 1001:0 /ASP.NET/DataProtection-Keys

# don't run as root user - this is a good security practice, and also required for running this container in OpenShift
RUN chown 1001:0 dotnet-react-example.dll
RUN chmod g+rwx dotnet-react-example.dll
USER 1001

ENTRYPOINT ["dotnet", "dotnet-react-example.dll"]