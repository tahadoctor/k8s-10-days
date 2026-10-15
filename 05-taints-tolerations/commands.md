Three taint effects:

| Effect | Meaning |
|---|---|
| NoSchedule | New pods without a toleration are not scheduled. Running pods stay. |
| PreferNoSchedule | Soft version. Scheduler tries to avoid the node but can still use it. |
| NoExecute | New pods blocked and running pods without a toleration are evicted. |

```bash
# add a taint
kubectl taint nodes worker1 env=prod:NoSchedule
kubectl taint nodes worker1 env=prod:PreferNoSchedule
kubectl taint nodes worker1 env=prod:NoExecute

# check
kubectl describe node worker1 | grep Taints

# remove a taint (note the minus at the end)
kubectl taint nodes worker1 env=prod:NoSchedule-
kubectl taint nodes worker1 env=prod:PreferNoSchedule-
kubectl taint nodes worker1 env=prod:NoExecute-
```

A toleration only allows a pod on a tainted node. It does not pull the pod
there. To attract a pod to a node use node affinity.
