# Podman systemd services

This repository contains systemd user units for running a small monitoring stack
with rootless Podman. The units are configured for the `ec2-user` account and use
absolute paths under `/home/ec2-user` for the monitoring stack. The No-IP unit
uses `%h` to resolve the home directory of the user running the service.

## Services

### Signal API

- Unit: `signal-api.service`
- Image: `docker.io/bbernhard/signal-cli-rest-api:latest`
- Mode: `json-rpc-native` keeps a low-latency daemon running so alert requests
  complete within Gatus's HTTP timeout
- Access: internal `monitoring` network only, using the `signal-api` alias
- Data: `/home/ec2-user/signal-api` mounted at
  `/home/.local/share/signal-cli`

### Gatus

- Unit: `container-gatus.service`
- Image: `docker.io/twinproduction/gatus:latest`
- Access: `https://gatus.ddns.net`, through Caddy
- Transport: HTTP on container port `8080`, internal `monitoring` network only
- Data: `/home/ec2-user/gatus/config`, `/home/ec2-user/gatus/data`, and
  the configured CA bundle

### Gatus HTTPS proxy

- Unit: `container-gatus-caddy.service`
- Image: `docker.io/library/caddy:2`
- Access: host TCP `443`, mapped to Caddy port `443`
- Configuration: `/home/ec2-user/gatus/caddy/Caddyfile` from the Gatus repository
- Host storage: `~/gatus-caddy-data/data` (certificates, keys, ACME state) and
  `~/gatus-caddy-data/config` (Caddy runtime configuration), bind-mounted with
  SELinux labels into `/data` and `/config`
- Issuance and renewal: Let's Encrypt TLS-ALPN challenge on TCP `443`

Keep `gatus.ddns.net` pointed at this host's public IP. Allow inbound TCP 443
in its firewall and cloud security group. Port 443 must reach Caddy directly
for the TLS-ALPN challenge; a TLS-terminating load balancer or proxy in front
will prevent validation. Port 80 is not needed or published, and HTTP redirects
are disabled. Use the HTTPS URL explicitly. Caddy renews certificates while
running; no Certbot or renewal cron job is required. See
[Caddy automatic HTTPS](https://caddyserver.com/docs/automatic-https).

#### Migrate an existing Gatus service

Place `caddy/Caddyfile` from the Gatus repository at
`/home/ec2-user/gatus/caddy/Caddyfile` before starting the proxy. From this
repository, run as `ec2-user`:

The unit creates both storage directories with mode 0700 before startup, using
`%h` to resolve the service user's home directory. The source Caddyfile remains
in the Gatus checkout. If Caddy already ran with the previous named volumes,
stop `container-gatus-caddy.service` and copy each volume's contents into the
corresponding host directory before restarting, preserving ownership and file
permissions. Keep the old volumes until HTTPS works with the migrated data to
avoid losing existing certificates and ACME account state.

```sh
test -f /home/ec2-user/gatus/caddy/Caddyfile
# Validate configuration without requesting a certificate.
podman run --rm \
  -v /home/ec2-user/gatus/caddy/Caddyfile:/etc/caddy/Caddyfile:ro,Z \
  docker.io/library/caddy:2 caddy validate --config /etc/caddy/Caddyfile
```

After validation succeeds, install both units and restart Gatus to release its
old port 443 mapping. This briefly interrupts dashboard access:

```sh
install -m 0644 container-gatus.service container-gatus-caddy.service \
  /home/ec2-user/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user restart container-gatus.service
systemctl --user enable --now container-gatus-caddy.service
journalctl --user -u container-gatus-caddy.service -n 50
```

From an external client, open `https://gatus.ddns.net` and verify the certificate
is trusted, covers that hostname, and was issued by Let's Encrypt. Logs should
report successful certificate issuance. If issuance fails, inspect DNS, TCP 443
reachability, and Caddy logs before repeatedly restarting the service.

### Uptime Kuma

- Unit: `uptime-kuma.service`
- Image: `docker.io/louislam/uptime-kuma:latest`
- Access: `http://HOST:3001`
- Data: `/home/ec2-user/uptime-kuma` mounted at `/app/data`

All monitoring containers and Caddy join the rootless Podman network named `monitoring`.
Gatus and Uptime Kuma declare ordering and soft dependencies on the Signal API:
systemd starts `signal-api.service` first, but a Signal API failure does not stop
the monitoring services from starting. The Signal API is not published on a host
port; other containers reach it by the `signal-api` network alias.

The units use `--rm` and `--replace`, so systemd recreates their containers when
they start. Persistent state remains in the host directories listed above.

### No-IP Dynamic DNS

- Unit: `noip-duc.service`
- Image: `ghcr.io/noipcom/noip-duc:latest`
- Configuration: `~/noip/noip-duc.env` on the host (`%h/noip/noip-duc.env`
  in the unit)
- Network: Podman's default rootless network; no published ports or dependency
  on the monitoring stack

The [official No-IP image](https://github.com/noipcom/linux-update-client-docker)
accepts configuration as environment variables. Podman reads the host file with
`--env-file`; no configuration bind mount is needed.

Run these commands as the host user who will own the service, from this
repository:

```sh
install -d -m 0700 "$HOME/noip"
```

For a new configuration, copy the template (skip this command if the destination
already exists):

```sh
install -m 0600 noip-duc.env.example "$HOME/noip/noip-duc.env"
```

Edit `~/noip/noip-duc.env`, replacing `NOIP_USERNAME` and `NOIP_PASSWORD` with
your No-IP DDNS Key credentials. Keep `NOIP_HOSTNAMES=all.ddnskey.com` when using
a DDNS Key. For account credentials instead, set `NOIP_HOSTNAMES` to the actual
hostname(s), comma-separated. Use plain `KEY=value` lines without `export` or
surrounding quotes; Podman reads the file directly rather than through a shell.
Never commit the populated file.

Install and start the service after filling in the credentials:

```sh
chmod 0600 "$HOME/noip/noip-duc.env"
install -d -m 0755 "$HOME/.config/systemd/user"
install -m 0644 noip-duc.service "$HOME/.config/systemd/user/"
systemctl --user daemon-reload
systemctl --user enable --now noip-duc.service
systemctl --user --no-pager --full status noip-duc.service
journalctl --user -u noip-duc.service -n 50
```

Confirm the logs show a successful DNS update or an already-current address.
To keep it running after logout and start at boot, enable lingering for the
service user:

```sh
sudo loginctl enable-linger "$(id -un)"
```

After changing the configuration, run
`systemctl --user restart noip-duc.service`. If startup fails, check that the
config exists, credentials are valid, and the host can reach GHCR and No-IP.

## Prerequisites

- A Linux host running systemd with user services
- Podman installed at `/usr/bin/podman`
- The `ec2-user` account, normally with UID and GID 1000 as required by the
  Signal API unit
- Permission to expose TCP ports 443 and 3001 on the host, including firewall
  and security-group rules where applicable
- Rootless binding to port 443 permitted by the host's
  `net.ipv4.ip_unprivileged_port_start` setting (443 or lower)
- SELinux configuration that permits the labeled bind mounts, if SELinux is
  enabled

Run the Podman and systemd commands below as `ec2-user`. Do not install these
units as root or under `/etc/systemd/system`; `%t/containers` and
`default.target` are intended for the user's systemd manager.

## Prepare the host

Before starting Gatus, check the host's rootless port threshold:

```sh
sysctl net.ipv4.ip_unprivileged_port_start
```

If the value exceeds 443, an administrator must configure the host to permit
rootless binding to that port. For example, persist
`net.ipv4.ip_unprivileged_port_start=443` in a file under `/etc/sysctl.d/` and
apply it with `sudo sysctl --system`. This host-wide setting allows all
unprivileged users to bind ports 443 and above. See the
[Podman rootless limitations](https://github.com/containers/podman/blob/main/rootless.md).

Create the shared network and persistent directories:

```sh
podman network exists monitoring || podman network create monitoring

install -d -m 0700 \
  /home/ec2-user/gatus/config \
  /home/ec2-user/gatus/data \
  /home/ec2-user/gatus/certs \
  /home/ec2-user/signal-api \
  /home/ec2-user/uptime-kuma
```

Create `/home/ec2-user/gatus/.env` with the environment required by the local
Gatus configuration. This repository does not define those deployment-specific
values. The file must exist because the Gatus unit loads it through both
systemd's `EnvironmentFile` directive and Podman's `--env-file` option.

```sh
install -m 0600 /dev/null /home/ec2-user/gatus/.env
```

If the file already exists, do not run that command because it replaces its
contents. Instead, confirm its permissions:

```sh
chmod 0600 /home/ec2-user/gatus/.env
```

Place the required Gatus configuration under
`/home/ec2-user/gatus/config`. Also provide a PEM CA bundle at
`/home/ec2-user/gatus/certs/ca-certificates.crt`; the unit mounts that exact
file over the container's `/etc/ssl/certs/ca-certificates.crt`. How the bundle
is assembled depends on the deployment's trust requirements.

Place the Gatus repository's `caddy/Caddyfile` at
`/home/ec2-user/gatus/caddy/Caddyfile`. The service creates its host storage under
`~/gatus-caddy-data` on first start. Preserve it when recreating containers.

Ensure all persistent files belong to `ec2-user`. The Signal API container maps
its application UID and GID to 1000, matching the values embedded in its unit.

## Install and start

From this repository, install the units into the user systemd directory:

```sh
install -d -m 0755 /home/ec2-user/.config/systemd/user
install -m 0644 \
  signal-api.service \
  container-gatus.service \
  container-gatus-caddy.service \
  uptime-kuma.service \
  /home/ec2-user/.config/systemd/user/

systemctl --user daemon-reload
```

Enable the Signal API first, followed by its consumers:

```sh
systemctl --user enable --now signal-api.service
systemctl --user enable --now container-gatus.service uptime-kuma.service
systemctl --user enable --now container-gatus-caddy.service
```

Enabling the units makes them start with the user's `default.target`. To keep
the user manager and containers running when `ec2-user` is logged out, an
administrator can enable lingering:

```sh
sudo loginctl enable-linger ec2-user
```

Confirm the services and containers are running:

```sh
systemctl --user --no-pager --full status \
  signal-api.service container-gatus.service container-gatus-caddy.service uptime-kuma.service
podman ps
```

## Operations

View recent logs or follow a service:

```sh
journalctl --user -u container-gatus.service -n 100
journalctl --user -u signal-api.service -f
journalctl --user -u uptime-kuma.service -f
```

Restart the stack in dependency order:

```sh
systemctl --user restart signal-api.service
systemctl --user restart container-gatus.service uptime-kuma.service
```

Stop all services, with the consumers first:

```sh
systemctl --user stop container-gatus-caddy.service
systemctl --user stop container-gatus.service uptime-kuma.service
systemctl --user stop signal-api.service
```

After changing an installed unit, copy it back into
`/home/ec2-user/.config/systemd/user`, reload the user manager, and restart the
affected service:

```sh
systemctl --user daemon-reload
systemctl --user restart UNIT.service
```

## Container updates

Each container has the `io.containers.autoupdate=registry` label and uses a
mutable image tag (`2` for Caddy, `latest` for the other services). Review
available updates before applying them:

```sh
podman auto-update --dry-run
podman auto-update
```

Where the Podman package provides the user auto-update timer, it can run these
checks periodically:

```sh
systemctl --user enable --now podman-auto-update.timer
systemctl --user list-timers podman-auto-update.timer
```

Automatic updates can change application behavior because the image tags are
not pinned to digests. Back up persistent data and review upstream release notes before
enabling unattended updates.

## Troubleshooting

If a unit fails, inspect both systemd and Podman state:

```sh
systemctl --user --no-pager --full status UNIT.service
journalctl --user -u UNIT.service -b
podman ps -a
podman network inspect monitoring
```

Common causes include:

- **No user bus:** log in as `ec2-user`, or enable lingering and retry the
  `systemctl --user` command with that user's environment.
- **Missing network:** create the rootless `monitoring` network as `ec2-user`.
  A network created by root belongs to a different Podman storage context.
- **Gatus fails before Podman starts:** verify that
  `/home/ec2-user/gatus/.env` exists and is readable.
- **Bind-mount failure:** verify every host path exists, has appropriate
  ownership and permissions, and has a usable SELinux label where applicable.
- **Port conflict:** check whether another process already listens on TCP 443
  or 3001. Restart Gatus with the updated unit to release its previous port 443
  mapping before starting Caddy.
- **Certificate issuance failure:** check public DNS and inbound TCP 443,
  including cloud security groups. Ensure no TLS-terminating proxy intercepts
  Caddy's TLS-ALPN challenge. Inspect `container-gatus-caddy.service` logs.
- **HTTPS returns 502:** verify Gatus is running and both containers join
  `monitoring`; Caddy must resolve `gatus` and reach its port 8080.
- **Rootless port permission denied:** confirm
  `net.ipv4.ip_unprivileged_port_start` is 443 or lower.
- **Signal integration failure:** confirm `signal-api.service` is healthy and
  that the calling container resolves `signal-api` on the `monitoring` network.

The generated units store container IDs beneath the user's runtime directory.
If diagnosing stale state, inspect the relevant `%t/UNIT.ctr-id` file (normally
under `/run/user/1000`) only after stopping the unit; let systemd and Podman
manage its normal lifecycle.

## Security and backups

Do not commit the Gatus `.env`, certificates, or persistent application data.
They can contain credentials, Signal identity keys and messages, monitoring
history, or private trust material. Restrict their ownership and permissions to
`ec2-user`, and include the Gatus data, Signal API state, and Uptime Kuma data in
the host's protected backup process. Also back up `~/gatus-caddy-data` as the
rootless service user; its `data` subdirectory contains private keys and ACME
account state. Do not remove it during routine container updates.
