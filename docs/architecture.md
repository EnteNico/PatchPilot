# Architecture notes

**Status:** Proposed design; implementation and feasibility validation are pending.

## Goals

- Reliably ingest GitHub Actions failures.
- Investigate failures with reproducible evidence.
- Separate trusted orchestration from untrusted code execution.
- Survive worker loss without losing workflow state.
- Keep compute bounded and externally visible actions approval-gated.

## Trust boundaries

### Control plane (trusted)

The API, Temporal workers, database, artifact store, and GitHub integration service form the control plane. GitHub App credentials are held only by the GitHub integration service or a secrets manager used by it.

The control plane validates webhook signatures, deduplicates deliveries, schedules work, records state, and decides which actions are permitted.

### Execution plane (untrusted)

Build scripts, repository code, test runners, dependency install hooks, logs, and generated artifacts are attacker-controlled. They must not receive GitHub write tokens, database credentials, orchestration credentials, or host filesystem access.

For the first release, untrusted execution remains disabled until the isolation host and threat model have been reviewed and tested.

## Proposed workflow

1. Receive a GitHub Actions webhook and verify its signature.
2. Persist the delivery identifier and run metadata; reject duplicate deliveries.
3. Create a durable Temporal workflow keyed by repository, run, and attempt.
4. Fetch only the metadata and logs required for diagnosis.
5. Create a pinned execution specification (commit SHA, image digest, lockfiles, workflow metadata).
6. Request a disposable worker through a narrow worker-provisioning interface.
7. Stream bounded logs and artifacts to external storage.
8. Terminate the worker and record the result.
9. Generate a diagnosis and proposed patch from collected evidence.
10. Run targeted verification in a fresh execution environment.
11. Prepare a draft pull request only after policy checks; require maintainer approval for publication.

## State ownership and worker loss

Workers are disposable and never own authoritative workflow state. Temporal owns workflow progress and retry scheduling; PostgreSQL stores application metadata; object storage holds logs, reports, patches, and other durable artifacts.

A worker reports activity heartbeats and writes outputs to external storage. If it disappears, the activity times out or fails. A retry provisions a new worker from the same immutable execution specification and resumes from the last completed durable stage.

Activities may be retried, so all side effects must be idempotent. Use stable idempotency keys for webhook deliveries, workflow stages, artifact writes, and GitHub operations.

Do not attempt to restore a dead VM's memory. Reconstruct from persisted inputs and completed stage outputs.

## Environment reproducibility

Record at minimum:
- source commit SHA;
- OCI image digest (not a mutable tag);
- workflow file revision;
- dependency lockfile hashes;
- runtime and toolchain versions;
- relevant non-secret environment metadata.

Secrets are not part of the environment fingerprint and must never be copied into artifacts or worker logs.

## Compute controls

- Per-stage wall-clock deadlines.
- CPU, memory, disk, and process limits.
- Bounded retries with backoff.
- Targeted test selection before broader suites.
- Cache immutable OCI layers and dependency artifacts where safe.
- Stop and report inconclusive when reproduction is not reliable.

Flaky-test classification is probabilistic; it must not automatically trigger code changes.

## Execution isolation direction

Firecracker microVMs on dedicated Linux/KVM hosts are the current design direction for hostile workloads, with a narrowly scoped provisioning service, jailer, seccomp, cgroups, and network policy. This is not implemented or security-certified.

A worker must not have:
- the host Docker socket;
- privileged host mounts;
- GitHub credentials or control-plane secrets;
- unrestricted network egress;
- access to other jobs' disks or artifacts.

If the required isolation is unavailable, untrusted jobs must be rejected rather than silently run in ordinary containers.

## GitHub side effects

The GitHub integration service uses a narrowly scoped GitHub App installation. It validates repository identity and permissions, applies idempotency keys, and prepares draft pull requests only. Merge, direct pushes, and automatic issue/PR publication are out of scope for the MVP.
