# Training Part 3 - Networking (continued)
## Show and tell
Create 3 NGINX Pods, and an ingress object

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
### Manifest 2: a service of type ClusterIp
```yaml
# Create a service of type ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: lb-service-test
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
###  Manifest 3: Ingress object (Internal LB)

```yaml
# Create an ingress object
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"  # Internal LB for Azure
spec:
  ingressClassName: nginx  # Specifies the Ingress class name
  rules:
  - host: nginx.internal
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

## Demonstration
### Look at the service deployment and ingress
```bash
# Create 3 NGINX Pods, an internal LB service, and a client pod to demonstrate
kubectl apply -f ./docs/ops-training/ops-training-manifests/networking/ingress-example.yaml

# Look at the deployment. Notice the IP address of each pod 
kubectl get po -o wide

# Look at the service. 
kubectl get svc -o wide

# Notice that the service IP address is not routable from outside the cluster.
# ping the service from your client machine
ping <ip address of the svc>

# Inspect the ingress object
k get ingress

```

### Demonstrate access to the application from outside the cluster through an ingress object
1. Create a local hosts record for nginx.internal on your local machine
2. From a browser access http://nginx.internal

NOTE: the ingress controller running on the cluster is configured to use session affinity, so anytime you refresh the browser the welcome message stays the same

## NOTE: Traffic Splits DO NOT WORK with our Ingress Controller (Nginx... Nginx Plus is required)

### Demonstrate an ingress with a traffic split
In this demonstration we will deploy a second version of the application and we will split the traffic between the two applications. The below manifest is for the new ingress object.

It will split the traffic between two different applications
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-split-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "always"
    nginx.ingress.kubernetes.io/canary-weight: "50"  # 50% traffic to v2
spec:
  rules:
  - host: nginx.split.internal
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-v1-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-v2-service
            port:
              number: 80
```
Here is the deployment that we'll be using. It's very similar to the original deployment. In this scenario we are deploying 2
versions of the application. The first deployment is for version 1 and the second for version 2.

Here is version 1 of the deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v1
  labels:
    app: lb-service-test
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: lb-service-test
      version: v1
  template:
    metadata:
      labels:
        app: lb-service-test
        version: v1
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
        command: ['sh', '-c', "echo 'Welcome to nginx Version 1 from pod $(POD_NAME)' > /usr/share/nginx/html/index.html && sleep 3600"]
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

Here is version 2 of the deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v2
  labels:
    app: lb-service-test
    version: v2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: lb-service-test
      version: v2
  template:
    metadata:
      labels:
        app: lb-service-test
        version: v2
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
        command: ['sh', '-c', "echo 'Welcome to nginx Version 2 from pod $(POD_NAME)' > /usr/share/nginx/html/index.html && sleep 3600"]
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
As you can see there are minor differences between the deployments.
```bash
# Let's deploy the application
kubectl apply -f ./docs/ops-training/ops-training-manifests/networking/ingress-trafficsplit-example.yaml

# Look at the current deployments. Notice that we now have three deployments. The original from the first section and two from the traffic spit exercise
k get deploy

# Look at the current ingress objects. Notice that we have a second ingress object for the new application.
# Pay attention to the IP address of the ingress object we'll need this in the next step 
k get ingress

```
## NOTE: Traffic Splits DO NOT WORK with our Ingress Controller (Nginx... Nginx Plus is required)

### Clean-up the environment
```bash
# destroy the deployments, and service
kubectl delete -f ./docs/ops-training/ops-training-manifests/networking/lb-service-example.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/networking/ingress-example.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/networking/ingress-trafficsplit-example.yaml
```
