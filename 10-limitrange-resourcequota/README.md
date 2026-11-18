# LimitRange and ResourceQuota

CPU units: 1 CPU = 1000m (millicores). So 500m is half a core and 100m is 0.1 core.

Every container has a request and a limit.
- request = the CPU the pod is guaranteed to get
- limit = the max CPU it can use

A request can never be bigger than the limit.

## LimitRange

A LimitRange sets rules for each container in a namespace.

- min: container can't ask for less than this
- max: container can't ask for more than this
- default: limit added if I don't write a limit
- defaultRequest: request added if I don't write a request

default is only a fallback value, not a restriction. The real upper cap is max.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraint
spec:
  limits:
  - default:
      cpu: 500m
    defaultRequest:
      cpu: 500m
    max:
      cpu: "1"
    min:
      cpu: 100m
    type: Container
```

For a pod to be created this should hold:

```
min <= request <= limit <= max
```

## ResourceQuota

A ResourceQuota limits the total of the whole namespace. All pods are added up and checked against it.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cpu-quota
spec:
  hard:
    requests.cpu: "500m"
    pods: "10"
```

LimitRange checks each pod on its own. ResourceQuota checks the total. A pod is created only if it passes both.

## Practical

Pod with only a request of 700m and no limit:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-conflict-with-limitrange-cpu
spec:
  containers:
  - name: demo
    image: registry.k8s.io/pause:3.8
    resources:
      requests:
        cpu: 700m
```

It failed with this error:

```
kubectl create -f pod.yml
The Pod "example-conflict-with-limitrange-cpu" is invalid:
spec.containers[0].resources.requests: Invalid value: "700m":
must be less than or equal to cpu limit of 500m
```

Reason: no limit was given, so the LimitRange added its default limit of 500m. 700m is more than 500m, so it failed. The 500m did not come from me, it came from the LimitRange default.

After changing the request:

| request | limit (default) | result |
|---|---|---|
| 700m | 500m | fails |
| 200m | 500m | created |
| 400m | 500m | created |

The second pod needs a different name (I added -1) because two pods can't have the same name.

If 700m is really needed, give the limit yourself:

```yaml
resources:
  requests:
    cpu: 700m
  limits:
    cpu: 900m
```

Other cases:
- request 500m, no limit: works
- request 600m, no limit: fails (default limit is 500m)
- limit 2 CPU: fails, max is 1 CPU
- request 50m: fails, min is 100m

## ResourceQuota example

Quota requests.cpu = 500m. pod-a asks 400m, pod-b asks 200m.

- pod-a: 400m used out of 500m, created
- pod-b: 400 + 200 = 600m, more than 500m, rejected

With a quota of 600m or 1 CPU both would be created. Order matters, the pod that comes first takes the quota.

## Commands

```bash
kubectl create -f pod.yml
kubectl describe limitrange -n <namespace>
kubectl describe quota -n <namespace>
```
