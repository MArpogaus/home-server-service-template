# Contributing

Work on `dev`. `main` takes a merge from `dev` with `--no-ff`.

Write conventional commits. The commitizen hook rejects a message that does not
follow the format.

Install the hooks in this repository:

```bash
pre-commit install --install-hooks -t pre-commit -t commit-msg -t pre-push
```

Plain `pre-commit install` installs the pre-commit stage alone, and the commit
message and branch hooks then do not run. CI runs the pre-commit stage hooks on
a push and on a pull request.

Every GitHub action is pinned to a commit SHA. Dependabot updates the actions
and the hook revisions weekly against `dev`. `pinact run -u` updates and
re-pins the actions by hand.

## Releases

A release is a merge of `dev` into `main`, then an annotated tag on the merge
and `git push --follow-tags`. The `release` workflow turns every pushed tag into
a GitHub release. GitHub writes its notes: the pull requests merged since the
previous release and a link that compares the two tags.

## House style

A comment says why, never what. Longer reasoning belongs in the README of the
repository that owns the code. Write the prose in Simplified Technical English:
short sentences, one meaning per word, and the condition before the command.

## Rules a service follows

- Each rootless user has its own container network. Talk to other services
  through the host, on the published port, never by container name. Use a
  literal address, not a name. `home-server-bunker/README.md` has the reasoning
  and the traps.
- To reach a port that another pod published on the host loopback, put
  `Network=pasta:--map-host-loopback,<address>` on this pod. Use that address.
  The default host address reaches routable addresses only.
- Inside a pod use `127.0.0.1:<port>`. A rootless pod binds IPv4 only, and
  `localhost` resolves to `::1` first.
- Order the Quadlet sections `[Unit] [Container] [Service] [Install]`.
  `quadlet_service` in `home-server-core` gives every container the restart
  policy, `AutoUpdate=registry`, no capabilities, `no-new-privileges` and a
  pids limit as `container.d/` drop-ins. `quadlets/container.d/` of the service
  adds its own, such as the log driver. A container carries only what is its
  own. systemd applies a drop-in after the unit file, so you cannot override a
  key that a drop-in sets. Pick another key instead.
- A bind mount of a file this repository owns carries `z` (shared with the
  other containers of the pod) or `Z` (this container alone). Podman labels it
  for SELinux on each start, and nothing else does. A host path such as
  `/var/log/journal` or `/dev/log` carries neither: relabelling it breaks the
  service that owns it.
- Pin image tags to a major/minor. `AutoUpdate=registry` follows the tag.
- Every container gets a `Memory=` ceiling. If the entrypoint runs as root and
  switches user or fixes ownership, add `AddCapability=SETUID SETGID` (`CHOWN`,
  ...) to that container. Add a comment that names the step that needs it. Try
  without the capability first: an image with `USER` set needs nothing.
- Add `HealthCmd` + `HealthOnFailure=kill` only after you ran the check against
  the image. A wrong check plus `kill` restarts a healthy container forever.
- Logs go to stdout. `quadlets/container.d/log.conf` sets
  `LogDriver=passthrough`. A program that opens `/dev/stdout` by path fails
  under passthrough. Make that program log through syslog to a mounted
  `/dev/log`, or keep journald for that pod.

## Checks in this repository

- Hooks: the basics, ansible-lint and commitizen.
- Ansible variables are `<role>_*`.
- Renovate updates the container image tags in the role defaults, through the
  preset that `.github/renovate.json` extends.
