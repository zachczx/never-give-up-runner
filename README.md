# GitHub Runner on Coolify

A reusable Docker Compose setup for repository-scoped GitHub Actions runners on Coolify.

## Why

After server reboots, the runner sometimes became stuck before it started listening for jobs. The container still appeared healthy, so GitHub Actions jobs remained pending. The health check therefore verifies that GitHub considers the runner online, rather than only checking for a local process.

The runner is also intentionally long-lived, so CI should not repeatedly pay network and extraction costs that the host can safely retain between jobs.

This setup keeps job workspaces disposable while making runner infrastructure durable:

- checks GitHub to confirm the runner is online and ready for jobs;
- detects the stuck state where the container is running but GitHub cannot dispatch work to it;
- gives CI configurable lower CPU priority and a memory limit;
- persists the GitHub tool cache used by actions such as `setup-go` and `setup-node` across container recreation;
- keeps Go module/build caches and the npm cache under the persistent runner volume; and
- warms GitHub's official first-party action archive cache on runner startup.

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
