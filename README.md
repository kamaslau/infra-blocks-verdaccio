# infra-blocks-verdaccio

[![Blocks](https://img.shields.io/badge/infra--blocks-index-555?style=flat-square)](https://github.com/kamaslau/infra-blocks)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
![Repository size](https://img.shields.io/github/repo-size/kamaslau/infra-blocks-verdaccio?color=56BEB8)

A unit, not a demo: one service, one port, one documented credential, disposable with a single command.

Typical use: private packages cannot go to the public registry, and a build has to work on a host that is offline next week.

[Verdaccio](https://verdaccio.org/docs/docker) as a private npm registry, with an uplink to npmjs so a cached public package installs from here too.

## Service URL

- Registry and web UI `http://localhost:4873`

Ports can be changed in `.env`; see Usage.

## Default user

None, and registration is open — which is how Verdaccio ships and the first thing to change on a host that is reachable from anywhere but your own machine.

| Setting | Default |
| ------- | ------- |
| Accounts | created by `npm adduser`, stored in `htpasswd` inside the storage volume |
| Access | `$all` — anonymous users can read |
| Publish | `$authenticated` — any account that exists, including one just created |
| Uplink | `npmjs` → `https://registry.npmjs.org/` |

`conf/config.yaml` also declares `taobao`, `tencentCloud` and `huawei` uplinks. They are unused until a package block names them in `proxy:`; they are there so the switch is one word, not a URL to look up.

## Usage

### Start with [Docker Compose](https://docs.docker.com/compose/)

```bash
# Initiate .env file
cp .env.example .env
# Start services
docker compose up -d
```

Update existing composed containers with latest images:

```bash
docker compose pull && \
docker compose down && \
docker compose up -d
```

### Point a client at it

```bash
# npm
npm config set registry http://localhost:4873/

# pnpm
pnpm config set registry http://localhost:4873/

# Check
npm config get registry
```

`nrm use` does the same thing if you keep several registries:

```bash
pnpm add -g nrm
nrm add verdaccio http://localhost:4873/
nrm use verdaccio
```

### Publish something

```bash
npm adduser --registry http://localhost:4873/
npm publish --registry http://localhost:4873/
```

The first account created becomes an ordinary user; publish rights come from `publish: $authenticated` in the config, not from being first.

### Stop, and what that keeps

```bash
docker compose down      # keeps packages, users and plugins
docker compose down -v   # removes the registry
```

## Verify

```bash
curl -s http://localhost:4873/-/ping
```

`{}` is the whole answer — the registry is up and answering, before any account exists. `docker compose ps` should say `(healthy)` for the same container.

A round trip, which is the check that matters:

```bash
npm config set registry http://localhost:4873/
npm adduser --registry http://localhost:4873/
npm publish --registry http://localhost:4873/
npm view http-response-text version --registry http://localhost:4873/
```

The last command proves the uplink works: the package is not local, so Verdaccio fetched it from npmjs and cached it. `docker compose logs verdaccio` shows the `GET` that did it.

## Why this shape

Upstream Verdaccio's own advice is to bind-mount `conf`, `storage` and `plugins` and then `chown -R 10001:65533` the host directories. This unit bind-mounts only `conf`, read-only, and puts the other two in named volumes: Docker copies the image's directory ownership into a fresh named volume, so the container's user can write without a `chown` on the host, and `down -v` removes them cleanly. The `conf` directory stays in git because it is configuration, not state.

The config is the packaged `docker.yaml`, with two edits: `VERDACCIO_PUBLIC_URL` is commented out, because a tarball URL baked into a repository is a URL that outlives the hostname it was written for; and the storage path is left at `/verdaccio/storage/data`, one level under the volume mount, so the `htpasswd` file and the package tarballs sit in the same tree that gets backed up.

Registration is open and `max_users` is unset. That is the upstream default and it is wrong for a host with a public interface: `max_users: -1` freezes registration, and accounts are then created by editing `htpasswd` or by toggling the value for the duration of an onboarding.

`publish: $authenticated` on both `@*/*` and `**` means any account that exists can publish any package name. Removing the `proxy:` line from a private package block is what stops dependency confusion — Verdaccio's own hardening note, and the reason the private scope is usually split into its own block first.

## Notes for deployments in China

The `verdaccio/verdaccio` image pulls from Docker Hub and needs a registry mirror. The uplink is the more interesting half: from a mainland host, `registry.npmjs.org` is slow or unreachable, and `taobao` in `conf/config.yaml` points at `registry.npmmirror.com`, which is the same content on a domestic edge. Set `proxy: taobao` in the package blocks and public installs come from the mirror while private packages stay local.

---

Part of [infrastructure blocks](https://github.com/kamaslau/infra-blocks) · ← back to index
