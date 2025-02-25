# Using sidecar containers

The Operator allows you to deploy additional (so-called *sidecar*) containers to
the Pod. You can use this feature to run debugging tools, some specific
monitoring solutions, etc.

!!! note

    Custom sidecar containers [can easily access other components of your cluster](https://kubernetes.io/docs/concepts/workloads/pods/#resource-sharing-and-communication).
    Therefore they should be used carefully and by experienced users only.

## Adding a sidecar container

You can add sidecar containers to Percona Server for MySQL Pods. Just use
`sidecars` subsection ing the `mysql` section of the `deploy/cr.yaml`
configuration file. In this subsection, you should specify the name and image of
your container and possibly a command to run:

```yaml
spec:
  mysql:
    ....
    sidecars:
    - image: busybox
      command: ["sleep", "30d"]
      name: my-sidecar-1
    ....
```

Apply your modifications as usual:

```{.bash data-prompt="$"}
$ kubectl apply -f deploy/cr.yaml
```

Running `kubectl describe` command for the appropriate Pod can bring you the
information about the newly created container:

```{.bash data-prompt="$"}
$ kubectl describe pod cluster1-mysql-0
....
Containers:
....
my-sidecar-1:
  Container ID:  docker://e8fbaae09c3b20c49da259c490d65cd68182b227c33e0fec560271a569b01394
  Image:         busybox
  Image ID:      docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678
  Port:          <none>
  Host Port:     <none>
  Command:
    sleep
    30d
  State:          Running
    Started:      Thu, 06 Jan 2022 10:38:15 +0300
  Ready:          True
  Restart Count:  0
  Environment:    <none>
  Mounts:
    /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-lkk2n (ro)
....
```

## Getting shell access to a sidecar container

You can login to your sidecar container as follows:

```{.bash data-prompt="$"}
$ kubectl exec -it cluster1-mysql-0 -c my-sidecar-1 -- sh
/ #
```

## Mount volumes into sidecar containers

It is possible to mount volumes into sidecar containers.

Following subsections describe different [volume types](https://kubernetes.io/docs/concepts/storage/volumes/#volume-types),
which were tested with sidecar containers and are known to work.

### Persistent Volume

You can use [Persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) when you need dynamically provisioned storage which doesn’t depend on the Pod lifecycle.
To use such volume, you should *claim* durable storage with [persistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/volumes/#persistentvolumeclaim) without specifying any non-important details.

The following example requests 1G storage with `sidecar-volume-claim`
PersistentVolumeClaim, and mounts the correspondent Persistent Volume to the
`my-sidecar-1` container’s filesystem under the `/volume1` directory:

```yaml
...
  sidecars:
  - image: busybox
    command: ["sleep", "30d"]
    name: my-sidecar-1
    volumeMounts:
    - mountPath: /volume1
      name: sidecar-volume-claim
  sidecarPVCs:
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: sidecar-volume-claim
    spec:
      resources:
        requests:
          storage: 1Gi
      volumeMode: Filesystem
      accessModes:
        - ReadWriteOnce
```

### Secret

You can use a [secret volume](https://kubernetes.io/docs/concepts/storage/volumes/#secret)
to pass the information which needs additional protection (e.g. passwords), to
the container. Secrets are stored with the Kubernetes API and mounted to the
container as RAM-stored files.

You can mount a secret volume as follows:

```yaml
...
  sidecars:
  - image: busybox
    command: ["sleep", "30d"]
    name: my-sidecar-1
    volumeMounts:
    - mountPath: /secret
      name: sidecar-secret
  sidecarVolumes:
  - name: sidecar-secret
    secret:
      secretName: mysecret
```

The above example creates a `sidecar-secret` volume (based on already existing
`mysecret` [Secret object](https://kubernetes.io/docs/concepts/configuration/secret/))
and mounts it to the `my-sidecar-1` container’s filesystem under the
`/secret` directory.

!!! note

    Don’t forget you need to [create a Secret Object](https://kubernetes.io/docs/concepts/configuration/secret/) before you can use it.

### configMap

You can use a [configMap volume](https://kubernetes.io/docs/concepts/storage/volumes/#configmap) to pass some configuration data to the container.
Secrets are stored with the Kubernetes API and mounted to the container as RAM-stored files.

You can mount a configMap volume as follows:

```yaml
...
  sidecars:
  - image: busybox
    command: ["sleep", "30d"]
    name: my-sidecar-1
    volumeMounts:
    - mountPath: /config
      name: sidecar-config
  sidecarVolumes:
  - name: sidecar-config
    configMap:
      name: myconfigmap
```

The above example creates a `sidecar-config` volume (based on already existing
`myconfigmap` [configMap object](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/))
and mounts it to the `my-sidecar-1` container’s filesystem under the
`/config` directory.

!!! note

    Don’t forget you need to [create a configMap Object](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-a-configmap) before you can use it.
