# Kubernetes in 10 Days: Hands-on

I have been learning DevOps. This repo has the files and commands I used
while practicing Kubernetes on my own cluster for 10 days.

## Cluster setup
I built a cluster in two ways:
1. Vagrant, using local VMs as the master and worker nodes (see `Vagrantfile`)
2. The official Kubernetes documentation with kubeadm

## Folders
| Folder | What it has |
|---|---|
| 01-deployment-stateless | Deployment (stateless app) |
| 02-statefulset | StatefulSet with stable pod names, and one with a disk per pod |
| 03-rc-rs | ReplicationController, ReplicaSet, matchExpressions |
| 04-labels-selectors | Pods with labels, selector commands |
| 05-taints-tolerations | NoSchedule, PreferNoSchedule, NoExecute |
| 06-cordon-uncordon | Cordon and uncordon commands |
| 07-node-affinity | required and preferred rules |
| 08-pod-affinity | Web pods placed next to a cache pod |
| 09-pod-anti-affinity | required and preferred spreading |

## How to use
```bash
kubectl apply -f 03-rc-rs/rs.yaml
kubectl get pods -o wide
```

Any passwords in the YAML files are demo values only.

## Next
Services, Ingress, ConfigMaps, Secrets, Helm.
