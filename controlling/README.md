### Controlling workloads

#### Changing permissions for a pod
Run a simple pod like below. Of course, feel free to use your own or some other pod if you like.

```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-example
spec:
  containers:
  - name: sec-ctx
    image: gcr.io/google-samples/node-hello:1.0
```

Once started, shell into the pod 

`kubectl exec -it security-context-example -- sh`

execute the `id` and `ps aux` commands and see what userid and groupid you have and under which account the processes are running.

Now, delete the pod and recreate it with the following settings:
```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-example
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: sec-ctx
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false
```

Inspect the userid in the pod again

#### Controlling pod admission

Create a new namespace with the following settings:
```
apiVersion: v1
kind: Namespace
metadata:
  name: controlling
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

Now apply the pod.yaml you created above to this namespace and observe any messages.

When trying to run a container with too many permissions it will be blocked by Kubernetes. Now try again with the settings below:

```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-example
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: sec-ctx-demo-2
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      runAsNonRoot: true
      runAsUser: 2000
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      seccompProfile:
        type: RuntimeDefault
```

Inspect the newly added configuration and read more about these settings here: [https://kubernetes.io/docs/concepts/security/pod-security-admission/](https://kubernetes.io/docs/concepts/security/pod-security-admission/)

#### Resource quota

Create a resource quota for your namespace:
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: example
  namespace: controlling
spec:
  hard:
    requests.cpu: '1'
    requests.memory: 1Gi
    limits.cpu: '2'
    limits.memory: 2Gi
    pods: '1'
    persistentvolumeclaims: '5'
    requests.storage: 5Gi
```

Drop the pod you created before and 
create a deployment and apply it to this namespace and observe any messages.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  extra-app
  namespace: controlling
  labels:
    app:  extra-app
spec:
 selector:
    matchLabels:
      app: extra-app
 replicas: 1
 template:
    metadata:
      labels:
        app:  extra-app
    spec:
      containers:
      - name:  extra-app
        image: gcr.io/google-samples/node-hello:1.0
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 100m
            memory: 100Mi
        securityContext:
          runAsNonRoot: true
          runAsUser: 2000
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
          seccompProfile:
            type: RuntimeDefault
      restartPolicy: Always
```

Now inspect what happens when you request more replicas. Hint: look at the deployment object and any replicasets as well