# Applications

## Initialization

Create `apps` namespace.

```bash
kubectl apply -f apps-namespace.yaml
```

## Adding Applications

Just apply the directory.

E.g:

```
kubectl apply -f wallabag -n apps
```
