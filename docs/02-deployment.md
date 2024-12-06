# Part 2 Kubernetes Objects - Deployment
## Create a deployment with a single pod
#### Deployment - YAML

```yaml
# deploy a single Nginx container pod with 3 replicas exposed on port 80
# in this example we are not specifying the latest version of the nginx pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1       # At most, 1 pod can be unavailable during the update.
      maxSurge: 2             # Up to 2 extra pods can be created during the update.
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

#### Create a deployment
```bash
# create a deployment
kubectl apply -f ./docs/ops-training/ops-training-manifests/training-session-2/deploy-single-container-pod.yaml

# Look at the pods created by the deployment
kubectl get po

# Look at the replica set created by the deployment
kubectl get rs

# delete a pod created by the deployment
kubectl delete po <a pod created by the deployment>

# look at the pods again. Notice that the pod has been re-created
kubectl get po

# delete the replicaset
kubectl delete rs

# Notice that the pods and replicasets are recreated

# Look at the deployments
kubectl get deploy

# Show the containers in the deployment, the images and the selector
kubectl get deploy -o wide

# Describe the deployment
k describe deploy nginx-deployment

# Notice the RollingUpdateStrategy

# delete the deployment
kubectl delete -f ./docs/ops-training/ops-training-manifests/training-session-2/deploy-single-container-pod.yaml

# look at the pods. What happened to them?
kubectl get po

```
