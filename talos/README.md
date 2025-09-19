# Talos

## Set Up

- Replace the `install-image` with the appropriate Talos image that has secure boot and the necessary extensions.
- Add the tailscale auth key to the [tailscale patch](patches/tailscale.patch.yaml)

## Generate and Apply Config

Generate config a config file that can be applied to a Talos cluster.

```
export CLUSTER_NAME=skuxnet
export CONTROL_PLANE_IP=<controlplane-ip>

talosctl gen config $CLUSTER_NAME https://$CONTROL_PLANE_IP:6443 \
  --install-image=factory.talos.dev/metal-installer-secureboot/4a0d65c669d46663f377e7161e50cfd570c401f26fd9e7bda34a0216b6f1922b:v1.11.1  \
  --install-disk=/dev/sda \
  --with-secrets secrets.yaml
```

Apply the generated config to the Talos cluster.

```
talosctl apply-config --insecure --nodes $CONTROL_PLANE_IP --file controlplane.yaml
```

## Apply Patches

Apply the required patches to the Talos server. (`system-disk-encryption` should happen before `raw-volume`, `tailscale` should happen before `network-ingress`)

> Note `--mode=staged` means that the config will only be applied after reboot. This is required for disk encryption (and probably partitioning) patches.

```
talosctl patch mc --nodes $HOMELAB_IP --patch @patches/4-tailscale.yaml
```

> **Note:** After applying the tailscale patch use `talosctl edit machineconfig -n $HOMELAB_IP` to change the controlplane ip to match the tailnet ip address.
