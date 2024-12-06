# Part 2 Kubernetes Objects - Namespaces
## 
```bash
# create a namespace
kubectl create namespace test

# deploy the single container pod into the test namespace
kubectl apply -f ./docs/ops-training/ops-training-manifests/training-session-2/pod-single-container.yaml --namespace test

# look at the recently created pod
kubectl get po single-container-pod
kubectl get po

# you can't see the pod because it's not in the default namespace. Look for the pod in the test namespace
kubectl get po single-container-pod --namespace test

# try with the namespace short name
kubectl get po single-container-pod -n test

# find the IP address of the newly created pod
kubectl get po single-container-pod -n test -o wide

###########################################################################################################
# NOTE: basic discussion of services - Services will be discussed in greatly detail in Part 3: Networking
###########################################################################################################

# create a pod in the default namespace, exec into the pod and attempt to resolve the name of a pod in the test namespace
kubectl apply -f ./docs/ops-training/ops-training-manifests/training-session-2/pod-multi-container.yaml
kubectl exec -it multi-container-pod -c example-nginx-container -- sh

# install ping and attempt to ping the pod in the other namespace
apt-get update
apt-get install iputils-ping

# ping something
ping 10.10.117.27

# ping the IP address of the single container pod in the test namespace
ping <IP address of the single container pod in the test namespace>

# ping by name. what happens?
ping single-container-pod

# ping by FQDN. what happens?
ping single-container-pod.test.svc.cluster.local

# exit from the pod
exit

####################################################################
# NOTE: by default, you can ping a pod by IP address as long as it exists on the same cluster and you're
# pinging from within that cluster. However, you can't access a pod by name without a service.
# We'll talk much more about services in the first networking session
####################################################################
```

## Clean up
```bash
# delete the pod created previously
kubectl delete -f ./docs/ops-training/ops-training-manifests/training-session-2/pod-multi-container.yaml --namespace test

# delete the test namespace
k delete namespace test
```
