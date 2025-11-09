# VolSync

The installation instructions are taken from [here](https://volsync.readthedocs.io/en/stable/installation/index.html).

## Steps

Deploy the chart in your cluster

```bash
# Update dependencies (downloads the chart version specified in Chart.yaml)
helm dependency update

# Install from the local chart with dependencies
helm install --create-namespace -n volsync-system volsync .
```
