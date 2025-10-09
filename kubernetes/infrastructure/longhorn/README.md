# Longhorn

Most of the instructions are from [here](https://joshrnoll.com/installing-longhorn-on-talos-with-helm)

## Steps

Create namespace

```bash
kubectl apply -f namespace.yaml
```

Add longhorn repo

```bash
helm repo add longhorn https://charts.longhorn.io && helm repo update
```

Install on cluster

```bash
helm install longhorn longhorn/longhorn --namespace longhorn-system --values=values.yaml
```

> For now - while I don't have any load balancers, etc set up - the Longhorn UI can be accessed by forwarding its port with `kubectl port-forward service/longhorn-frontend 8080:80 -n longhorn-system` and visiting http://localhost:8080


## Setting up an Encrypted Storage Class

> This is following instructions from [here](https://longhorn.io/docs/archives/1.3.0/advanced-resources/security/volume-encryption/)

The [encrypted-storage-class.yaml](./encrypted-storage-class.yaml) deployment defines a kubernetes secret and an encrypted Longhorn storage class using the secret as its encryption key.

1. Decrypt `encrypted-storage/secrets/longhorn-encrypted-storage-secrets.yaml` with sops.
2. Apply with `kubectl`.

   ```bash
   kubectl apply --recursive -f encrypted-storage
   ```

3. Remove the current default storage class and set `longhorn-crypto-global` as default (see [docs](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/)).

```bash
# Mark current default as non-default.
kubectl patch storageclass longhorn -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Set the `longhorn-crypto-global` as default.
kubectl patch storageclass longhorn-crypto-global -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## Setting up snapshotting for use with VolSync.

I'm not sure why this is not coming with the helm chart, but that is for another day.

Apply these.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v8.3.0/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v8.3.0/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v8.3.0/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
```

Verify their installation.

```bash
kubectl get crd | grep snapshot.storage.k8s.io
```

Install the kubernetes cluster-wide snapshot controller.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v8.3.0/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v8.3.0/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```

Create a VolumeSnapshotClass

```bash
kubectl apply -f snapshot-functionality
```
