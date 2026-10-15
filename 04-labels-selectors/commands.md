```bash
kubectl apply -f pods-with-labels.yaml
kubectl get pods --show-labels

# equality selector
kubectl get pods -l env=prod

# set-based selectors
kubectl get pods -l 'env in (dev,prod)'
kubectl get pods -l 'env notin (dev)'
kubectl get pods -l env

# add, change and remove a label
kubectl label pod pod-dev team=backend
kubectl label pod pod-dev env=test --overwrite
kubectl label pod pod-dev team-
```

Relabeling a pod owned by a ReplicaSet makes the RS stop counting it,
so the RS creates a replacement pod.
