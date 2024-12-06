# Part 5 - Resource Requests and Resource Limits Demonstration

| Scenario | Result |
|----------|----------|
| CPU underutilized | The unused capacity is available for other workloads|
| CPU overutilized | CPU throttling occurs |
| Memory underutilized | Unused memory is available for other pods |
| Memory overutilized | If a pod has a resource limit set, the process is terminated and the pod status becomes OOMKilled |

We will demonstrate these scenarios below.

## Step 1: Create a deployment with a resource request (minimum) and resource limit (maximum)
### Deployment yaml example
```yaml
# Deployment with CPU/memory requests and limits

apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-demo
  labels:
    app: resource-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resource-demo
  template:
    metadata:
      labels:
        app: resource-demo
    spec:
      containers:
      - name: demo-container
        image: nginx:latest
        resources:
          requests:
            memory: "128Mi" # Minimum amount of memory requested for the pod
            cpu: "250m" # Minimum amount of millicores requested for the pod. 250m = 25% of one 1 core.
          limits:
            memory: "256Mi" # Maximum amount of memory allowed for pod. OOMKill will occur if more than 256Mi of memory is used
            cpu: "500m" # Maximum amount of CPU allowed for this pod. CPU will be throttled for this Pod if more than 500 millicores are used: 500m = 50% of one CPU core
```
### Create Deployment with request resource and limit
```bash
# Create the demo deployment - resource limits
kubectl apply -f ./docs/ops-training/ops-training-manifests/requests-limits/deploy-resource-demo.yaml

# Create the service - no svc example as it is a simple cluserIP svc type
kubectl apply -f ./docs/ops-training/ops-training-manifests/requests-limits/svc-resource-demo.yaml

```
## Step 2: Create a load testing pod with K6
### Deployment configMap with 10 virtual users
```yaml
# ConfigMap references the script used by the K6 load test pod. This ConfigMap is mounted as a volume in the pod
apiVersion: v1
kind: ConfigMap
metadata:
  name: k6-load-test-script
data:
  load-test.js: |
    import http from 'k6/http';
    import { sleep } from 'k6';

    export const options = {
      vus: 1000, // Number of virtual users
      duration: '30s', // Test duration
    };

    export default function () {
      http.get('http://resource-demo-service.default.svc.cluster.local');
      sleep(1);
    }
```
### Deployment that refrences the CM above... 10 users
```yaml
# Create a pod used to generate a load on on the resource-demo pod
apiVersion: v1
kind: Pod
metadata:
  name: k6-load-test
spec:
  containers:
  - name: k6
    image: grafana/k6:latest
    command:
      - "k6"
      - "run"
      - "/scripts/load-test.js"
    resources:
      requests:
        cpu: "2"
        memory: "2Gi"
      limits:
        cpu: "4"
        memory: "4Gi"
    volumeMounts:
    - name: script-volume
      mountPath: /scripts
  restartPolicy: Never
  volumes:
  - name: script-volume
    configMap:
      name: k6-load-test-script
```
### Create load test deployment
```bash
# Create the configMap with a script for 10 virtual users
kubectl apply -f ./docs/ops-training/ops-training-manifests/requests-limits/cm-10vu-120seconds.yaml

# Create K6 load test pod and configMap
kubectl apply -f ./docs/ops-training/ops-training-manifests/requests-limits/k6-load-test-pod.yaml
```

## Step 3: Observe
Check resources via top
```bash
# View resource usage
kubectl top pod
```
Check logs vie k6 load tester
```bash
k logs k6-load-test
```

Check Grafana for the resource-demo pod

[DKP Dashboard](https://dkp-mgt.amwins.local/dkp/kommander/dashboard)

### Adjust the load
Change the number of virtual users to 100 and set the duration to 120 seconds
YAML configMap
```yaml
# We're going to change the ConfigMap using this example

apiVersion: v1
kind: ConfigMap
metadata:
  name: k6-load-test-script
data:
  load-test.js: |
    import http from "k6/http";
    import { sleep } from "k6";

    export const options = {
      vus: 100, // Number of virtual users
      duration: "120s", // Test duration
    };

    export default function () {
      http.get("http://resource-demo-service.default.svc.cluster.local");
      sleep(1);
    }
```
Apply the CM with 100 virtual users and 120 second duration
```bash
# Change the number of virtual users to 100 and duration of 120 seconds
kubectl apply -f ./docs/ops-training/ops-training-manifests/requests-limits/cm-100vu-120seconds.yaml

# restart the k6-load-test pod
kubectl delete pod k6-load-test

# recreate the load generator pod
kubectl apply -f ./docs/ops-training/ops-training-manifests/requests-limits/k6-load-test-pod.yaml
```
Check Grafana for the resource-demo pod
[DKP Dashboard](https://dkp-mgt.amwins.local/dkp/kommander/dashboard)

#### Increase the number of virtual users to 1k
YAML configMAp
```yaml
# We're going to change the ConfigMap using this example

apiVersion: v1
kind: ConfigMap
metadata:
  name: k6-load-test-script
data:
  load-test.js: |
    import http from "k6/http";
    import { sleep } from "k6";

    export const options = {
      vus: 1000, // Number of virtual users
      duration: "120s", // Test duration
    };

    export default function () {
      http.get("http://resource-demo-service.default.svc.cluster.local");
      sleep(1);
    }
```
Check Grafana for the resource-demo pod

[DKP Dashboard](https://dkp-mgt.amwins.local/dkp/kommander/dashboard)
#### Increase to 1000 virtual users
```bash
# Change the number of virtual users to 1K and duration of 120 seconds
kubectl apply -f ./docs/ops-training/ops-training-manifests/requests-limits/cm-1000vu-120seconds.yaml

# delete the k6-load-test pod
kubectl delete pod k6-load-test

# recreate the load generator pod
kubectl apply -f ./docs/ops-training/ops-training-manifests/requests-limits/k6-load-test-pod.yaml
```
Check Grafana for the resource-demo pod

[DKP Dashboard](https://dkp-mgt.amwins.local/dkp/kommander/dashboard)
#### Increase to 5k virtual users
```bash
# Change the number of virtual users to 5K and duration of 120 seconds
kubectl apply -f ./docs/ops-training/ops-training-manifests/requests-limits/cm-5kvu-120seconds.yaml

# delete the k6-load-test pod
kubectl delete pod k6-load-test

# recreate the load generator pod
kubectl apply -f ./docs/ops-training/ops-training-manifests/requests-limits/k6-load-test-pod.yaml
```
Check resource used in K8s for the pod and check the K6 logs
```bash
k top po

# Look at the k6 load test logs
k logs k6-load-test
```
Check Grafana for the resource-demo pod

[DKP Dashboard](https://dkp-mgt.amwins.local/dkp/kommander/dashboard)
#### Increase the number of virtual users to 10k
```bash
# Change the number of virtual users to 10k and diration of 120 second
kubectl apply -f ./docs/ops-training/ops-training-manifests/requests-limits/cm-10kvu-120seconds.yaml

# delete the k6-load-test pod
kubectl delete pod k6-load-test

# recreate the load generator pod
kubectl apply -f ./docs/ops-training/ops-training-manifests/requests-limits/k6-load-test-pod.yaml

```
Check Grafana for the resource-demo pod

[DKP Dashboard](https://dkp-mgt.amwins.local/dkp/kommander/dashboard)
#### Check resources used in K8s for the pod and check the K6 logs
```bash
k top po

# Look at the k6 load test logs and notice failure
k logs k6-load-test
```
The k6-load-test pod failed with warning request failed. This is due to over 2 cores being used by the k6-load-test pod. The pod cpu limit for the load tester is 2 CPU (2000 milicores = 2 CPU cores). 

## Memory over-utilization
If a pod has a resource limit set, the process is terminated and the pod status becomes OOMKilled
```bash
# find the name of a pod to test with
k get po

# Run a memory intense script in the pod
k exec -it <pod-name> -- sh -c "dd if=/dev/zero of=/dev/null bs=1M count=512"

# Describe the pod
k describe pod <pod-name>

# Check the pod logs
k logs <pod-name>
```

## Clean up the environment
```bash
kubectl delete -f ./docs/ops-training/ops-training-manifests/requests-limits/k6-load-test-pod.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/requests-limits/deploy-resource-demo.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/requests-limits/svc-resource-demo.yaml
kubectl delete -f ./docs/ops-training/ops-training-manifests/requests-limits/cm-10kvu-120seconds.yaml

```

