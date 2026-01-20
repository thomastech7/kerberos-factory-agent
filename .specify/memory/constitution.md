<!--
Sync Impact Report
- Version change: none → 1.0.0
- Modified principles: N/A (initial adoption)
- Added sections: Core Principles; Security Requirements; Development Workflow; Governance
- Removed sections: None
- Templates requiring updates:
	- ✅ Updated: .specify/templates/plan-template.md (Constitution Check gates)
	- ✅ Updated: .specify/templates/tasks-template.md (test expectations)
	- ⚠ Pending/Not Applicable: .specify/templates/commands/*.md (folder not present)
- Follow-up TODOs: None
-->

# Kerberos Factory Agent Constitution

## Core Principles

### I. Security & Privacy (NON-NEGOTIABLE)
Kerberos Factory Agent MUST protect user data and infrastructure. Requirements:
- MUST ship secure defaults: no default credentials in production; warn when defaults detected.
- MUST support encryption at rest and in-flight where configured; secrets MUST never be logged and
	MUST be provided via environment variables or secret stores.
- MUST validate and sanitize all external inputs (RTSP/MQTT/HTTP/WebRTC) to prevent injection and
	resource abuse; avoid shell execution unless strictly necessary and reviewed.
- MUST redact sensitive values in logs and responses; PII and keys MUST NOT appear in telemetry.

Rationale: Video surveillance contains sensitive content; compromise harms users and organizations.

### II. Reliability & Data Integrity
Recording and signaling MUST be dependable under normal and degraded conditions.
- MUST never drop recordings silently. On drop, emit an explicit error event and log at `error`.
- MUST implement bounded storage with automatic cleanup; defaults MUST be safe (prevent disk full).
- MUST ensure crash-safe writes for finalized segments; fsync or equivalent on segment finalize.
- SHOULD apply backpressure or graceful degradation under overload rather than OOM or panics.

Rationale: The agent’s primary value is trustworthy capture and availability of recordings.

### III. Observability & Operability
The system MUST be diagnosable and operable in containers and at the edge.
- MUST emit structured logs (JSON) to stdout with level, timestamp, component, and correlation ID.
- MUST expose health/readiness endpoints and basic metrics (e.g., frame rate, queue depth, errors).
- SHOULD provide text/JSON interfaces and CLI hooks for local debugging where feasible.

Rationale: Clear signals enable fast triage in constrained, distributed environments.

### IV. Backward Compatibility & Upstream Alignment
This fork MUST remain compatible with Kerberos Agent where possible.
- MUST keep existing configuration keys and REST APIs compatible; additive changes by default.
- Breaking changes REQUIRE a deprecation window, migration guide, and a MAJOR version bump.
- Risky features MUST be behind opt-in flags disabled by default until stabilized.
- Maintain a documented parity/backlog with upstream and a periodic sync process.

Rationale: Users depend on Kerberos Agent conventions; compatibility lowers migration cost.

### V. Test-First & Reproducible Builds
Quality is enforced via tests and deterministic builds.
- New features and bug fixes impacting capture/recording, protocols, or storage MUST include
	tests (unit and/or integration) authored before or alongside implementation.
- Provide mocks/fixtures for RTSP, MQTT, and minimal WebRTC flows where practical.
- CI MUST build deterministically (locked dependencies) and run tests; builds MUST be reproducible
	via Docker/devcontainer.

Rationale: The domain is stateful and concurrent; regressions are costly without tests.

## Security Requirements

- Secrets Management: All secrets via environment variables or secret stores; never committed.
- Key Handling: If encryption/signing used, keys MUST be rotatable and not embedded in images.
- Vulnerability Hygiene: Regularly run dependency and container scans; address high/critical issues
	before release.
- SBOM/Provenance: Prefer generating an SBOM for release artifacts; document provenance steps.
- Default Hardening: Disable unused services; minimal base images; least-privilege runtime.

## Development Workflow

- Specifications: Each feature SHOULD include `spec.md` and `plan.md` using the provided templates.
- Constitution Check: PRs MUST demonstrate compliance with principles II–V via the plan’s gates.
- Reviews: At least one maintainer review REQUIRED for changes touching security, storage, or
	protocols.
- Versioning: Semantic Versioning ($MAJOR.MINOR.PATCH$) applies project-wide.
	- MAJOR: Breaking behavior or config.
	- MINOR: Backward-compatible features or materially expanded guidance.
	- PATCH: Fixes, refactors, or non-semantic documentation changes.
- Changelog: Document user-facing changes and breaking migrations with clear upgrade steps.

## Governance

- The Constitution supersedes other practice documents for areas it covers.
- Amendments: Propose via PR updating this file; include a Sync Impact Report and version bump
	rationale; obtain maintainer approval.
- Compliance: Reviewers MUST block PRs that violate non-negotiable requirements unless an explicit
	temporary exception is documented with a remediation plan and due date.
- Review Cadence: Reassess principles quarterly or after substantial upstream changes.

**Version**: 1.0.0 | **Ratified**: 2026-01-20 | **Last Amended**: 2026-01-20
