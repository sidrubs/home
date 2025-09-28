# Longhorn

Most of the instructions are from [here](https://joshrnoll.com/installing-longhorn-on-talos-with-helm)

## Steps

Create namespace

```
kubectl apply -f namespace.yaml
```

Add longhorn repo

```
helm repo add longhorn https://charts.longhorn.io && helm repo update
```

Install on cluster

```
helm install longhorn longhorn/longhorn --namespace longhorn-system --values=values.yaml
```

> For now - while I don't have any load balancers, etc set up - the Longhorn UI can be accessed by forwarding its port with `kubectl port-forward service/longhorn-frontend 8080:80 -n longhorn-system` and visiting http://localhost:8080


## Setting up an Encrypted Storage Class

> This is following instructions from [here](https://longhorn.io/docs/archives/1.3.0/advanced-resources/security/volume-encryption/)

The [encrypted-storage-class.yaml](./encrypted-storage-class.yaml) deployment defines a kubernetes secret and an encrypted Longhorn storage class using the secret as its encryption key. 

1. Replace the `CRYPTO_KEY_VALUE: "<replace-with-passphrase>"` with an actual passphrase.
2. Apply with `kubectl`.

   ```bash
   kubectl apply -f encrypted-storage-class.yaml
   ```

3. Remove the current default storage class and set `longhorn-crypto-global` as default (see [docs](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/)).

```bash
# Mark current default as non-default.
kubectl patch storageclass longhorn -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Set the `longhorn-crypto-global` as default.
kubectl patch storageclass longhorn-crypto-global -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```


