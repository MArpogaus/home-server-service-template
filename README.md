# service-__NAME__

`__NAME__` service for the SecureBlue home server. Copy this repo, replace
`__NAME__`, then:

1. Add the service to `base_setup_extra_services` in
   `deployment-private/secrets/vars.yml`.
2. Add its `ansible-role` to `roles_path` in `ansible-base/ansible.cfg`.
3. Add its name to `SERVICES` when you run the deploy and test scripts.

## Structure

```
service-__NAME__/
├── ansible-role/__NAME___service/
│   ├── defaults/main.yml      Image tags, resource ceilings, auto_update
│   ├── vars/main.yml          File modes that differ from 0644
│   └── tasks/main.yml         Data directories, then import quadlet_service
├── quadlets/
│   ├── __NAME__.pod           Pod: published ports
│   ├── __NAME__-*.container.j2  Containers (templated)
│   └── configs/               Env and config files; .j2 is templated, the rest copied
└── containers/                (optional) custom image build
```

## Role Contract

Received from `site.yml`:

| Var | Example |
|-----|---------|
| `service_name` / `service_user` | `__NAME__` |
| `service_home` | `/var/services/__NAME__` |
| `service_repo` | `../service-__NAME__` |

The role creates its data directories, then imports `quadlet_service` from
`ansible-base`. That role deploys `quadlets/`, `quadlets/container.d/` and
`quadlets/configs/`, reloads the user manager and restarts the pod when a file
changed. A file that must not
be world-readable gets its mode in `vars/main.yml`:

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
- Every container gets a `--memory` ceiling, `--pids-limit` and
  `--security-opt=no-new-privileges`.
- Give a container `HealthCmd` + `HealthOnFailure=kill` when the image offers a
  check you have actually run. A wrong check plus `kill` is worse than none: it
  restarts a healthy container forever. Three cases legitimately have none in
  this project: php-fpm speaks FastCGI rather than HTTP and is covered end to
  end by the web container's `status.php` check, Alloy's image ships no HTTP
  client, and a cron loop has no meaningful liveness signal.
- Logs go to stdout. `quadlets/container.d/log.conf` sets
  `LogDriver=passthrough`, so lines reach the journal at the unit's priority;
  Alloy ships them to Loki. A program that opens `/dev/stdout` by path fails
  under passthrough (a journal stream is a socket): make it log via syslog to
  a mounted `/dev/log`, or keep journald for that pod.

## Development

Work on `dev`. Conventional commits.

```bash
pre-commit install --install-hooks -t pre-commit -t commit-msg -t pre-push
```

Plain `pre-commit install` wires up the pre-commit stage only, which leaves the
commit-message and branch hooks dormant.

## License

MIT
