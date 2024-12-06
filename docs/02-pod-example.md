# Part 2 Kubernetes Objects - Pods
## Create a pod with a single container
#### Single container in a pod - YAML
```yaml
# this manifest creates a pod with a single Nginx container and exposes the pod on port 80
apiVersion: v1
kind: Pod
metadata:
  name: single-container-pod
  labels:
    purpose: demonstrate-single-container
spec:
  containers:
  - name: example-container
    image: nginx:latest
    ports:
    - containerPort: 80
```
```bash
# Create a pod with one container using the manifest referenced below 
# From the Dev cluster
kubectl apply -f ./docs/ops-training/ops-training-manifests/training-session-2/pod-single-container.yaml
```
#### Review a pod with a single container
```bash
# Get running pods
kubectl get po

# Get running pods and show IP address and node hosting the pod
kubectl get po -o wide

# Note: the IP address is from the pod network

# Describe the pod
k describe po single-container-pod

######################################################################################################################################
# Notice the node value shows the host and IP address of the node
#
# Notice the labels. Labels are used heavily in Kubernetes
#
# Notice the container section of the configuration
## Image name. If there is no FQDN the image came from Docker Hub. In this example there is no version name
## this means that we're pulling the latest image from Docker Hub
#
# Notice the Events section. We can see the life cycle of the Pod. If there are issues with the Pod, this is an excellent place to look
#######################################################################################################################################

# Show the manifest of the single pod
k get po single-container-pod -o yaml

```
## Create a pod with two containers
#### Two containers in a pod yaml
```yaml
# create a pod with two containers
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
  labels:
    purpose: demonstrate-multi-container
spec:
  containers:
  - name: example-nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
  - name: example-mysql-container
    image: mysql:5.7
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "yourpassword"
```

#### Create two containers in a pod
```bash
# Create a pod with two containers using the manifest referenced below 
# From the Dev cluster
kubectl apply -f ./docs/ops-training/ops-training-manifests/training-session-2/pod-multi-container.yaml
```
#### Review the example pod with two containers
```bash
# Get running pods and show IP address and node hosting the pod
kubectl get po -o wide

# Notice that multi-container-pod shows a ready state of 2/2. This means there are two ready containers in the pod
# Notice that the multi-container pod has a single IP address

# Describe the pod
k describe po multi-container-pod 

###########################################################################################################################################
# Notice the container section of the configuration has two containers: example-nginx-container and example-mysql-container
# Notice that the mysql container has no port. This container can only be accessed via TCP within the Pod
# Notice that Volumes is outside of the container section of the config. All containers in the pod share the volumes that are mounted
############################################################################################################################################

## Image name. If there is no FQDN the image came from Docker Hub. In this example there is no version name
## this means that we're pulling the latest image from Docker Hub

# Notice the Events section is combined

# Show the manifest of the multi-container pod
k get po multi-container-pod -o yaml
```
#### Step into a running container
```bash
# step into the single container pod you just created. Notice the message about the default container
kubectl exec -it multi-container-pod -- sh

# Since the multi-cluster pod example has multiple running containers, you must specify the container you want to exec into a specific container that is not default (the first container in the manifest)
kubectl exec -it multi-container-pod -c example-nginx-container -- sh
ls
exit
kubectl exec -it multi-container-pod -c example-mysql-container -- sh
ls
exit
```

#### Clean up the pods that were created
```bash
kubectl delete -f ./docs/ops-training/ops-training-manifests/training-session-2/pod-single-container.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/training-session-2/pod-multi-container.yaml
```



