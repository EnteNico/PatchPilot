# Threat model (initial)

**Status:** Initial design notes. This document is not a security audit or certification.

## Assets to protect

- GitHub App credentials and installation permissions
- Control-plane and database credentials
- Other tenants' source code and build artifacts
- Worker host and orchestration infrastructure
- Integrity of diagnoses, patches, and benchmark results

## Adversaries and untrusted inputs

Assume a pull request may be intentionally malicious. Treat repository code, workflow YAML, build scripts, dependency lifecycle hooks, test output, logs, artifacts, issue text, and model-generated content as untrusted.

Potential attacks include:
- prompt injection embedded in source files, logs, comments, or test output;
- attempts to read environment variables or exfiltrate credentials;
- container or VM escape attempts;
- network scanning and access to cloud metadata endpoints;
- resource exhaustion and fork bombs;
- malicious dependencies and build-time scripts;
- forged or replayed webhook deliveries;
- artifact path traversal or cross-job data access.

## Security requirements

1. Verify GitHub webhook signatures and deduplicate delivery IDs.
2. Keep GitHub write credentials outside the execution plane.
3. Use least-privilege GitHub App permissions and repository allowlists.
4. Isolate untrusted execution on dedicated disposable microVM infrastructure.
5. Apply default-deny networking, resource quotas, deadlines, and bounded output sizes.
6. Do not mount host paths or the Docker socket into untrusted jobs.
7. Validate all control-plane requests and artifact paths.
8. Treat model output as a proposal; enforce policy in deterministic code.
9. Require human approval before creating or publishing externally visible changes.
10. Record auditable events without storing secrets.

## Fail-safe behavior

If isolation, signature verification, policy validation, or environment pinning cannot be established, do not execute the job. Terminate suspicious workers, quarantine outputs, and record the reason.

A failed reproduction must be reported as inconclusive; it must not be represented as a verified fix.

## Open questions

- Which Firecracker host configuration and network model will be supported first?
- How will artifact scanning and retention be implemented?
- Which GitHub App permissions are strictly necessary for the MVP?
- What adversarial test suite will be required before enabling untrusted PR execution?
