# Docker How-To
## 1. Create Dockerfile
First create your Docker file that describes *Environment* required in order to run your Application

``` 
FROM microsoft/aspnetcore:2.0 #This is ASP.NET CORE RUNTIME (not SDK)
ARG source
WORKDIR /app
#EXPOSE 8888
COPY bin/Debug/netcoreapp2.0/publish .
ENTRYPOINT ["dotnet", "Codmatic.Api.dll"]
```

## 2. Create an container Image from Dockerfile
> docker build -t dtarczynski/simpleapi:latest .

## 3. List all Images
> docker image ls

## 4. Run the Container
> docker run --rm -it -p 8989:80 dtarczynski/simpleapi

## Handy commands

### Start container
> docker start 2db

### List running containers
> docker ps

### List all containers
> docker ps --all

### Show container logs. When something goes wrong.
> docker logs 6cf52a3b570b

### Stop all running containers
> docker stop $(docker ps -a -q)

#### Download an image
> docker pull image_name

### Delete dangling images
> docker rmi $(docker images -q -f dangling=true)

or

> docker image prune

## Scenario1. You want to same changes made to running container
> docker commit container_id hubdoc.azurecr.io/myapp/database:latest

### Docker Cheat Sheet
http://files.zeroturnaround.com/pdf/zt_docker_cheat_sheet.pdf
