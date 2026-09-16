# service-__NAME__

`__NAME__` service for the SecureBlue home server. Copy this repo, replace
`__NAME__`, add the service to `base_setup_services` in
`ansible-base/roles/base_setup/defaults/main.yml`.

## Structure

```
service-__NAME__/
├── ansible-role/__NAME___service/
│   ├── defaults/main.yml      Image tags, resource ceilings, auto_update
│   ├── handlers/main.yml      daemon-reload + pod restart (only on change)
│   ├── tasks/main.yml         Deploy logic (the contract)
│   └── templates/             Env files
├── quadlets/
│   ├── __NAME__.pod           Pod: published ports
│   └── __NAME__-*.container.j2  Containers (templated)
└── containers/                (optional) custom image build
```

## Role Contract

Received from `site.yml`:

| Var | Example |
|-----|---------|
| `service_name` / `service_user` | `__NAME__` |
| `service_home` | `/var/services/__NAME__` |
| `service_repo` | `../service-__NAME__` |

The role MUST:
1. Create data directories under `{{ service_home }}`
2. Copy the `.pod` file and template every `*.container.j2` into `{{ service_home }}/.config/containers/systemd/`
3. Template env/config files into `.../systemd/configs/` (secrets with mode `0600`)
4. `notify` the `__NAME__ quadlets changed` handler from every file task, `flush_handlers`, then `start` the pod

## Conventions

- Each rootless user has its own container network. Talk to other services via
  the host address on the published port, never by container name. Prefer an
  address over `host.containers.internal`: some resolvers ignore `/etc/hosts`.
- Inside a pod use `localhost:<port>`.
- Pin image tags to a major/minor; `AutoUpdate=registry` follows the tag.
- Every long-running container gets `HealthCmd` + `HealthOnFailure=kill` and a `--memory` ceiling.
- Logs go to stdout; journald has them, Alloy ships them to Loki.

## Development

```bash
pre-commit install --install-hooks -t pre-commit -t commit-msg -t pre-push
```

Plain `pre-commit install` wires up only the pre-commit stage, so the
commitizen message and branch checks stay dormant. Hooks: shellcheck,
ansible-lint (which owns YAML style here), commitizen for conventional commits.
CI runs the same set on push and pull request. Actions are pinned to SHAs, and
dependabot updates actions and hook revisions weekly against `dev`.

## License

MIT
