# Podman systemd services

This repository contains systemd user units for running a small monitoring stack
with rootless Podman. The units are configured for the `ec2-user` account and use
absolute paths under `/home/ec2-user`.

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
- Access: `http://HOST:8085`, mapped to container port `8080`
- Data: `/home/ec2-user/gatus/config`, `/home/ec2-user/gatus/data`, and
  the configured CA bundle

### Uptime Kuma

- Unit: `uptime-kuma.service`
- Image: `docker.io/louislam/uptime-kuma:latest`
- Access: `http://HOST:3001`
- Data: `/home/ec2-user/uptime-kuma` mounted at `/app/data`

All three containers join the rootless Podman network named `monitoring`.
Gatus and Uptime Kuma declare ordering and soft dependencies on the Signal API:
systemd starts `signal-api.service` first, but a Signal API failure does not stop
the monitoring services from starting. The Signal API is not published on a host
port; other containers reach it by the `signal-api` network alias.

The units use `--rm` and `--replace`, so systemd recreates their containers when
they start. Persistent state remains in the host directories listed above.

## Prerequisites

- A Linux host running systemd with user services
- Podman installed at `/usr/bin/podman`
- The `ec2-user` account, normally with UID and GID 1000 as required by the
  Signal API unit
- Permission to expose TCP ports 8085 and 3001 on the host
- SELinux configuration that permits the labeled bind mounts, if SELinux is
  enabled

Run the Podman and systemd commands below as `ec2-user`. Do not install these
units as root or under `/etc/systemd/system`; `%t/containers` and
`default.target` are intended for the user's systemd manager.

## Prepare the host

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

Ensure all persistent files belong to `ec2-user`. The Signal API container maps
its application UID and GID to 1000, matching the values embedded in its unit.

## Install and start

From this repository, install the units into the user systemd directory:

```sh
install -d -m 0755 /home/ec2-user/.config/systemd/user
install -m 0644 \
  signal-api.service \
  container-gatus.service \
  uptime-kuma.service \
  /home/ec2-user/.config/systemd/user/

systemctl --user daemon-reload
```

Enable the Signal API first, followed by its consumers:

```sh
systemctl --user enable --now signal-api.service
systemctl --user enable --now container-gatus.service uptime-kuma.service
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
  signal-api.service container-gatus.service uptime-kuma.service
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
`latest` image tag. Review available updates before applying them:

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
not pinned. Back up persistent data and review upstream release notes before
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
- **Port conflict:** check whether another process already listens on TCP 8085
  or 3001.
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
the host's protected backup process.
