# Part 2 Kubernetes Objects - ConfigMaps
## ConfigMap Examples
### This configMap definition stores two configuration values
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-configmap
data:
  APP_ENV: "production"
  LOG_LEVEL: "debug"
  config-file: |
    key1=value1
    key2=value2
```
### Deploy the configMap
```bash
# Deploy a configMap that will be used by the pod examples below
kubectl apply -f ./docs/ops-training/ops-training-manifests/configmap/00-cm.yaml
```
### Pod uses values from the configMap above and sets them as environment variables
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-var-example
spec:
  containers:
  - name: app
    image: busybox
    command: [ "sh", "-c", "echo APP_ENV=$APP_ENV && echo LOG_LEVEL=$LOG_LEVEL && sleep 3600" ]
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: my-configmap
          key: APP_ENV
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: my-configmap
          key: LOG_LEVEL
```
### Deploy pod with environment variables
```bash
# Example pod that sets environment variables from configMap
kubectl apply -f ./docs/ops-training/ops-training-manifests/configmap/01-po-env-var.yaml

# describe the Pod
k describe po env-var-example

# exec into Pod
k exec -it env-var-example -- /bin/sh

# look at environment variables
env

###################################
# Notice that APP_ENV=production
#####################################

```
### This Pod definition mounts the configMap as a volume
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-mount-example
spec:
  containers:
  - name: app
    image: busybox
    command: [ "sh", "-c", "cat /etc/config/config-file && sleep 3600" ]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: my-configmap
```
### Deploy pod with configMap mounted as a volume
```bash
# deploy
kubectl apply -f ./docs/ops-training/ops-training-manifests/configmap/02-po-volumemount.yaml

# exec into the container to look at the config-file that is mounted
k exec -it volume-mount-example -- cat /etc/config/config-file

# change the configMap value and look again
k edit cm my-configmap
# Add key3=value3

# Look at the config-file values again
k exec -it volume-mount-example -- cat /etc/config/config-file
```
### This Pod definition shows how you can use both approaches in the same pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: combined-example
spec:
  containers:
  - name: app
    image: busybox
    command: [ "sh", "-c", "echo APP_ENV=$APP_ENV && cat /etc/config/config-file && sleep 3600" ]
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: my-configmap
          key: APP_ENV
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: my-configmap
```
### Deploy pod def that uses both approaches
```bash
# deploy
kubectl apply -f ./docs/ops-training/ops-training-manifests/configmap/03-po-combine.yaml

# Describe the pod
k describe po combined-example

# exec into the po to show environment variables
k exec -it combined-example -- env

# exec into the pod to show the list contents of the volume mount
k exec -it combined-example -- ls /etc/config

# See how both pods have the same config key values
k exec -it combined-example -- cat /etc/config/config-file
```

### This pod definition shows how to pass command line arguments to the container
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-args-example
spec:
  containers:
  - name: app
    image: busybox
    command: [ "sh", "-c", "echo $1 $2 && sleep 3600" ]
    args:
    - "$(APP_ENV)"
    - "$(LOG_LEVEL)"
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: my-configmap
          key: APP_ENV
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: my-configmap
          key: LOG_LEVEL
```

### Deploy pod that passes command line arguments to the container
```bash
# deploy pod that passes cli arugment
kubectl apply -f ./docs/ops-training/ops-training-manifests/configmap/04-po-command-args.yaml

# describe the pod
k describe po command-args-example

# show the environment variables
k exec -it command-args-example -- env

```

## Clean up
```bash
kubectl delete -f ./docs/ops-training/ops-training-manifests/configmap/04-po-command-args.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/configmap/03-po-combine.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/configmap/02-po-volumemount.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/configmap/01-po-env-var.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/configmap/00-cm.yaml
```
