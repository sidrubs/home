# AdGuard

> **Note:** For first boot ever it seems that the setup dashboard is accessible on port 3000. After that the web interface is accessible on port 80.

## DNS Rewrite instructions

To allow wildcard subdomains for the reverse proxy. Log into AdGuard web interface and configure DNS rewrite rules:

1. Go to **Filters → DNS rewrites**
2. Add these rewrites:
   ```
   *.skuxnet → 100.83.86.86  (wildcard for future apps)
   ```

> **Note:** `100.83.86.86` is the tailnet IP address of the reverse proxy.
