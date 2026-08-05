# GitHub Runner on Coolify

A reusable Docker Compose setup for repository-scoped GitHub Actions runners on Coolify.

## Problem

After server reboots, the runner sometimes became stuck before it started listening for jobs. The container still appeared healthy, so I only noticed when GitHub Actions jobs remained pending.

The default health check only verified that a runner process existed. It did not verify that GitHub considered the runner online and ready to receive work.

## What this adds to default Coolify compose file

* Checks GitHub to confirm runner is ready for jobs
* Detects stuck state where the container is running but jobs remain pending;
* Gives CI a configurable lower CPU priority during contention
* Adds configurable memory limit