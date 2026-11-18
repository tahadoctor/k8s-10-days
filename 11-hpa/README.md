# HPA (Horizontal Pod Autoscaler)

Notes and practical from my minikube cluster.

## What is HPA

HPA changes the number of pods of a Deployment on its own based on load. If CPU goes up it adds pods, if load drops it removes them. I only give it a target (like CPU 50%) and a min and max number of pods. No need to run `kubectl scale` by hand.

Horizontal means number of pods. Making one pod bigger (more CPU/RAM) is vertical scaling, that is VPA.

## How it works

metrics-server collects the CPU/memory usage of pods. The HPA controller checks these metrics every 15 seconds or so and calculates:

```
desiredReplicas = ceil( currentReplicas x currentUsage / targetUsage )
```

Example: 1 pod using 40% CPU, target is 10%. So ceil(1 x 40 / 10) = 4 pods. The result is always kept between min and max.

## Things needed before HPA works

- metrics-server running in the cluster
- CPU request set in the Deployment. HPA calculates the percentage from the request (usage / request). Without a request it shows a missing request for cpu error
- A Deployment, ReplicaSet or StatefulSet to scale. It does not work on a bare pod or a DaemonSet

One thing I understood while doing this: with `requests.cpu: 100m` and target 10%, HPA starts adding pods when the average usage per pod goes above 10m. So the percentage is of the request, not of the whole node.

## Practical

### 1. Install metrics-server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.7.2/components.yaml
kubectl get pod -n kube-system
kubectl top nodes
```

In my cluster the metrics-server pod stayed 0/1 and `kubectl top nodes` gave:

```
error: Metrics API not available
```

The reason is that the kubelet in a lab cluster uses a self-signed certificate.

### 2. Fix metrics-server

```bash
kubectl edit deployment -n kube-system metrics-server
```

Under `containers:` and `args:` I added one line, with the same indentation as the other args:

```yaml
    - --kubelet-insecure-tls
```

After saving, a new metrics-server pod is created. I waited about a minute until it was 1/1 Running. Then:

```bash
kubectl top nodes
kubectl top pods
```

Now real numbers came, for example `minikube 218m 10% 922Mi 54%`.

Note: this flag is fine for a lab only. In production the kubelet should have a proper certificate.

### 3. Create the deployment

```bash
kubectl create deployment nginx --image=nginx --replicas=1
kubectl get pods
```

The name in my notes was mynginx but I used nginx, so I had to use nginx in every command after this. When the name did not match I got `deployments.apps "mynginx" not found`.

### 4. Add CPU request to the deployment

```bash
kubectl edit deploy nginx
```

```yaml
    spec:
      containers:
      - image: nginx
        name: nginx
        resources:
          requests:
            cpu: 100m
```

The same thing can be done without editing:

```bash
kubectl set resources deployment nginx --requests=cpu=100m
```

After this the pod name changed (a new ReplicaSet is created when the pod template changes), so describing the old pod name gave not found.

### 5. Create the HPA

```bash
kubectl autoscale deployment nginx --cpu=10% --min=1 --max=10
kubectl get hpa
```

I used a low target of 10% so it is easy to cross during the test. The older flag `--cpu-percent=10` still works but gives a deprecation warning.

Same HPA as YAML:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 10
```

At first TARGETS showed `cpu: <unknown>/10%`. It stays like this for a minute or two until metrics-server gives the first data, then it changes to `0%/10%`.

```bash
kubectl describe hpa nginx
```

In the conditions, `ScalingActive True` means metrics are being read fine. `ScalingLimited True (TooFewReplicas)` just means the load is 0 and it is sitting at min 1 replica. Old `FailedGetResourceMetric` events are only history, not a current error.

### 6. Expose the deployment as a service

```bash
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl get svc
```

Testing it:

```bash
curl http://nginx
```

This failed on the host with `Could not resolve host: nginx`, because the service name only resolves inside the cluster. Two ways that work:

```bash
# from inside a pod
kubectl exec -it <nginx-pod-name> -- /bin/bash
curl http://nginx

# from the host, using the ClusterIP
curl http://10.96.36.160
```

### 7. Generate load

```bash
kubectl run load-generator --image=busybox:1.28 --restart=Never -- sh -c 'while true; do wget -q -O- http://nginx; done'
```

One load generator was not enough to cross 10%, so I ran 4 of them (load-generator, load-generator-1, load-generator-2, load-generator-3) with the same command.

### 8. Watch it scale

In a second terminal:

```bash
kubectl get hpa -w
kubectl get pods
kubectl top pods
```

What I saw: `cpu: 12%/10%` and REPLICAS went up to 10 (the max). 9 pods were Running and 1 pod was Pending.

Why Pending:

```bash
kubectl describe pod <pending-pod-name>
```

```
FailedScheduling: 0/1 nodes are available: 1 Insufficient cpu.
```

HPA asked for 10 pods, but minikube has only one node and its CPU (by requests) was full. HPA only increases the number of pods, it cannot add nodes. Adding nodes is the job of the Cluster Autoscaler.

### 9. Stop the load and watch scale down

```bash
kubectl delete pod load-generator load-generator-1 load-generator-2 load-generator-3
kubectl get hpa -w
```

CPU went to 0% quickly but the pods did not reduce at once. It took about 5 minutes to come back to 1 replica. This is the stabilization window and it is normal. It is there so the pod count does not keep going up and down.

### 10. Cleanup

```bash
kubectl delete hpa nginx
kubectl delete svc nginx
kubectl delete deployment nginx
```

## Extra things I looked up

HPA can also use memory:

```yaml
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

The scale down wait time can be changed with the behavior field (autoscaling/v2):

```yaml
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
```

Changing min and max of an existing HPA:

```bash
kubectl patch hpa nginx -p '{"spec":{"minReplicas":2,"maxReplicas":8}}'
```

If I run `kubectl scale` on a deployment that has an HPA, the HPA changes the replicas back according to its own metrics. So manual scaling does not stay.

HPA vs VPA vs Cluster Autoscaler:

- HPA changes the number of pods
- VPA changes the CPU/memory of a pod (the pod restarts)
- Cluster Autoscaler changes the number of nodes

HPA and VPA should not run on the same metric (CPU or memory) together, they will work against each other.

## Mistakes I made

| Problem | Reason and fix |
|---|---|
| Metrics API not available | metrics-server was 0/1, added --kubelet-insecure-tls |
| Edit cancelled, no changes made | did not save in vim, use :wq |
| cpu: unknown/10% | metrics not available yet, wait 1 to 2 minutes, or requests missing in the deployment |
| mynginx not found | deployment name was nginx, keep the same name everywhere |
| curl http://nginx fails on host | service name works only inside a pod, use exec or the ClusterIP |
| command not found errors | I pasted the terminal output back, copy only the command |
| One pod Pending, Insufficient cpu | single node ran out of CPU, keep max lower or add a node |

## Commands

```bash
kubectl top nodes
kubectl top pods
kubectl get hpa
kubectl get hpa nginx -w
kubectl describe hpa nginx
kubectl autoscale deployment nginx --cpu=50% --min=1 --max=10
kubectl edit hpa nginx
kubectl delete hpa nginx
kubectl set resources deployment nginx --requests=cpu=100m --limits=cpu=200m
```
