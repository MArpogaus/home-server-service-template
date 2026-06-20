# service-__NAME__

`__NAME__` service for SecureBlue deployment.

## Structure

```
service-__NAME__/
├── ansible-role/
│   └── __NAME___service/     Ansible role for deploying this service
│       ├── defaults/main.yml  Role default variables
│       ├── tasks/main.yml     All deployment logic (the contract)
│       ├── templates/         Jinja2 templates (env files, etc.)
│       └── files/             Static files (scripts, etc.)
├── quadlets/                  Podman Quadlet files
│   ├── __NAME__.pod           Pod definition
│   ├── __NAME__-*.container   Container definitions
│   ├── *.volume               Named volumes
│   ├── shared-network.network Shared bridge network
│   ├── promtail-__NAME__.*    Log shipping
│   └── configs/               Config files
├── containers/                (optional) Custom container image build
│   ├── Containerfile
│   └── context/
└── .github/workflows/         CI/CD
```

## Role Contract

The Ansible role receives these variables from `site.yml`:

| Var | Description | Example |
|-----|-------------|---------|
| `service_name` | Service name | `__NAME__` |
| `service_user` | System user | `__NAME__` |
| `service_uid` | User UID | `1003` |
| `service_home` | Home directory | `/var/services/__NAME__` |
| `service_repo` | Repo path | `../service-__NAME__` |

The role MUST:
1. Create required data directories under `{{ service_home }}`
2. Copy static Quadlet files to `{{ service_home }}/.config/containers/systemd/`
3. Template any `.container.j2` files to the same directory (strip `.j2`)
4. Copy config files to `{{ service_home }}/.config/containers/systemd/configs/`
5. Template env files to `configs/` (mode `0600`)
6. Deploy and enable/start any systemd timer units
