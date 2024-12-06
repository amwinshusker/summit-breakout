# Training Part 3 - Networking
## Show and tell
Create 3 NGINX Pods, an internal LB service, and a client pod to demonstrate
We'll use 3 manifests in this demonstration

### Manifest 1: Deployment - 3 NGINX Pods
```yaml
# Create a deployment to access through a service
# this deployment uses a side car container to write to an HTML file on the NGINX container
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: lb-service-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: lb-service-test
  template:
    metadata:
      labels:
        app: lb-service-test
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html

      - name: content-generator
        image: busybox
        command: ['sh', '-c', "echo 'Welcome to nginx from pod $(POD_NAME)' > /usr/share/nginx/html/index.html && sleep 3600"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html

      volumes:
      - name: html-volume
        emptyDir: {}
```
### Manifest 2: Service of type LoadBalancer - Internal
```yaml
# Create a service of type load-balancer - Private
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb-service
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"  # Internal LoadBalancer for Azure
spec:
  selector:
    app: lb-service-test
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
### Manifest 3: Create a client we can use to show all three pods are accessed
```yaml
# create a client Pod used to send traffic to the NGINX service
apiVersion: v1
kind: Pod
metadata:
  name: curl-client
spec:
  containers:
  - name: curl-client
    image: curlimages/curl:latest
    command: ["/bin/sh", "-c", "sleep 3600"]
```
## Demonstration
### Show a client accessing the Pod through a load balancer
```bash
# Create 3 NGINX Pods, an internal LB service, and a client pod to demonstrate
kubectl apply -f ./docs/ops-training/ops-training-manifests/networking/lb-service-example.yaml

# Look at the deployment. Notice the IP address of each pod 
kubectl get po -o wide

# Look at the service.
kubectl get svc -o wide

# exec into the client pod
kubectl exec -it curl-client -- /bin/sh

# access the application through the load balancer
curl http://nginx-lb-service
curl http://nginx-lb-service
curl http://nginx-lb-service
curl http://nginx-lb-service

# notice how the pod name changes when using the curl command multiple times. The service is a basic load balancer
```

### Demonstrate how the service re-attaches after a pod is deleted
```bash
# Delete one of the Pods
kubectl get po -o wide
kubectl delete po <pod-name>

# Look at the pods again to see if a new IP address was given
kubectl get po -o wide

# access the application through the service again to prove that it is re-associated
kubectl exec -it curl-client -- /bin/sh

# access the application through the load balancer
curl http://nginx-lb-service
curl http://nginx-lb-service
curl http://nginx-lb-service
curl http://nginx-lb-service

# exit the client container
exit
```

### Clean-up the environment
```bash
# destroy the deployments, and service
kubectl delete -f ./docs/ops-training/ops-training-manifests/networking/lb-service-example.yaml
```
