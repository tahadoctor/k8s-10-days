```bash
# stop new pods from landing on the node, running pods are not touched
kubectl cordon worker1
kubectl get nodes        # shows SchedulingDisabled

# test: new pods will not go to worker1
kubectl create deployment cordon-test --image=nginx:1.25 --replicas=4
kubectl get pods -o wide

# back to normal
kubectl uncordon worker1
```
