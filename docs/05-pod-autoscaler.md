# Part 5 - Horizontal Pod Autoscaler
## Deployment Pod autoscaler YAML definition
```yaml
# Deployment to demonstrate autoscaling a pod

apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  labels:
    app: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: demo-container
        image: nginx
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
        ports:
        - containerPort: 80

```
## Create deployment to test pod autoscaler
```bash
# Create a deployment
kubectl apply -f ./docs/ops-training/ops-training-manifests/scaling/pod-ha-scaling.yaml

```
## Service definition to test pod autoscaler - YAML
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-service
spec:
  selector:
    app: demo
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP

```
## Create service for pod autoscaler
```bash
# Create pod autoscaler
kubectl apply -f ./docs/ops-training/ops-training-manifests/scaling/svc-ha-scaling.yaml
```

## Horizontal Autoscaler definition - YAML
```yaml
# This HPA scales a deployment when 50% CPU utilization is reached
# It will scale to a maximum of 5 pods
# It will scale down 2 minutes after the load generator job ends

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: demo-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: demo-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 120  # Scale down after 2 minutes
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60

```
## Create Horizontal Pod Autoscaler
```bash
# Create horizontal pod autoscaler
kubectl apply -f ./docs/ops-training/ops-training-manifests/scaling/hpa.yaml

```
## Job to create load to autoscale the pod
```yaml
# The stress utility is a purpose-built tool for generating load on CPU, memory, and other system resources.

apiVersion: batch/v1
kind: Job
metadata:
  name: load-generator
spec:
  template:
    spec:
      containers:
      - name: loader
        image: progrium/stress
        args:
        - "--cpu"
        - "2"  # Number of CPUs to stress
        - "--timeout"
        - "300s"  # Run for 300 seconds (5 minutes)
      restartPolicy: Never
  backoffLimit: 4

```

## create job auto-scaler
This will direct HTTP requests to the demo service which will cause the associated pods to scale out
```bash
# Create job to initiate HPA
kubectl apply -f ./docs/ops-training/ops-training-manifests/scaling/job-autoscaler.yaml

```
## Monitor
Look at the HPA in action.
```bash
# Check the status of the HPA
k get hpa

# Check system resources used by pods in the default namespace
kubectl top pods

# Look for scaling pods
kubectl get pods -l app=demo --watch

```

## Scale down the HPA
```bash
# Delete the load generator job
kubectl delete -f ./docs/ops-training/ops-training-manifests/scaling/job-autoscaler.yaml

```

## Watch the Pods scale down
```bash
# Look for pods scaling down
kubectl get pods -l app=demo --watch
```

## Clean up the demo
```bash
kubectl delete -f ./docs/ops-training/ops-training-manifests/scaling/pod-ha-scaling.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/scaling/svc-ha-scaling.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/scaling/hpa.yaml
```
