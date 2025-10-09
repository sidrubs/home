# VolSync

The installation instructions are taken from [here](https://volsync.readthedocs.io/en/stable/installation/index.html).

## Steps

Add the Backube Helm repo

```bash
helm repo add backube https://backube.github.io/helm-charts/
```

Deploy the chart in your cluster

```bash
helm install --create-namespace -n volsync-system volsync backube/volsync
```
