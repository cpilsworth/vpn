# vpn

A script to manage a throwaway DigitalOcean droplet that runs as a [Tailscale](https://tailscale.com) exit node.
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

You'll need:

- **A [DigitalOcean](https://www.digitalocean.com) account** — the droplet host
  and what you'll be billed by (~$0.006/hr while running, $4/mo flat if you
  ever leave one up). Generate a personal access token at
  <https://cloud.digitalocean.com/account/api/tokens> with the
  `droplet:read,create,delete` scopes. Add `ssh_key:read` if you want the
  script to resolve SSH keys by name (see `VPN_SSH_KEYS` below).
- **A [Tailscale](https://tailscale.com) account** — what makes the droplet
  reachable as an exit node (free tier is fine for personal use). You'll
  generate a pre-auth key from
  <https://login.tailscale.com/admin/settings/keys> when setting up secrets
  (see [Tailscale auth key](#tailscale-auth-key-kept-out-of-git) below).
- **An SSH key registered with DigitalOcean** named `ssh`, if you want one
  baked into the droplet for emergency console-less recovery. Optional —
  the cloud-config enables Tailscale SSH, which is enough for everything.

Then install and authenticate the CLI:

```bash
brew install doctl
doctl auth init           # paste your DigitalOcean token
doctl compute droplet list   # confirm it works (lists your droplets)
```

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

Recommended setup using Keychain (works for both initial install and key
rotation — `-U` updates an existing entry, and omitting the value after `-w`
makes `security` prompt for it interactively so the key never lands in your
shell history):

```bash
security add-generic-password -U -s vpn-tailscale-authkey -a "$USER" -w
# (you'll be prompted twice: enter the tskey-auth-... key, then confirm)
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

## Triggering from your phone (GitHub Actions)

[`.github/workflows/vpn.yml`](.github/workflows/vpn.yml) lets you run
`./vpn up|down|status` on a GitHub-hosted runner on demand. One-tap from an
iOS Shortcut, no always-on box of your own.

### One-time setup

1. Push this repo to a **private** GitHub repository (the cloud-config only
   contains the placeholder, but a private repo is still cleaner).
2. Add two repository secrets at `Settings → Secrets and variables → Actions`:
   - `DIGITALOCEAN_ACCESS_TOKEN` — needs scopes `droplet:read`, `droplet:create`,
     `droplet:delete`. Generate at
     <https://cloud.digitalocean.com/account/api/tokens>.
   - `TS_AUTHKEY` — your Tailscale pre-auth key. Use a **reusable + ephemeral**
     key from <https://login.tailscale.com/admin/settings/keys> so the runner
     can spin up a fresh node every time without the key burning out, and old
     droplets fall out of the device list automatically.
3. Confirm it works once from the GitHub UI: `Actions → vpn → Run workflow →
   action: status`.

### Trigger from the GitHub mobile app

`Actions` tab → `vpn` workflow → `Run workflow` → choose `up` / `down` /
`status`. Works but takes several taps.

### Trigger from a terminal (`gh` CLI)

```bash
gh workflow run vpn.yml -f action=up      # or down / status
gh run watch                              # tail the latest run live
gh run view --log                         # dump the full log of the latest run
```

### Trigger from an iOS Shortcut (one tap)

Create a shortcut with a single **Get Contents of URL** action:

- URL: `https://api.github.com/repos/<USER>/<REPO>/actions/workflows/vpn.yml/dispatches`
- Method: `POST`
- Headers:
  - `Authorization`: `Bearer <FINE_GRAINED_PAT>`
  - `Accept`: `application/vnd.github+json`
  - `X-GitHub-Api-Version`: `2022-11-28`
- Request body (JSON):
  ```json
  { "ref": "main", "inputs": { "action": "up" } }
  ```

Make a fine-grained personal access token at
<https://github.com/settings/personal-access-tokens> scoped to **only this repo**
with the `Actions: read and write` permission. Duplicate the shortcut for
`up` / `down` / `status` and pin them to your home screen.

The runner's job log shows the droplet's public IP. To read it from the phone,
use the GitHub mobile app or hit the GitHub REST API for the latest run.

## Why "destroy" instead of "stop"

DigitalOcean **bills powered-off droplets at the same rate as running ones** —
you're paying for the reserved disk + IP. To actually stop paying you have to
destroy the droplet. The cloud-config takes ~2 min to bring a fresh node up,
which is fine for a personal VPN.

If you'd rather keep the disk around (faster restart, fixed IP), the
alternatives are: snapshot + destroy + recreate from snapshot (cheap, slow),
or reserve a static IP separately. Out of scope for this script.
