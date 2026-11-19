# PatchPilot

**An open-source AI engineering agent that investigates, reproduces, and helps fix failing CI pipelines.**

PatchPilot is an early-stage project exploring a careful, evidence-driven workflow for CI failure investigation. The initial target is **GitHub Actions for Docker-based projects**.

> **Status:** Project scaffold / architecture in progress. The workflows described below are goals, not implemented capabilities yet.

## Why PatchPilot?

A failed CI run often leaves a developer with scattered logs, environment details, recent commits, and uncertain test results. PatchPilot aims to bring those signals together and help maintainers move from a failure to a well-supported, reviewable fix.

The guiding principle is:

**Don't just explain why a build failed. Reproduce it, gather evidence, and propose a verifiable fix.**

## Initial MVP scope

1. Receive GitHub Actions workflow failure events.
2. Collect relevant run metadata and logs.
3. Classify failures and identify likely flaky tests versus regressions.
4. Reproduce supported failures in a restricted, disposable environment.
5. Produce an evidence-backed diagnosis and proposed patch.
6. Run targeted regression checks.
7. Prepare a **draft** pull request for maintainer review.

PatchPilot will not merge changes or publish fixes automatically. Human approval remains required.

## Design principles

- **Evidence over confidence:** distinguish reproduced facts from hypotheses.
- **Least privilege:** GitHub credentials and write operations stay in a trusted control plane, not in untrusted build workers.
- **Untrusted code is hostile:** pull-request code and test output are treated as attacker-controlled.
- **Disposable execution:** workers can be terminated and recreated without losing workflow state.
- **Bounded compute:** enforce timeouts, retry budgets, and resource limits.
- **Honest status:** unsupported or inconclusive failures are reported as such.

## Planned architecture

The initial architecture is being evaluated around:

- **TypeScript monorepo:** pnpm workspaces and Turborepo.
- **API / control plane:** Node.js and Fastify.
- **Dashboard:** Next.js.
- **Durable orchestration:** Temporal.
- **Application metadata:** PostgreSQL with Prisma.
- **Artifacts:** S3-compatible object storage (MinIO for local development).
- **Execution isolation:** a separately operated Linux/KVM worker host using Firecracker microVMs, subject to feasibility and security review.

This is a proposed design, not a claim that these services are already integrated. Ordinary Docker containers alone are not considered a sufficient security boundary for arbitrary untrusted pull-request code.

## Repository layout (planned)

```text
.
├── apps/
│   ├── api/                 # Control-plane API
│   └── web/                 # Maintainer dashboard
├── packages/
│   ├── config/              # Shared configuration
│   ├── contracts/           # Typed API and workflow contracts
│   └── database/            # Prisma schema and database access
├── services/
│   ├── orchestrator/        # Temporal workflows and activities
│   └── worker/              # Isolated execution worker interface
├── docs/
│   ├── architecture.md
│   └── threat-model.md
└── README.md
```

The layout will evolve as implementation begins.

## Development roadmap

### Phase 1 — Reliable intake
- GitHub App / webhook design
- Signature verification and idempotent event intake
- Persist workflow metadata and raw event references

### Phase 2 — Investigation workflow
- Temporal workflow lifecycle
- Log and test-result normalization
- Failure classification with explicit uncertainty

### Phase 3 — Controlled reproduction
- Pinned source commit and environment fingerprints
- Disposable execution infrastructure
- Strict resource, filesystem, credential, and network boundaries

### Phase 4 — Verified suggestions
- Targeted reproduction and regression tests
- Evidence-backed patch proposal
- Draft pull request behind maintainer approval

## Security posture

PatchPilot is intended to process untrusted repository content. Pull-request code, build scripts, logs, artifacts, and issue text must never be treated as trusted instructions.

The design aims to:
- keep GitHub credentials outside execution workers;
- avoid privileged containers, Docker socket mounts, and host filesystem mounts;
- isolate untrusted execution on dedicated infrastructure;
- apply default-deny networking and resource quotas;
- store workflow state and artifacts outside disposable workers;
- make externally visible actions idempotent and approval-gated.

No sandbox is claimed to be escape-proof. The execution design requires review and adversarial testing before untrusted code execution is enabled.

See [the architecture notes](docs/architecture.md) and [the threat model](docs/threat-model.md).

## Contributing

This project is at an early stage. Issues and thoughtful design feedback are welcome. Please do not submit secrets, private customer logs, or untrusted artifacts without authorization.

## License

PatchPilot is licensed under the Apache License 2.0. See [LICENSE](LICENSE).
