# home-server-__NAME__

A skeleton to copy for a new service: an Ansible role and a rootless Podman
Quadlet pod. `__NAME__` marks the service name and `__PORT__` the pod's
loopback port.

Copy it to `home-server-<name>`, then rename the paths and replace the
placeholders in the file contents, and delete the placeholder rule in
`.github/renovate.json`. The name is the Linux user, the pod, the
`service` label, the role `<name>_service` and the variable prefix
`<name>_service_`. `home-server/README.md`, "Adding a service", has the steps
outside this repository.

| Container | Job | Memory ceiling |
|---|---|---|
| `__NAME__-main` | | 256M |

```
home-server-__NAME__/
├── ansible-role/__NAME___service/
│   ├── defaults/main.yml        Image tags
│   ├── vars/main.yml            Modes of config files that are not 0644
│   └── tasks/main.yml           Data directories, then import quadlet_service
├── quadlets/
│   ├── __NAME__.pod             Pod and published ports
│   ├── __NAME__-*.container.j2  Containers
│   ├── container.d/             Drop-ins for every container of this service
│   └── configs/                 Env and config files
├── monitoring/                  Rules, dashboards and log filters
└── containers/                  (optional) build of an own image
```

## Configuration

| Variable | Default | Controls |
|---|---|---|
| `__NAME___service_main_image` | `docker.io/example/__NAME__:1` | Image and tag |

## Role contract

`site.yml` includes `<service_repo>/ansible-role/<name>_service` once for each
entry of `base_setup_services`, and passes:

| Var | Value |
|---|---|
| `service_name` | `__NAME__` |
| `service_home` | `/var/services/__NAME__` |
| `service_repo` | `<playbook dir>/services/__NAME__` |

Before that, `base_setup` creates the user with its `uid` and subuid range, the
home as a Btrfs subvolume with mode `0750`, a snapshot timer for it, linger and
the user's `podman-auto-update.timer`.

`base_setup` admits in `/etc/containers/policy.json` only the repositories of
the `*_image` variables in `defaults/main.yml` and of the
`<name>_service_*_image` host variables. A reference names
`registry/namespace/name`, such as `docker.io/library/nginx`; a shorter one
fails the deploy. An image under `ghcr.io/marpogaus` needs this project's
cosign signature.

The role creates its data directories, then imports `quadlet_service` from
`home-server`. `quadlet_service`:

- packs `quadlets/`, its own `container.d/` drop-ins and the extra files into
  one reproducible archive. It renders each `.j2` file without the suffix and
  copies the other files. The drop-ins set the restart policy,
  `AutoUpdate=registry`, `DropCapability=ALL`, `NoNewPrivileges=true` and
  `PidsLimit=512`.
- compares the archive with the one it last unpacked on the host. When they
  differ, it deletes `~/.config/containers/systemd/` of the service user,
  unpacks the archive there, reloads the user manager and restarts
  `<name>-pod.service`, from `quadlets/<name>.pod`. Otherwise it only starts the
  pod if it is stopped.

Nothing else writes into the Quadlet directory: the next change deletes it.

| Var | Default | Use |
|---|---|---|
| `quadlet_service_config_modes` | `{}` | Mode per file, relative to `configs/`, in `vars/main.yml` |
| `quadlet_service_restart` | `false` | `true` restarts the pod for a reason of the role |
| `quadlet_service_extra_files` | `[]` | More files: `dest` plus `src` or `content` |

```yaml
quadlet_service_config_modes:
  __NAME__.env: '0600'
```

## Monitoring

All files are optional. `home-server-monitoring/README.md`, "Monitoring files of
a repository", says how they reach the host.

| File | Holds |
|---|---|
| `monitoring/prometheus-rules.yaml` | Prometheus rule groups |
| `monitoring/loki-rules.yaml` | Loki ruler groups |
| `monitoring/dashboards/*.json` | Grafana dashboards |
| `monitoring/alloy-drop.txt` | One RE2 regex per line; Alloy drops a line of this service that matches |
| `monitoring/alloy-redact.txt` | One RE2 regex per line; its capture groups become `<redacted>` in every line |

- The files are not templates. In the Alloy files, `#` lines and blank lines
  do not count.
- A service's own alert name starts with its service name. The generic alerts
  of `home-server` and `home-server-monitoring` do not. `labels.severity` is
  `critical`, `warning` or `info`. `annotations.summary` is one line.
- A Loki rule selects only on the labels in `home-server-monitoring/README.md`,
  "Labels".
- A dashboard carries the tag `home-server` and no `links`, and uses the
  datasource uids `prometheus` and `loki`. The role adds the link bar.

A new service gets the generic alerts without a rule of its own:
`ContainerRestartLoop`, `UserUnitFailed`, the snapshot `JobStale` and the host
memory alerts.

## LLM coding tools

This project is developed with LLM-based coding tools. They write most of the
code and documentation. The maintainer sets the goals and the design, reviews
every change and is responsible for it. Changes are tested on a VM before they
reach a host.

## License

MIT
