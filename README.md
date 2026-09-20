# service-__NAME__

`__NAME__` in a rootless Podman pod under the `__NAME__` user. The host is set
up by `ansible-base`, whose README is the entry point for the project.

Copy this repo, replace `__NAME__`, then:

1. Add the service to `base_setup_extra_services` in
   `deployment-private/secrets/vars.yml`.
2. Add its `ansible-role` to `roles_path` in `ansible-base/ansible.cfg`.
3. Replace `__PORT__` in `quadlets/__NAME__.pod` with a free host port (taken:
   8080 Nextcloud, 8081 ntfy, 3000/3100/9090 monitoring).
4. Run the `HealthCmd` in `__NAME__-main.container.j2` against your image, or
   delete the four `Health*` lines.
5. Add its name to `SERVICES` when you run `functional_test.sh` and `reset.sh`.

## Architecture

| Container | Image | Purpose |
|---|---|---|
| `__NAME__-main` | `example/__NAME__:1` | |

```
service-__NAME__/
├── ansible-role/__NAME___service/
│   ├── defaults/main.yml      Image tags, resource ceilings, auto_update
│   ├── vars/main.yml          File modes that differ from 0644
│   └── tasks/main.yml         Data directories, then import quadlet_service
├── quadlets/
│   ├── __NAME__.pod           Pod: published ports
│   ├── __NAME__-*.container.j2  Containers (templated)
│   ├── container.d/log.conf   LogDriver=passthrough for every container
│   └── configs/               Env and config files; .j2 is templated, the rest copied
└── containers/                (optional) custom image build
```

## Configuration

| Variable | Default | Controls |
|---|---|---|
| `__NAME___service_main_image` | `docker.io/example/__NAME__:1` | Pinned image tag |
| `__NAME___service_main_extra_args` | `--memory=256M ...` | Container ceilings |
| `__NAME___service_auto_update` | `registry` | Podman auto-update |

## Role contract

Received from `site.yml`:

| Var | Example |
|-----|---------|
| `service_name` / `service_user` | `__NAME__` |
| `service_home` | `/var/services/__NAME__` |
| `service_repo` | `../service-__NAME__` |

The role creates its data directories, then imports `quadlet_service` from
`ansible-base`. That role deploys `quadlets/`, `quadlets/container.d/` and
`quadlets/configs/`, reloads the user manager and restarts the pod when a file
changed. A file that must not be world-readable gets its mode in
`vars/main.yml`:

```yaml
quadlet_service_config_modes:
  __NAME__.env: '0600'
```

A pod file that is not `<service_name>.pod` is named with
`quadlet_service_pod`. A task that must restart the pod for a reason of its own
passes `quadlet_service_restart: true` to the import.

## Conventions

- Each rootless user has its own container network. Talk to other services
  through the host, on the published port, never by container name. Use an
  address, not `host.containers.internal`: nginx resolves an upstream name
  through its `resolver` directive, which never reads `/etc/hosts`.
- To reach a port that another pod published on the host loopback, put
  `Network=pasta:--map-host-loopback,<address>` on this pod and use that
  address. The default host address reaches routable addresses only.
- Inside a pod use `127.0.0.1:<port>`. A rootless pod binds IPv4 only, and
  `localhost` resolves to `::1` first.
- Pin image tags to a major/minor; `AutoUpdate=registry` follows the tag.
- Every container gets `--memory`, `--pids-limit` and
  `--security-opt=no-new-privileges` (see `defaults/main.yml`).
- Add `HealthCmd` + `HealthOnFailure=kill` only after you ran the check against
  the image. A wrong check plus `kill` restarts a healthy container forever.
- Logs go to stdout; `quadlets/container.d/log.conf` sets
  `LogDriver=passthrough`. A program that opens `/dev/stdout` by path fails
  under passthrough. Make it log via syslog to a mounted `/dev/log`, or keep
  journald for that pod.

## Development

Work on `dev`. Conventional commits. Hook setup: `ansible-base/README.md`.

## License

MIT
