# ConfigMap and Secret

Notes and practical.

## What is a ConfigMap

A ConfigMap stores normal (non-sensitive) configuration outside the pod, like app mode, log level, URLs or a config file. The benefit is that I can change the config without rebuilding the image.

## What is a Secret

A Secret stores sensitive data like passwords, tokens and API keys.

| | ConfigMap | Secret |
|---|---|---|
| Used for | normal config | sensitive data |
| Stored as | plain text | base64 encoded |
| Example | APP_MODE=prod | DB_PASSWORD |

Important: base64 is only encoding, not encryption. Anyone who can read the Secret can decode it. So access should be controlled with RBAC, and encryption at rest can be enabled on the cluster.

## Practical

### 1. Create a ConfigMap

From literal values:

```bash
kubectl create configmap app-config --from-literal=APP_MODE=prod --from-literal=LOG_LEVEL=info
kubectl get configmap
kubectl describe configmap app-config
```

From a file:

```bash
echo "color=blue" > config.properties
kubectl create configmap file-config --from-file=config.properties
```

With YAML:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: prod
  LOG_LEVEL: info
```

### 2. Create a Secret

```bash
kubectl create secret generic db-secret --from-literal=DB_PASSWORD=MyPass123
kubectl get secret
kubectl describe secret db-secret
```

`describe` does not show the value, only the size. To see the stored value:

```bash
kubectl get secret db-secret -o yaml
```

The value comes as base64 (`TXlQYXNzMTIz`). To decode it:

```bash
echo 'TXlQYXNzMTIz' | base64 -d
```

This shows base64 is not real security.

With YAML the value must be base64:

```bash
echo -n 'MyPass123' | base64
```

The `-n` is important. Without it a newline is added and the password becomes wrong.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_PASSWORD: TXlQYXNzMTIz
```

If I don't want to encode by hand, `stringData` accepts plain text and Kubernetes encodes it:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_PASSWORD: MyPass123
```

### 3. Use them as environment variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: demo
    image: busybox:1.28
    command: ["sh", "-c", "sleep 3600"]
    env:
    - name: APP_MODE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_MODE
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_PASSWORD
```

```bash
kubectl apply -f env-pod.yaml
kubectl exec -it env-pod -- env
```

Both APP_MODE and DB_PASSWORD show up in the env list. Inside the pod the Secret value is already decoded.

To load all keys at once use `envFrom`:

```yaml
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: db-secret
```

### 4. Mount them as a volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vol-pod
spec:
  containers:
  - name: demo
    image: busybox:1.28
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
    - name: secret-vol
      mountPath: /etc/secret
      readOnly: true
  volumes:
  - name: config-vol
    configMap:
      name: app-config
  - name: secret-vol
    secret:
      secretName: db-secret
```

```bash
kubectl apply -f vol-pod.yaml
kubectl exec -it vol-pod -- ls /etc/config
kubectl exec -it vol-pod -- cat /etc/config/APP_MODE
kubectl exec -it vol-pod -- cat /etc/secret/DB_PASSWORD
```

Every key becomes a file, and the file content is the value.

### 5. What happens when I update them

```bash
kubectl edit configmap app-config
```

Change `APP_MODE` from prod to dev, then check:

```bash
kubectl exec -it env-pod -- env | grep APP_MODE
kubectl exec -it vol-pod -- cat /etc/config/APP_MODE
```

| How it is used | After the ConfigMap changes |
|---|---|
| Environment variable | old value stays, pod must be restarted |
| Volume mount | file updates on its own after a short delay (up to about a minute) |

The volume update does not work if the file is mounted with `subPath`.

To restart a deployment after changing a ConfigMap:

```bash
kubectl rollout restart deployment <name>
```

### 6. Cleanup

```bash
kubectl delete pod env-pod vol-pod
kubectl delete configmap app-config file-config
kubectl delete secret db-secret
```

## Extra things I looked up

- A ConfigMap or Secret must be in the same namespace as the pod that uses it.
- If a pod references a ConfigMap or Secret that does not exist, the pod stays in `CreateContainerConfigError`. Check it with `kubectl describe pod`.
- A ConfigMap can be made unchangeable with `immutable: true`. This is also good for performance on big clusters.
- There are other Secret types too, like `kubernetes.io/tls` for certificates and `kubernetes.io/dockerconfigjson` for private registry login.
- The size limit of a ConfigMap or Secret is 1 MiB.
- Never commit a real Secret YAML to GitHub. Base64 is not hiding anything.

## Mistakes to avoid

| Problem | Reason and fix |
|---|---|
| Wrong password inside the pod | base64 was made without `-n`, so a newline got added |
| Env variable still shows old value | env variables are read only when the pod starts, restart the pod |
| CreateContainerConfigError | the ConfigMap or Secret name or key is wrong, or it is in another namespace |
| Config not updating in volume | the file was mounted with subPath |

## Commands

```bash
kubectl get configmap
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
kubectl get secret
kubectl get secret db-secret -o yaml
kubectl edit configmap app-config
kubectl delete configmap app-config
kubectl delete secret db-secret
echo -n 'text' | base64
echo 'dGV4dA==' | base64 -d
```
