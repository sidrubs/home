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

## Upgrading Talos

Get the correct image url from the [Talos image factory](https://factory.talos.dev). Make sure you add the correct extensions (see below),

```
Your image schematic ID is: 708747e350d604ae9e57227d8dcf274091453ddb1097b765d4ea8884f1992c1f

 customization:
    systemExtensions:
        officialExtensions:
            - siderolabs/iscsi-tools
            - siderolabs/tailscale
            - siderolabs/util-linux-tools
```

Check current version

```bash
talosctl version -n $HOMELAB_IP
```

Then run the following with the upgrade image path set to the correct version.

```bash
talosctl upgrade -n $HOMELAB_IP --image factory.talos.dev/metal-installer-secureboot/708747e350d604ae9e57227d8dcf274091453ddb1097b765d4ea8884f1992c1f:v1.11.2 --preserve
```

> `--preserve` ensures that custom mounts and ephemeral data is kept.

Upgrade to the latest version of `talosctl` to match the version of the cluster. This can be done with the following command.

```bash
curl -sL https://talos.dev/install | sh
```

## Upgrading Kubernetes

```bash
talosctl upgrade-k8s --to 1.34.1 -n $HOMELAB_IP
```

Upgrading kubectl

```bash
# Download binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Validate checksum
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# Install binary
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
