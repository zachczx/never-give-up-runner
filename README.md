# GitHub Runner on Coolify

A reusable Docker Compose setup for repository-scoped GitHub Actions runners on Coolify.

## Why

After server reboots, the runner sometimes became stuck before it started listening for jobs. The container still appeared healthy, so GitHub Actions jobs remained pending. The health check therefore verifies that GitHub considers the runner online, rather than only checking for a local process.

The runner is also intentionally long-lived, so CI should not repeatedly pay network and extraction costs that the host can safely retain between jobs.

This setup keeps job workspaces disposable while making runner infrastructure durable:

- checks GitHub to confirm the runner is online and ready for jobs;
- detects the stuck state where the container is running but GitHub cannot dispatch work to it;
- checks that the runner can still reach the host Docker daemon, which jobs with service containers need;
- keeps Docker access working after the host Docker daemon restarts;
- gives CI configurable lower CPU priority, an optional CPU cap and a memory limit;
- persists the GitHub tool cache used by actions such as `setup-go` and `setup-node` across container recreation;
- keeps Go module/build caches and the npm cache under the persistent runner volume; and
- warms GitHub's official first-party action archive cache on runner startup.

## Docker access

Jobs that start service containers, such as Postgres for integration tests, use the host Docker daemon. The runner used to bind-mount the single file `/var/run/docker.sock`. When the host Docker daemon restarts, for example during a Coolify update, it creates a new socket file, and a file bind mount keeps pointing at the old one. This is the most likely cause of an outage in October 2026: the runner stayed online for GitHub and healthy for Coolify, but every Docker call failed with `Cannot connect to the Docker daemon`.

The runner now mounts the host's `/var/run` directory read-only at `/host-run` and links `/var/run/docker.sock` to `/host-run/docker.sock` at startup, so it always reaches the current socket. The read-only flag stops the container from changing files in the host's `/var/run`. It does not restrict Docker itself: as before, the runner has full control of the host Docker daemon. The health check also runs `docker version` and reports the container unhealthy when Docker is unreachable.

The startup link needs the image's default `RUN_AS_ROOT=true`, because a non-root runner cannot write `/var/run`. Neither Docker Compose nor Coolify restarts a container just because it is unhealthy. The directory mount is what recovers from Docker restarts; the health check only makes any other Docker outage visible.

## Resource limits

Set these through the resource's environment variables. Coolify's Resource Limits form does not apply to a Docker Compose deployment: Coolify writes those values only into the compose file it generates for Dockerfile, Nixpacks, image and static deployments, and a Compose deployment uses this file as written.

| Variable | Default | Effect |
| --- | --- | --- |
| `RUNNER_CPU_SHARES` | `256` | CPU weight against Docker's default of 1024. CI yields to other services only while the host is busy, and gets the whole host when it is idle. |
| `RUNNER_CPUS` | `0` (no cap) | Hard CPU cap. It throttles CI even on an idle host, so prefer the weight above for protecting production. |
| `RUNNER_MEMORY_LIMIT` | `4g` | Memory limit for the runner and its jobs. A Playwright job running a Vite dev server, Chromium and the test runner uses about 1.7 GB, and the runner process and page cache count against the same limit. The old 2 GB default left no headroom. |

Keep the memory limits of all runners on a host, plus the host's other services, within its RAM. A limit is a cap, not a reservation, so an overcommitted host can still run out of memory.

After a redeploy, confirm the values Docker applied:

```sh
docker inspect <runner-container> --format 'cpus={{.HostConfig.NanoCpus}} quota={{.HostConfig.CpuQuota}} shares={{.HostConfig.CpuShares}} mem={{.HostConfig.Memory}}'
```

The defaults show `cpus=0 quota=0 shares=256 mem=4294967296`. Different values mean the environment variables above are set, or the deployment is not using this file.

## Persistent caches

`runner-data` stores the work directory and local dependency/build caches under `/runner`. `runner-toolcache` stores `/opt/hostedtoolcache`, which is where the upstream `myoung34/github-runner` image exposes the GitHub Actions tool cache.

The action archive warm-up uses the latest `actions/action-versions` release. It checks the release on startup and only downloads the archive when the cached release changes. The release SHA-256 digest is verified when GitHub publishes one.

Cache warm-up is best-effort. If GitHub is unavailable or the archive cannot be verified or extracted, the runner still starts. Missing action archives fall back to normal GitHub Actions downloads.

Set `WARM_ACTION_CACHE=false` to disable the startup warm-up.

GitHub's archive contains popular first-party actions, not arbitrary third-party actions. Third-party actions will still be downloaded normally.

## Workflow guidance

A persistent runner does not need to download and unpack remote dependency caches on every job when the same local disk already owns those caches. Keep setup actions for version selection, but disable their remote dependency-cache feature where appropriate.

For Go:

```yaml
- uses: actions/setup-go@v6
  with:
    go-version-file: api/go.mod
    cache: false
```

Continue running dependency installation in each job. A clean workspace is useful CI hygiene; the persistent package/module stores make those installs cheap without carrying `node_modules` or generated build output between jobs.

For package managers other than npm, configure their content-addressed store or download cache to a path under `/runner` if you want it to survive container recreation.
