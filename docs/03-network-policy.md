# Part 3 Network Policies
Example of a network policy configuration that denies all egress traffic for pods based on label
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-all-egress
  namespace: test-network-policy
spec:
  podSelector:
    matchLabels:
      app: restricted-app # Target pods with this label
  policyTypes:
    - Egress
  egress: [] # Deny all egress traffic
```
## Network policy demonstration 1
### Deny all egress traffic for pods based on label
Deny all egress traffic for pods in a namespace based on label
```bash
# Create a namespace named test-network-policy
kubectl create ns test-network-policy

# Create a network policy that blocks egress based on label
kubectl apply -f ./docs/ops-training/ops-training-manifests/networking/network-policy-block-egress.yaml -n test-network-policy

# Create test pod in the test-network-policy namespace
kubectl apply -f ./docs/ops-training/ops-training-manifests/training-session-2/pod-single-container.yaml -n test-network-policy

# Check if pod-single-container can ping 10.10.117.27
k -n test-network-policy exec -it single-container-pod -- /bin/sh
# Install iputils
apt-get update
apt-get install iputils-ping

ping 10.10.117.27
exit

# Look at the network policy
k get networkpolicy -n test-network-policy

# Look at the Spec secion. The spec defines what happens.

# Associate the pod-single-container with the network policy
k edit po single-container-pod -n test-network-policy

# Add the restricted app label
#   labels:
#    purpose: demonstrate-single-container
#    app: restricted-app
##############################

# check if the pod is still able to ping 10.10.117.27
k -n test-network-policy exec -it single-container-pod -- /bin/sh
ping 10.10.117.27
```
Clean up example 1 (leave the namespace - we'll delete that at the end)
```bash
# clean up
kubectl delete -f ./docs/ops-training/ops-training-manifests/networking/network-policy-block-egress.yaml -n test-network-policy
kubectl delete -f ./docs/ops-training/ops-training-manifests/training-session-2/pod-single-container.yaml -n test-network-policy

```
### Deny traffic between specific pods
This example policy denies traffic between pods with the client label and pods with the server label
```yaml
# This policy denies traffic between pods with the client label and pods with the server label

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-client-to-server
  namespace: test-network-policy
spec:
  podSelector:
    matchLabels:
      app: server
  ingress:
    # Allow traffic from all pods except the 'client' pod
    - from:
        - podSelector:
            matchExpressions:
              - key: app
                operator: NotIn
                values:
                  - client
      ports:
        - protocol: TCP
          port: 80

```
## Network policy demonstration 2
This policy denies traffic between pods with the client label and pods with the server label
```bash

# Create two pods. A client and server pod
kubectl apply -f ./docs/ops-training/ops-training-manifests/networking/pods-network-policy.yaml

# Find the pod IP address for client and server
k get po -n test-network-policy -o wide

# Test a connection to port 80 from the client pod to the server pod
kubectl exec -it client -n test-network-policy -- curl <server IP address>:80

# Create a network policy that denies traffic between pods with the client label and pods with the server label
kubectl apply -f ./docs/ops-training/ops-training-manifests/networking/network-policy-block-client-to-server.yaml

# Test a connection to port 80 from the client pod to the server pod
kubectl exec -it client -n test-network-policy -- curl <server IP address:80
```
Clean up example 2
```bash
Clean up example 1 (leave the namespace - we'll delete that at the end)
```bash
# clean up
kubectl delete -f ./docs/ops-training/ops-training-manifests/networking/network-policy-block-client-to-server.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/networking/pods-network-policy.yaml
k delete ns test-network-policy
```
