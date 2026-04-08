# Talos

Cluster configuration is managed with [talhelper](https://github.com/budimanjojo/talhelper). The source of truth is [`little-hp/talconfig.yaml`](little-hp/talconfig.yaml). Generated machine configs are not committed (gitignored by talhelper).

## Requirements

- [talosctl](https://github.com/siderolabs/talos)
- [talhelper](https://github.com/budimanjojo/talhelper)
- [sops](https://github.com/getsops/sops)
- [age](https://github.com/FiloSottile/age)

## Initial Setup (already done)

Generate a secrets bundle and encrypt it with SOPS:

```bash
cd little-hp/
talhelper gensecret > talsecret.sops.yaml
sops -e -i talsecret.sops.yaml
```

## Generate and Apply Config

Generate machine configs from `talconfig.yaml`:

```bash
cd little-hp/
talhelper genconfig
```

Apply config to the node (use `--insecure` on first boot before Talos is fully configured):

```bash
cd little-hp/
talhelper gencommand apply --extra-flags --insecure | bash
```

For subsequent applies (node already running):

```bash
cd little-hp/
talhelper gencommand apply | bash
```

## Bootstrap

After applying config, bootstrap the cluster:

```bash
cd little-hp/
talhelper gencommand bootstrap | bash
```

Then fetch the kubeconfig:

```bash
cd little-hp/
talhelper gencommand kubeconfig | bash
```

## Upgrading Talos

Update `talosVersion` in `talconfig.yaml`, regenerate configs, then:

```bash
cd little-hp/
talhelper genconfig
talhelper gencommand upgrade --extra-flags "--preserve" | bash
```

## Kubelet Serving Certificate CSRs

The kubelet is configured with `serverTLSBootstrap: true`, which means it requests a serving certificate from the Kubernetes CA rather than using a self-signed cert. These CSRs are not auto-approved and require manual approval.

If `kubectl logs`, `kubectl port-forward` or `kubectl exec` stop working, check for pending CSRs:

```bash
kubectl get csr
```

Approve any pending `kubernetes.io/kubelet-serving` CSRs:

```bash
kubectl certificate approve <csr-name>
```

Note: kubelet serving certs rotate automatically before expiry, so this will need to be repeated periodically.

## Upgrading Kubernetes

Update `kubernetesVersion` in `talconfig.yaml`, then:

```bash
cd little-hp/
talhelper gencommand upgrade-k8s | bash
```
