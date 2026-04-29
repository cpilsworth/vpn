# tsexit

A throwaway DigitalOcean droplet that runs as a [Tailscale](https://tailscale.com) exit node.
Spin it up when you want VPN, tear it down when you don't.

The droplet is bootstrapped by [`cloud-config.yaml`](./cloud-config.yaml), which:

- installs Tailscale,
- joins the tailnet with a pre-auth key,
- advertises itself as an exit node + enables Tailscale SSH,
- locks the host firewall down so only Tailscale (UDP 41641) and tailnet-only
  SSH are reachable from the internet.

The [`vpn`](./vpn) script wraps `doctl` so you don't have to remember droplet
IDs or flags.

## Prereqs

```bash
brew install doctl
doctl auth init   # paste a token from cloud.digitalocean.com/account/api
doctl account get # confirm it works
```

You'll also want at least one SSH key registered with DigitalOcean if you ever
need console-less recovery — but it's optional, since the cloud-config enables
Tailscale SSH.

### Tailscale auth key (kept out of git)

The committed [`cloud-config.yaml`](./cloud-config.yaml) contains a placeholder,
not a real key:

```yaml
--auth-key=__TS_AUTHKEY__
```

`vpn up` substitutes the real key into a tempfile at create time. The script
finds the key via, in order:

1. `$TS_AUTHKEY` env var
2. macOS Keychain item named `vpn-tailscale-authkey` (override with
   `VPN_AUTHKEY_KEYCHAIN_ITEM`)

Recommended one-time setup using Keychain:

```bash
security add-generic-password \
  -s vpn-tailscale-authkey \
  -a "$USER" \
  -w 'tskey-auth-...'   # paste the real key here
```

Generate the key in the Tailscale admin console at
<https://login.tailscale.com/admin/settings/keys>. Recommended flags:
**reusable** (so the same key works across many spin-ups) and **ephemeral**
(so old droplets disappear from your device list automatically when destroyed).

This means the repo is safe to push to a public GitHub — the cloud-config
contains only the placeholder.

## Usage

```bash
./vpn up      # create the exit-node droplet (idempotent)
./vpn status  # show whether it exists, with IP / region / created-at
./vpn down    # destroy it
```

`up` returns as soon as DigitalOcean reports the VM "active". Cloud-init keeps
running for ~30–90 s after that — that's when Tailscale installs and joins.
Confirm it appears on the tailnet from any client:

```bash
tailscale status | grep tailnet-exit-ams
```

Once it's listed, the exit node is available to every device on your tailnet.
Each device chooses whether to route through it on its own (Tailscale GUI →
Exit nodes, or `tailscale set --exit-node=tailnet-exit-ams` on the CLI) — that
isn't this script's job.

## Configuration

All defaults are overridable via env vars:

| Var | Default | Notes |
|---|---|---|
| `VPN_NAME` | `tailnet-exit-ams` | Droplet name + Tailscale hostname |
| `VPN_REGION` | `ams3` | `doctl compute region list` for options |
| `VPN_SIZE` | `s-1vcpu-512mb-10gb` | $4/mo, smallest tier. `doctl compute size list` |
| `VPN_IMAGE` | `ubuntu-24-04-x64` | Must support cloud-init |
| `VPN_TAG` | `vpn-exit-node` | How the script finds "its" droplet |
| `VPN_SSH_KEYS` | `ssh` | Comma-separated DigitalOcean key names, MD5 fingerprints, or numeric IDs. Names are resolved via `doctl compute ssh-key list` (token needs `ssh_key:read`). |
| `VPN_CLOUD_CONFIG` | `./cloud-config.yaml` | Path to the cloud-init file |
| `VPN_ENABLE_PRIVATE_NETWORKING` | `0` | Set `1` only if other droplets in the same region need to reach this one over the private VPC |

Defaults mirror the reference droplet `ubuntu-s-1vcpu-512mb-10gb-ams3`. Example
override:

```bash
VPN_REGION=fra1 VPN_SIZE=s-1vcpu-1gb ./vpn up
```

## Why "destroy" instead of "stop"

DigitalOcean **bills powered-off droplets at the same rate as running ones** —
you're paying for the reserved disk + IP. To actually stop paying you have to
destroy the droplet. The cloud-config takes ~2 min to bring a fresh node up,
which is fine for a personal VPN.

If you'd rather keep the disk around (faster restart, fixed IP), the
alternatives are: snapshot + destroy + recreate from snapshot (cheap, slow),
or reserve a static IP separately. Out of scope for this script.

## Notes & gotchas

- **The filename is arbitrary, but the first line matters.** Cloud-init
  identifies the file by its `#cloud-config` magic header on line 1, not by
  its filename. Don't remove that line.

- **Rotate the original key before pushing.** The first version of
  `cloud-config.yaml` in your local working tree had a real `tskey-auth-...`
  in plaintext. Even though the placeholder version is what gets committed,
  you should still treat the original key as compromised and revoke it in
  the Tailscale admin console — anywhere that string was pasted (clipboard
  history, scrollback, this Claude Code session) is a leak vector.

- **`vpn up` is idempotent by tag.** If a droplet with `VPN_TAG` already
  exists, it just prints the IP — it won't create a duplicate. So if you
  want a fresh one, `vpn down && vpn up`.

- **DigitalOcean's `--ssh-keys` flag only accepts IDs or MD5 fingerprints**,
  not names. The script works around this by calling `doctl compute ssh-key
  list` and resolving the name itself, but that requires the API token to
  include the `ssh_key:read` scope. If your token doesn't (you'll get a 403
  on `doctl compute ssh-key list`), pass the fingerprint directly:

  ```bash
  # compute MD5 fingerprint locally — matches what DigitalOcean stores
  ssh-keygen -E md5 -lf ~/.ssh/id_ed25519.pub | awk '{print $2}' | sed 's/MD5://'
  # then:
  VPN_SSH_KEYS=aa:bb:cc:... ./vpn up
  ```
