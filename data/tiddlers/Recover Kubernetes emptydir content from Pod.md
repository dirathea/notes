Kubernetes has a simple volume type called `emptydir` where it asks the node to spare a disks that mounted on runtime. This type of volume really useful to store temporary files, logs, etc. This volume "survive" on container restarts, but removed on pod removal, makes it helpful for persisting data across restart process, without consumimg unnecessary disks.

But things gets more interesting afterward. Imagine a workload that runs as Kubernetes Job, then suddenly failing / misbehaving and we need to check the temporary files that its produces to emptyDir. Sure, mounting PVC could be one way to solve this, but it adds another layer just for debugging purposes.

Based on the https://kubernetes.io/docs/concepts/storage/volumes/#emptydir emptydir contents will persist as long as the pod is available on the node. This mechanism also works for popular tools like fluentd/fluentbit as log collector, etc. So I wonder, is it possible to retrieve emptyDir content when the pod hasn't been removed, but not in running state, for example on `Completed`, or `Error` state. Technically the pod is still there, isn't it? because we still able to get the pod using `kubectl get pod`.

# Preparation

I start to simply create a k8s cluster using K3D, spin up very minimal cluster to run 2 main pods. 1 pod will be the `target`, where the pod will mount an emptyDir, write file to it, then exit. Another pod will be the `fetch` where it mainly used for manual retriving from `target` pod emptyDir.

# Experiments

At first, we need to understand how emptyDir works. Basically when a pod with emptyDir scheduled on a node, it will be mounted as container mount from a path on the host. The host path location for empytDir are constructed in `/var/lib/kubelet/pods/{containerID}/volumes/kubernetes.io~empty-dir/{volume name}`. Logically, running another pod on the same node with `hostPath` volume, targeting that path should be enable us to retrieve the data.

Here are simple `target` pod and `fetch` pod declaration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: target
  labels:
    name: target
spec:
  containers:
  - name: target
    image: busybox
    command:
      - sh
    args:
      - -c
      - echo "hey! please retrieve me" > /target/output.txt && ls -al /target && sleep 30 && exit 1 # This could simulate the pod states into Error, or Completed. Remove the exit command to make it Completed instead of Error
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    volumeMounts:
      - name: target-output
        mountPath: /target
  restartPolicy: Never # To ensure that the pod in Error / Completed state, not CrashloopBack
  terminationGracePeriodSeconds: 0
  nodeName: k3d-local-agent-0
  volumes:
    - name: target-output
      emptyDir: {}

```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fetch
  labels:
    name: fetch
spec:
  containers:
  - name: target
    image: bash
    command:
      - tail
    args:
      - -f
      - /dev/null
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    volumeMounts:
      - name: fetch-target
        mountPath: /target
        # hostPath best practices to configure it as readOnly.
        readOnly: true
  restartPolicy: Never
  terminationGracePeriodSeconds: 0
  nodeName: k3d-local-agent-0
  volumes:
    - name: fetch-target
      hostPath:
        # mounting the root of pods directory so we could handle the dynamic pod id to explore
        path: /var/lib/kubelet/pods
        type: Directory
```

# Completed pod

when runs the target, configure it to write files into emptyDir and `exit 0`, the pod will be in Completed state. Try fetching it via `fetch` pod, the files is empty :(

# Error pod

Very similar story with Completed pod, the directory is empty when mounted on `fetch` pod.

# CrashloopBack pod

Changing the pod restart policy to Always, changes the end state results from Error to CrashloopBack. On this state, the files persist and we could retrieve the emptydir content via `kubectl cp` for example.