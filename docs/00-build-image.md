# Demostrate creating a Docker image

## Prerequisites
I'm using Docker Desktop to build the container below. Make sure that Docker Desktop is running if following along with Docker Desktop.

## Docker Desktop
Open Docker Desktop

Look at "Containers" in the Docker Desktop side menu. Notice that there are no containers listed

Look at "Images" and notice no image named demo1

## Create a new .net 8 web app
```bash
# Open VSCode
# Open a terminal (view > terminal
# CD to it-summit-demos
# Create a directory named demo1
mkdir demo1
# Open a terminal in VSCode and create a demo application
dotnet new web -n demo1
cd demo1
```
## Modify the application
Create a file named Dockerfile in the same directory as Program.cs
```cs
// modify the Program.cs file

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.Urls.Add("http://0.0.0.0:80");

app.MapGet("/", (context) => 
{
    context.Response.ContentType = "text/html";
    return context.Response.WriteAsync(@"
    <!DOCTYPE html>
    <html lang='en'>
    <head>
        <meta charset='UTF-8'>
        <meta name='viewport' content='width=device-width, initial-scale=1.0'>
        <title>Demo1</title>
        <style>
            body {
                background-color: blue;
                color: white;
                font-size: 48px;
                display: flex;
                justify-content: center;
                align-items: center;
                height: 100vh;
                margin: 0;
                font-family: Arial, sans-serif;
            }
        </style>
    </head>
    <body>
        .NET 8 application version 1
    </body>
    </html>");
});

app.Run();
```
Save the Program.cs file

## Create a Dockerfile
```Dockerfile
# Use the official .NET 8 SDK image for building the application
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80

# Use the SDK image for building the app
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /app

# Final stage: runtime image
FROM base AS final
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "demo1.dll"]
```
## Build the Docker image
```bash
# Build the docker image
docker build -t demo1:1.0 .
```
Notice that there is a new container Image named demo1
## Run the container
```bash
#  run the Docker image
docker run -p 8106:80 --name demo1 demo1:1.0
```
Notice that there is a new running container named demo1

Open the running container with Docker Desktop and look at the container logs, use exec, and look at the folder structure 

Open the container image and notice the layers

## Browse to the running container
Open a browser and go to http://localhost:8106

## Cleanup
delete the running container

delete the container image

#### delete the .net app
File > close
Delete the directory
