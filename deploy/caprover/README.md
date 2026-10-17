# The Bastion on CapRover

Deployment assets for running [The Bastion](https://ovh.github.io/the-bastion/) as a
CapRover app that acts as the `bastion` node fronting SSH access to the wider
infrastructure.

> **Trust boundary (accepted trade-off).** The bastion runs on the same control plane it
> protects. Anyone with root on the CapRover host, access to the Docker daemon, or a Swarm
> manager role can read every egress private key and every `ttyrec`. This bastion is **not**
> an independent security boundary against its own hosting plane. Accepted for this small
> setup — keep the blast radius in mind, and keep the break-glass path below.

## Files

| File | Purpose |
|------|---------|
| `captain-definition` | Points CapRover at `deploy/caprover/Dockerfile`. Build context = repo root. |
| `Dockerfile` | Production image (Debian bookworm). Runs install at **runtime**, not build. |
| `entrypoint.sh` | Restores persisted identity/host keys, runs new-install-or-upgrade, wires remote syslog, supervises sshd + syslog-ng + cron. |
| `service-update-override.json` | Host-mode SSH publish (preserves client IP), node pinning, ZeroTier caps. |

## Deploy

1. **Check out a release tag** (the image bakes in the uploaded tree):
   ```bash
   git -C /home/agents/agents/projects/the-bastion checkout v3.24.01-caprover.1
   ```
2. In CapRover, create an app, e.g. `bastion`. **Enable "Has Persistent Data".**
3. Set the app's **Captain Definition Path** to `./deploy/caprover/captain-definition`.
4. Add **persistent directories**, all under one volume root:
   - `/var/bastion-persist` → a named CapRover volume (e.g. `bastion-persist`)
   (the entrypoint lays out `home/ etc/ identity/ gnupg/ hostkeys/ logs/` inside it and
   binds the canonical paths).
5. Under **App Config**, set env vars (wired from the one-click spec's template vars):
   - `REMOTE_SYSLOG_HOST` — address of the syslog CapRover app (e.g. `srv-captain--syslog`)
   - `REMOTE_SYSLOG_PORT` — default `514`
   - `REMOTE_SYSLOG_PROTO` — `tcp` or `udp`
   Leave `REMOTE_SYSLOG_HOST` empty for local-only logging.
6. Set **"Do not expose as web-app"** (SSH is TCP; no HTTP route wanted).
7. Set **Instance Count = 1** (stateful; must never scale or reschedule freely).
8. Paste `service-update-override.json` into **Deployment → Service Update Override**,
   replacing `NODE_ID_HERE` with the target node id from `docker node ls`.
9. Deploy. Then **verify host-mode publish** and source-IP preservation:
   ```bash
   ssh -p 2222 <firstadmin>@<host>   # from an EXTERNAL host
   # on the bastion, confirm the logged source IP is the real client, not an overlay IP:
   grep -i 'from ' /var/log/bastion/bastion.log | tail
   ```

## Why host-mode publishing

CapRover's normal port mapping goes through the Swarm **ingress routing mesh**, which SNATs
the client address — the bastion would log the overlay IP for every connection, silently
breaking both audit trails and IP-based ingress restrictions (`ingressKeysFrom`,
`from="..."` key prefixes). `PublishMode: host` binds the port directly on the node running
the task, preserving the real source IP, and (bonus) exposes it only on that node — which
aligns with the single-node pinning. **Do not** also add a plain UI port mapping; the two
conflict.

## Persistence model (the single-file trap)

UNIX identity DBs (`/etc/passwd`, `shadow`, `group`, `gshadow`) **cannot** be bind-mounted:
shadow-utils writes a new file and renames it over the target, which breaks a symlink and can
`EBUSY` a single-file bind mount. So the entrypoint **snapshot-restores** them from
`identity/` on boot and **syncs back** after `install` and on shutdown. Whole directories
(`/home`, `/etc/bastion`, `/root/.gnupg`, `/var/log/bastion`) are bind-mounted normally.
SSH host keys are persisted too — otherwise every redeploy regenerates them and every user
gets a MITM warning. `install --new-install` (which regenerates host keys) runs **only** on a
truly fresh volume; every later boot is `install --upgrade --no-regen-hostkeys`.

### Acceptance test (run before trusting a redeploy)

```bash
# 1. create an account + group
ssh -p 2222 admin@host -- --osh accountCreate --account alice --uid 2001 ...
ssh -p 2222 admin@host -- --osh groupCreate   --group team --owner alice ...
# 2. note the host key fingerprint
ssh-keyscan -p 2222 host
# 3. redeploy the app in CapRover
# 4. confirm: alice + team still exist, host key fingerprint UNCHANGED
# 5. clean up: accountDelete / groupDelete
```

## Networking / egress (ZeroTier)

The bastion joins **ZeroTier** (`vpn.nativeit.net`, `10.242.0.x`) to reach other servers,
VMs, and networks — hence `CAP_NET_ADMIN` + `/dev/net/tun` in the override. Egress leaves
with the node/ZT identity, so target-side allowlisting is straightforward. Reaching "other
CapRover apps" over the overlay works by app name, but most app containers run no sshd —
realistic management is SSH to the CapRover **host** then `docker`, or adding sshd to
specific app containers only where truly wanted. Don't hand the bastion the Docker socket
just to manage containers.

## Break-glass (mandatory)

The bastion lives on the infra it protects. If Docker/Swarm/CapRover breaks, bastion access
dies with it. Keep **both** if possible:
- a key-only, IP-restricted **direct** host SSH on `:22` (host sshd stays on 22, bastion is 2222);
- provider console access.
Host sshd should eventually accept only the bastion's egress + this break-glass source.

## Swarm expansion (the planned second node)

The second "local mirror" is a Swarm **worker on a home/local LAN**, so the overlay would
cross the internet — **run Swarm over WireGuard between nodes** and keep `2377/tcp`,
`7946/tcp+udp`, `4789/udp` restricted to node IPs. CapRover's add-node flow does direct root
SSH to the new node (bypassing the bastion): allow it temporarily or `docker swarm join`
manually from inside a bastion session. A second node does **not** replicate this app's
node-local volumes — it is not HA. Real HA needs The Bastion's documented clustering.
