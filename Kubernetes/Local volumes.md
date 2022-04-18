# Kubernetes local volumes

The following YAML file describes a volume persisted locally on the node.

You'll have to manually create the directory /root/master-pv1. 

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: master-pv1
spec:
  capacity:
    storage: 20Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /root/master-pv1
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - kubernetes-master
```

You can then request it with the following PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: some-pv-claim
  labels:
    app: some-app
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: "local-storage"
  resources:
    requests:
      storage: 20Gi
```