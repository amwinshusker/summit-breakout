# Part 4: Persistent Volumes, Persistent Volume Claims, and Storage Classes - Azure Disk CSI examples
## Storage Class YAML example
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-fast
provisioner: disk.csi.azure.com
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
parameters:
  skuName: Premium_LRS

```
## Storage Class Creation demo
### Storage Class SSD disks
```bash
# create the Azure fast storage class - it uses SDD storage
kubectl apply -f ./docs/ops-training/ops-training-manifests/storage/sc-azure-fast.yaml

# Look at the storage classes
k get sc

# Look at the azure-fast and azure-slow storage classes in more detail
k describe sc azure-fast

```
### Storage Class HDD disks
```bash
# create the Azure slow storage class - it uses HDD storage
kubectl apply -f ./docs/ops-training/ops-training-manifests/storage/sc-azure-slow.yaml

# Look at the storage classes
k get sc

# Look at the azure-fast and azure-slow storage classes in more detail
k describe sc azure-slow
```
## Persistent Volume Claim (PVC) YAML example
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: azure-fast

```
## Persistent Volume Claim (PVC) creation demo
```bash
# Create PVC that references SC SDD class
kubectl apply -f ./docs/ops-training/ops-training-manifests/storage/pvc-azure-fast.yaml

# Look at the PVC that was created. Remember PVCs are namespace scoped
k get pvc # notice that fast-pvc is unbound, and associated with the azure-fast storage class

# Look closer at PVC
k describe pvc fast-pvc

```
## Deployment using PVC example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1-container
        image: nginx:latest
        volumeMounts:
        - mountPath: /mnt/data
          name: storage
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: fast-pvc

```
## Deployment Creation using PVC demo
Creating two deployments that will share the same PV. Write to the volume
### Deployment of app1
```bash
# Create Deployment using SDD PVC
kubectl apply -f ./docs/ops-training/ops-training-manifests/storage/deploy-azure-fast-app1.yaml
```
### Deployment of app2
```bash
# Create 2md Deployment using SDD PVC
kubectl apply -f ./docs/ops-training/ops-training-manifests/storage/deploy-azure-fast-app2.yaml
```
### Evaluate the environment
```bash
# look at the PVC now
k get pvc fast-pvc # notice it now shows as bound

# look at PVs - remember PVs are namespace scoped
k get pv # look for the claim named default/fast-pvc... there is only one

# look at the pod deployed from app 1
# first look at all pods in the default namesapce
k get po
k describe po app1<rest of the deployment name>

# Exec into the app 1 pod
k exec -it app1-<deployment - name> -- /bin/sh

# Change directory to the PV that has been created
cd /mnt/data

# create a file on the disk
touch test-from-app1.txt

# Exec into the app 2 pod
k exec -it app2-<deployment - name> -- /bin/sh

# Change directory to the PV that has been created
cd /mnt/data

# create a file on the disk
ls -la # notes that test-from-app1.txt file is available to the second pod... same PV is mounted

```
### NOTE: creating a Azure Disk type that shares a PV between pods isn't something we would typically do. The purpose of this is to show flexibility of K8s
### Typically we would create a PV per persistent pod. Azure File share would be a better use case to share a PV across multiple pods and even clusters

## Delete app1 and app2 deployments
Determine what happens to the PV
```bash
kubectl delete deployment app1 app2
```
## Create a new Deployment using the same PV
```bash
# Create 3rd Deployment and leverage the PV that was in use previously
kubectl apply -f ./docs/ops-training/ops-training-manifests/storage/deploy-azure-fast-app3.yaml
```
What happed to the disk?
```bash
# Look at your current pods
k get po
# exec into the new pod
k exec -it app3<deploy name> -- /bin/sh

# Change directory to the PV that has been created
cd /mnt/data

# Look for the file created previously
ls -la # it's there... the disk remains and it has not been wiped clean
```

## Let's test disk latency between Azure SSD disks and Azure HDD disks
We'll create two deployments, two services, we'll install fio in both pods, and we'll present the results via an html page
```bash
# Create a new PVC for azure slow - HDD disks
kubectl apply -f ./docs/ops-training/ops-training-manifests/storage/pvc-azure-slow.yaml

# Create a new deployment for slow HDD
kubectl apply -f ./docs/ops-training/ops-training-manifests/storage/deploy-azure-slow-app4.yaml

# Install disk latency tools on each VM
## Exc into Azure slow
k exec -it slow-disk-test<deploy name> -- /bin/sh

# install tools
apt update
apt install -y fio

# Run a benchmark test on SDD disk
mkdir -p /var/www/html
fio --name=write-test --ioengine=libaio --rw=write --bs=4k --direct=1 --size=512M --numjobs=1 --runtime=10 --group_reporting --filename=/mnt/data/testfile > /var/www/html/results.html
fio --name=read-test --ioengine=libaio --rw=read --bs=4k --direct=1 --size=512M --numjobs=1 --runtime=10 --group_reporting --filename=/mnt/data/testfile >> /var/www/html/results.html


## Exec into Azure fast
k exec -it app3<deploy name> -- /bin/sh

# install tools
apt update
apt install -y fio

# Run a benchmark test on HDD disk
kubectl exec -it <slow-disk-test-pod-name> -- bash
mkdir -p /var/www/html
fio --name=write-test --ioengine=libaio --rw=write --bs=4k --direct=1 --size=512M --numjobs=1 --runtime=10 --group_reporting --filename=/mnt/data/testfile > /var/www/html/results.html
fio --name=read-test --ioengine=libaio --rw=read --bs=4k --direct=1 --size=512M --numjobs=1 --runtime=10 --group_reporting --filename=/mnt/data/testfile >> /var/www/html/results.html

```
## Compare the results
### Create 2 services of type loadbalancer to access each pod
```bash
###################################################################
# Compare
##################################################################
# Copy the names of the Pod and use in the exec commands
k get po

kubectl exec -it <app4 pod-name> -- cat /var/www/html/results.html

kubectl exec -it <app4 pod-name> -- cat /var/www/html/results.html

###############################################################################################################
### Disregard the service creation section... this was an attempt to display the results via HTML
#############################################################################################################
# Create a service for app4
# kubectl apply -f ./docs/ops-training/ops-training-manifests/storage/svc-lb-app4.yaml

# Create a service of type load balancer for slow-disk-test
# kubectl apply -f ./docs/ops-training/ops-training-manifests/storage/svc-lb-slow-disk.yaml
###############################################################################

## Browse to the results
From a client machine browse to the web service of each pod through the LB
```bash
# determine the IP address of each LB service
k get svc -o wide

http://<ip address of app3 service>/results.html
http://<ip address of slow-disk-test>/results.html
```
## Clean-up
```bash
# delete
kubectl delete -f ./docs/ops-training/ops-training-manifests/storage/deploy-azure-fast-app1.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/storage/deploy-azure-fast-app2.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/storage/deploy-azure-fast-app3.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/storage/deploy-azure-slow-app4.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/storage/pvc-azure-fast.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/storage/pvc-azure-slow.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/storage/sc-azure-fast.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/storage/sc-azure-slow.yaml

# Look for any PVs that remain that are associated with a storage class
```
