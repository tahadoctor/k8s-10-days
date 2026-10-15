```bash
kubectl apply -f rc.yaml
kubectl apply -f rs.yaml
kubectl get rc,rs
kubectl get pods -w

# self-healing: delete a pod and watch a new one appear
kubectl delete pod <pod-name>

# scale
kubectl scale rs web-rs --replicas=5
```
