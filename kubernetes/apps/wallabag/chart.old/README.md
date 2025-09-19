# downloads

A Helm chart for downloads

## Installing the Chart

To install the chart with the release name `my-release`:

```bash
# Standard Helm install
$ helm install  my-release downloads

# To use a custom namespace and force the creation of the namespace
$ helm install my-release --namespace my-namespace --create-namespace downloads

# To use a custom values file
$ helm install my-release -f my-values.yaml downloads
```

See the [Helm documentation](https://helm.sh/docs/intro/using_helm/) for more information on installing and managing the chart.

## Configuration

The following table lists the configurable parameters of the downloads chart and their default values.

| Parameter                   | Default             |
| --------------------------- | ------------------- |
| `wallabag.imagePullPolicy`  | `IfNotPresent`      |
| `wallabag.replicas`         | `1`                 |
| `wallabag.repository.image` | `wallabag/wallabag` |
| `wallabag.repository.tag`   | ``                  |
| `wallabag.serviceAccount`   | ``                  |


