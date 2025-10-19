# Traefik

Most of the instructions are from [here](https://doc.traefik.io/traefik/getting-started/quick-start-with-kubernetes/).

## Install Traefik

```bash
kubectl create namespace traefik

helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install -n traefik traefik traefik/traefik -f values.yaml
```

## Setting up HTTPS with a self signed certificate

This guide creates a **self-signed TLS certificate** for your internal reverse proxy serving local apps like
`https://wallabag.skuxnet`, `https://adguard.skuxnet`, etc.

It uses **ECDSA (P-256)** — a modern, efficient elliptic-curve algorithm that provides security equivalent to a 3072–4096-bit RSA key with smaller size and faster performance.

---

###  1. Generate the Private Key

```bash
openssl ecparam -genkey -name prime256v1 -out skuxnet.key
```
### 2. Create the Self-Signed Certificate

```bash
openssl req -new -x509 -key skuxnet.key -out skuxnet.crt \
  -days 3650 -sha256 \
  -subj "/C=CA/ST=Ontario/L=Toronto/O=Skuxnet Homelab/CN=*.skuxnet" \
  -addext "subjectAltName=DNS:skuxnet,DNS:*.skuxnet"
```

Explanation:

- days 3650 → Valid for 10 years
- sha256 → Uses SHA-256 for the signature
- subjectAltName → Ensures both skuxnet and any subdomain (*.skuxnet) are trusted

### 3. Verify the Certificate

```bash
openssl x509 -in skuxnet.crt -text -noout
```

You should see your Subject Alternative Names (SAN):

```
DNS:skuxnet, DNS:*.skuxnet
```
