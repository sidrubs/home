# Traefik

Most of the instructions are from [here](https://doc.traefik.io/traefik/getting-started/quick-start-with-kubernetes/).

## Install Traefik

```bash
kubectl create namespace traefik

helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install -n traefik traefik traefik/traefik -f values.yaml
```
