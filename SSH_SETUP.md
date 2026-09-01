# SSHing into this container over Tailscale

Confirmed working: `ssh root@<tailscale-ip>` (and `ssh empty-cloud-container`
via MagicDNS) from another device on the tailnet. Builds on
`TAILSCALE_SETUP.md`.

## Approach used: Tailscale SSH

Tailscale's built-in SSH server, not `openssh-server`. It authenticates via
tailnet identity (no SSH keys/passwords to manage) — a better fit for an
ephemeral container with no durable place to keep host keys or
`authorized_keys` across restarts.

### Steps

1. Install and start `tailscaled` (see `TAILSCALE_SETUP.md` steps 1-2).
2. Bring the node up with SSH enabled:
   ```sh
   tailscale --socket=/var/run/tailscale/tailscaled.sock up \
     --hostname=empty-cloud-container --ssh
   ```
3. Approve the node via the printed login URL, same as before.
4. Confirm SSH is actually enabled on the node:
   ```sh
   tailscale --socket=/var/run/tailscale/tailscaled.sock debug prefs \
     | grep RunSSH
   # "RunSSH": true
   ```
5. From another tailnet device:
   ```sh
   ssh root@<tailscale-ip>
   # or, with MagicDNS:
   ssh empty-cloud-container
   ```
   This connected successfully as `root` with no extra ACL changes needed —
   the tailnet's default policy already permitted it. If a tailnet has a
   locked-down policy file, SSH access may need an explicit `ssh` grant in
   Access Controls (https://login.tailscale.com/admin/acls), e.g.:
   ```json
   "ssh": [
     {
       "action": "accept",
       "src":    ["autogroup:member"],
       "dst":    ["tag:cloud-container"],
       "users":  ["root", "autogroup:nonroot"]
     }
   ]
   ```

## Alternative (not used): run `openssh-server` directly

Install `openssh-server`, generate host keys, bind `sshd` to `0.0.0.0:22`
(or the tailnet interface), and reach it at `<tailscale-ip>:22`. Rejected
in favor of Tailscale SSH because host keys and `authorized_keys` would
need to be re-provisioned on every fresh container (no durable storage),
whereas Tailscale SSH has no per-container credential material at all.

## Caveat carried over from `TAILSCALE_SETUP.md`

Node identity lives in `/var/lib/tailscale/tailscaled.state`, which does
not survive a container restart. A fresh container needs a fresh
`tailscale up --ssh` and a new approval, and will appear as a new tailnet
device (confirmed happening once already this session — a restart wiped
the earlier registration and required re-approving).
