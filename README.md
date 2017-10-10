# Start container
> docker start 2db

# List running containers
> docker ps

# List all containers
> docker ps --all

# List all Images
> docker image ls

# Build Image from Dockerfile
> docker build -t dtarczynski/simpleapi .

# Show container logs. When something goes wrong.
> docker logs 6cf52a3b570b

# Stop all running containers
> docker stop $(docker ps -a -q)

# Download an image
> docker pull image_name

# Delete dangling images
> docker rmi $(docker images -q -f dangling=true)

or

> docker image prune

# Docker Cheat Sheet
http://files.zeroturnaround.com/pdf/zt_docker_cheat_sheet.pdf
