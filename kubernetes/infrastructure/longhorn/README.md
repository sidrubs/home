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
