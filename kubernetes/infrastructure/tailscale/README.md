# Tailscale

Mostly following [these](https://tailscale.com/learn/managing-access-to-kubernetes-with-tailscale) instructions to install the Tailscale operator.

## Install Tailscale

Create namespace.

```bash
kubectl apply -f namespace.yaml
```

Create an AuthKey on the [Tailscale dashboard](https://login.tailscale.com/admin/settings/keys). Add it to [tailscale-auth-secrets.yaml](./secrets/tailscale-auth-secrets.yaml).

Apply the secret.

```bash
kubectl -n tailscale apply -f secrets/tailscale-auth-secrets.yaml
```

Next, you must create a Kubernetes service account, role, and role binding to configure role-based access control (RBAC) for your Tailscale deployment. You'll run your Tailscale pods as this new service account. The pods will be able to use the granted RBAC permissions to perform limited interactions with your cluster.

See `tailscale-rbac.yaml`.

The role allows your tailscale service account to retrieve and modify the tailscale-auth secret you created above, which is required for Tailscale to function.

Use kubectl to create the Service Account, Role, and RoleBinding in your cluster:

```bash
kubectl -n tailscale apply -f tailscale-rbac.yaml
```

In your tailnet policy file, create the tags tag:k8s-operator and tag:k8s, and make tag:k8s-operator an owner of tag:k8s. If you want your Services to be exposed with tags other than the default tag:k8s, create those as well and make tag:k8s-operator an owner.

```json
"tagOwners": {
   "tag:k8s-operator": [],
   "tag:k8s": ["tag:k8s-operator"],
}
```

Create an OAuth client in the OAuth clients page of the admin console. Create the client with Devices Core and Auth Keys write scopes, and the tag `tag:k8s-operator`.

```bash
# Add the repository
helm repo add tailscale https://pkgs.tailscale.com/helmcharts

# Update your client’s package list
helm repo update
```

Install the operator

```bash
helm upgrade --install tailscale-operator tailscale/tailscale-operator \
  --namespace tailscale \
  -f values.yaml \
  --wait
```
