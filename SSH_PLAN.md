# Plan: SSH into this container over Tailscale

Goal: reach an interactive shell on this container from another device on
the tailnet, instead of serving a static page. Builds on
`TAILSCALE_SETUP.md`.

## Chosen approach: Tailscale SSH

Use Tailscale's built-in SSH server rather than running `openssh-server`.
It authenticates via tailnet identity (no SSH keys/passwords to manage) and
access is governed by the tailnet's `ssh` ACL policy, which is a better fit
for an ephemeral container that has no durable place to keep host keys or
`authorized_keys` across restarts.

### Steps

1. Install and start `tailscaled` as before (`TAILSCALE_SETUP.md` steps 1-2).
2. Bring the node up with SSH enabled:
   ```sh
   tailscale --socket=/var/run/tailscale/tailscaled.sock up \
     --hostname=empty-cloud-container --ssh
   ```
3. Approve the node via the printed login URL, same as before.
4. Confirm the tailnet's ACL policy (admin console → Access controls)
   permits SSH from the connecting user/device to this node. By default a
   tailnet's ACLs may not grant `ssh` access even if general connectivity
   is allowed — this is a separate grant from plain TCP reachability, and
   may need adding an `ssh` block, e.g.:
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
   (exact rule depends on what's already in the policy file — needs review
   before assuming this step is a no-op).
5. From another tailnet device:
   ```sh
   ssh empty-cloud-container
   ```
   or via tailnet IP: `ssh root@100.86.250.104` (or whichever OS user).

### Open questions / decisions needed

- **Which user to log in as.** This container currently runs everything as
  `root`. Logging in as root over SSH is broad; may want a non-root user
  created first and login restricted to it via the `ssh` ACL `users` field.
- **ACL policy access.** Editing tailnet ACLs requires access to the
  Tailscale admin console — that's on the user's end, not something doable
  from inside this container.

## Alternative (not chosen): run `openssh-server` directly

Install `openssh-server`, generate host keys, bind `sshd` to `0.0.0.0:22`
(or the tailnet interface), and reach it at `<tailscale-ip>:22`. Requires
managing `authorized_keys` for a login user and, without a `ssh` ACL, any
device that can reach port 22 on this node could attempt to authenticate —
so an ACL restricting port 22 access would still be worth adding. Rejected
in favor of Tailscale SSH primarily because host keys and
`authorized_keys` would need to be re-provisioned on every fresh container
(no durable storage), whereas Tailscale SSH has no such per-container
credential material.

## Caveat carried over from `TAILSCALE_SETUP.md`

Same ephemeral-container caveat applies: node identity lives in
`/var/lib/tailscale/tailscaled.state`, which does not survive a container
restart. A fresh container needs a fresh `tailscale up --ssh` and a new
approval, and will appear as a new tailnet device.
