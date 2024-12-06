# Explore the container
## Look at the layers of the container image
1. Open Docker Desktop
2. Select images in the menu
3. Select an image and review the image layers

## run a command in the container
```bash
# determine the container ID
docker ps

# ls in the /var directory
docker exec -it <containerID> ls /var
```
## Copy a file into a container
```bash
# determine the container ID
docker ps

# create a file
echo "Test file to copy to docker container" > test.txt

# copy the file to the running container
docker cp test.txt <containerID>:/app/test.txt

# Open the text file
docker exec -it <containerID> cat test.txt

```
## Step inside a running container
```bash
# determine the container ID
docker ps

# exec into the container
# in this example we're using bash, but some containers may not have bash. You may need to use /bin/sh
docker exec -it <containerID> /bin/bash

# Look at installed packages
dpkg -l

# look at the directory
ls

# Look at OS running in container
cat /etc/os-release
```
## Show the container is isolated and is 
```bash
# Steps while you are already stepped inside the container.. from previous step
# attempt to access the host filesystem
ls /host/path

```
