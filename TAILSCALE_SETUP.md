# Serving `index.html` from this container over Tailscale

Steps used to get this cloud container onto a Tailscale tailnet and serving
`index.html` to other devices on that tailnet.

## 1. Install Tailscale

```sh
curl -fsSL https://tailscale.com/install.sh | sh
```

## 2. Start `tailscaled`

This container has no `systemd` (PID 1 isn't systemd), so the daemon is
started directly instead of via `systemctl`:

```sh
mkdir -p /var/lib/tailscale /var/run/tailscale
tailscaled \
  --state=/var/lib/tailscale/tailscaled.state \
  --socket=/var/run/tailscale/tailscaled.sock &
```

Confirmed prerequisites: `CAP_NET_ADMIN` present and `/dev/net/tun` available
(check via `grep Cap /proc/self/status` and `ls /dev/net/tun`) — without
these, `tailscaled` can't create the `tailscale0` interface.

## 3. Authenticate the node

```sh
tailscale --socket=/var/run/tailscale/tailscaled.sock up \
  --hostname=empty-cloud-container
```

This prints a `https://login.tailscale.com/a/...` URL. Open it on an
already-authenticated device/browser to approve the node into the tailnet.

**Gotcha:** if the container's outbound network proxy cycles/restarts at the
same moment you approve the link, the registration call can fail
(`tailscaled` falls back to "logged out") even though you clicked approve.
If `tailscale status` still shows `Logged out` afterwards, just re-run
`tailscale up` for a fresh link and approve that one instead.

Verify with:

```sh
tailscale --socket=/var/run/tailscale/tailscaled.sock status
```

which lists the node's tailnet IP (e.g. `100.86.250.104`) alongside your
other devices.

## 4. Serve the page

```sh
python3 -m http.server 8000 --bind 0.0.0.0
```

**Gotcha:** binding to `127.0.0.1` (the default in some snippets) only
accepts local connections — it will *not* be reachable over the Tailscale
interface. Bind to `0.0.0.0` (or the Tailscale interface's address).

Note: `tailscale serve` (which gives a clean HTTPS URL with no port) needs
"HTTPS Certificates" enabled in the tailnet's admin console
(Settings → https://login.tailscale.com/admin/dns) — a separate toggle from
just joining the tailnet. We didn't have that enabled, so we served plain
HTTP directly on the node's tailnet IP instead.

## 5. Access from another device on the tailnet

```
http://<tailscale-ip>:8000/index.html
```

or, with MagicDNS enabled:

```
http://<hostname>:8000/index.html
```

## Caveats for this environment

- This is an ephemeral cloud container: its filesystem (including
  `/var/lib/tailscale/tailscaled.state`, which holds the node's identity
  key) does not persist across container restarts. A new container means a
  new node identity and a fresh `tailscale up` approval — it will not
  resume as the same tailnet device. Stale old nodes should be removed from
  the tailnet admin console periodically.
- Nothing here is started via a supervisor, so `tailscaled` and the HTTP
  server will not restart automatically if the container is recycled.
