# Part 2 Kubernetes Objects - Replicasets
## Review a configuration file for a replicaset

```yaml
# Create a replicaset for
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: multi-container-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: multi-container-app
  template:
    metadata:
      labels:
        app: multi-container-app
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
          value: "yourpassword"  # Replace with your actual password

```
## Create a replicaset with 3 replicas
```bash
# create a replicaset with 3 replicas of a multi-container pod
kubectl apply -f ./docs/ops-training/ops-training-manifests/training-session-2/rs-multi-container.yaml

# look at the replicaset
kubectl get replicaset multi-container-replicaset

# describe the replicaset
kubectl describe rs multi-container-replicaset

# Notice the selector references the label of the pod: app=multi-container-app
# Notice that there are 3 replicas

# change the number of replicas to 4
kubectl edit rs multi-container-replicaset

# how many pods do you have now?
kubectl get replicaset multi-container-replicaset

# delete one of the pods created by the replicaset
kubectl get po
## copy the name of one of the pods created by the replica set and paste the value below
kubectl delete po <name of pod you copied above>

```
## Clean up
```bash
# delete the previously created replicaset. Notice that even though you modified the replicaset you can still delete it based on the replicaset name in the manifest.
kubectl delete -f ./docs/ops-training/ops-training-manifests/training-session-2/rs-multi-container.yaml

```
