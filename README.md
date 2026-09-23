# home-server-__NAME__

A skeleton to copy for a new service. It holds an Ansible role and a Podman
Quadlet pod, with `__NAME__` where the name goes and `__PORT__` where the
pod's loopback port goes. A service that builds its own
image adds a `containers/` directory.

Copy it to `home-server-<repo>`. Then rename the paths and the file contents.
`home-server-core/README.md`, "Adding a service", has the steps that follow.

## Architecture

| Container | Image | Purpose |
|---|---|---|
| `__NAME__-main` | `docker.io/example/__NAME__:1` | |

```
home-server-__NAME__/
├── ansible-role/__NAME___service/
│   ├── defaults/main.yml      Image tags
│   ├── vars/main.yml          File modes that differ from 0644
│   └── tasks/main.yml         Data directories, then import quadlet_service
├── quadlets/
│   ├── __NAME__.pod           Pod: published ports
│   ├── __NAME__-*.container.j2  Containers (templated)
│   ├── container.d/           Drop-ins for every container of this service
│   │                          (log driver); core adds restart and hardening
│   └── configs/               Env and config files; .j2 is templated, the rest copied
├── monitoring/                Alert rules and dashboards for home-server-monitoring
└── containers/                (optional) custom image build
```

## Configuration

| Variable | Default | Controls |
|---|---|---|
| `__NAME___service_main_image` | `docker.io/example/__NAME__:1` | Pinned image tag |

## Role contract

Received from `site.yml`:

| Var | Example |
|-----|---------|
| `service_name` / `service_user` | `__NAME__` |
| `service_home` | `/var/services/__NAME__` |
| `service_repo` | `<playbook dir>/../home-server-__NAME__` |

The role creates its data directories, then imports `quadlet_service` from
`home-server-core`. That role deploys `quadlets/`, `quadlets/container.d/` and
`quadlets/configs/`, adds the shared restart and hardening drop-ins, reloads
the user manager and restarts the pod when a file changed. A file that must not
be world-readable gets its mode in `vars/main.yml`:

```yaml
quadlet_service_config_modes:
  __NAME__.env: '0600'
```

`quadlet_service_pod` names a pod file that is not `<service_name>.pod`. A task
that must restart the pod for a reason of its own passes
`quadlet_service_restart: true` to the import.

## Monitoring

`home-server-monitoring` collects `monitoring/loki-rules.yaml`,
`monitoring/prometheus-rules.yaml` and `monitoring/dashboards/*.json` from
every service repository. The files are plain, not templated. A rule selects
only on the labels of the contract in `home-server-monitoring/README.md`, and
its alert name starts with the service name. A new service gets the generic
container, snapshot and memory alerts without a rule of its own.

## License

MIT
