# Feature Specification: ONVIF Operator Wrapper

**Feature Branch**: `001-onvif-operator`  
**Created**: 2026-01-20  
**Status**: Draft  
**Input**: User description: "Wrapper process (Operator pattern) to auto-discover ONVIF devices and spawn a Kerberos Agent process or container per device"

## User Scenarios & Testing *(mandatory)*

<!--
  IMPORTANT: User stories should be PRIORITIZED as user journeys ordered by importance.
  Each user story/journey must be INDEPENDENTLY TESTABLE - meaning if you implement just ONE of them,
  you should still have a viable MVP (Minimum Viable Product) that delivers value.
  
  Assign priorities (P1, P2, P3, etc.) to each story, where P1 is the most critical.
  Think of each story as a standalone slice of functionality that can be:
  - Developed independently
  - Tested independently
  - Deployed independently
  - Demonstrated to users independently
-->

### User Story 1 - Auto-discover and spawn agents (Priority: P1)

The Wrapper runs on a host and continuously discovers ONVIF-compatible devices on the configured network(s). For each discovered device that passes connectivity and credentials validation, the Wrapper spawns a dedicated Kerberos Agent instance (process or container) with a minimal default configuration and marks it healthy.

**Why this priority**: Auto-provisioning delivers immediate value by turning discovered cameras into managed agents without manual setup.

**Independent Test**: Start the Wrapper on a test network with one ONVIF device and pre-configured credentials; verify one Agent instance is spawned and reachable, and discovery logs/metrics reflect a successful registration.

**Acceptance Scenarios**:

1. **Given** the Wrapper is running with discovery enabled and valid credentials provided, **When** a new ONVIF device is detected and responds to capability queries, **Then** the Wrapper spawns exactly one Agent instance for that device and reports it as healthy.
2. **Given** the Wrapper is running and a device is detected but credentials are missing or invalid, **When** the device fails authentication, **Then** the device remains in a "pending credentials" state and no Agent instance is spawned.

---

### User Story 2 - Reconciliation loop (Priority: P2)

The Wrapper maintains desired state: each eligible device has exactly one healthy Agent instance. If a device goes offline, the corresponding Agent instance is gracefully paused or stopped; if it returns, the instance is resumed or restarted. If an Agent instance crashes or becomes unhealthy, the Wrapper restarts it and records the event.

**Why this priority**: Ensures reliability and reduces manual intervention, aligning with operator-style management.

**Independent Test**: Simulate device availability changes and Agent failures; verify the Wrapper reconciles to the desired state without producing duplicate or orphaned instances.

**Acceptance Scenarios**:

1. **Given** an Agent instance is running for a device, **When** the device becomes unreachable, **Then** the Wrapper transitions the Agent to a stopped/paused state and prevents new recordings until the device is reachable again.
2. **Given** an Agent instance crashes, **When** the Wrapper detects the unhealthy state, **Then** it restarts the instance and logs the event with correlation to the device.

---

### User Story 3 - Per-device configuration mapping (Priority: P3)

Administrators can define default Agent configuration and per-device overrides (e.g., stream selection, recording mode, storage limits, naming). The Wrapper applies defaults for newly discovered devices and merges overrides when present.

**Why this priority**: Provides control and resource governance while maintaining easy onboarding.

**Independent Test**: Provide a configuration file with defaults and a device-specific override; discover the device; verify the resulting Agent instance uses the merged configuration and respects storage/time constraints.

**Acceptance Scenarios**:

1. **Given** a default configuration and device-specific override, **When** the device is discovered, **Then** the Agent instance is created with merged settings (override wins), including storage quota and naming.
2. **Given** a device-specific override is removed, **When** reconciliation runs, **Then** the Agent instance reverts to defaults without downtime.

---

[Add more user stories as needed, each with an assigned priority]

### Edge Cases

- Multiple devices discovered simultaneously; ensure no duplicate Agent instances are created.
- Devices flapping online/offline; ensure no thrashing (rate limit restarts, backoff policy).
- Network segments where multicast is restricted; support alternative discovery via configured CIDRs.
- Invalid or missing credentials; devices remain pending without spawning until credentials supplied.
- Resource limits reached (CPU, memory, disk); enforce max concurrent Agent instances and bounded storage.
- Duplicate device identifiers (e.g., same IP seen on different interfaces); deduplicate by stable device ID.

## Requirements *(mandatory)*

<!--
  ACTION REQUIRED: The content in this section represents placeholders.
  Fill them out with the right functional requirements.
-->

### Functional Requirements

- **FR-001**: The Wrapper MUST discover ONVIF-compatible devices on configured networks, supporting multicast-based discovery and configurable CIDR scanning.
- **FR-002**: The Wrapper MUST validate device reachability and credentials before spawning an Agent instance.
- **FR-003**: For each eligible device, the Wrapper MUST spawn exactly one Kerberos Agent instance and track its lifecycle state (starting, healthy, unhealthy, stopped).
- **FR-004**: The Wrapper MUST prevent duplicate Agent instances for the same device by using a stable device identifier.
- **FR-005**: The Wrapper MUST maintain a reconciliation loop to restore desired state when devices or instances change (offline/online, crash/healthy).
- **FR-006**: The Wrapper MUST apply a default Agent configuration and merge device-specific overrides when present.
- **FR-007**: The Wrapper MUST enforce bounded storage and resource limits per Agent (e.g., max recordings size, max concurrent Agents), emitting warnings before limits are hit.
- **FR-008**: The Wrapper MUST emit structured logs and basic metrics for discovery events, instance lifecycle changes, and errors without leaking secrets.
- **FR-009**: The Wrapper SHOULD provide a status interface (CLI or JSON output) listing discovered devices, pending devices, and active Agent instances.
- **FR-010**: The Wrapper MUST stop or pause Agent instances when devices become unreachable and resume/restart when they return.
- **FR-011**: The Wrapper MUST support naming conventions for Agent instances (e.g., agent-<stable-id>) and avoid collisions.
- **FR-012**: The Wrapper MUST redact credentials and sensitive data from all logs and status outputs.
- **FR-013**: The system MUST avoid silent recording drops; on failure, log an error and surface an event.
- **FR-014**: The system MUST provide health/readiness signals for the Wrapper itself.
- **FR-015**: The Wrapper MUST support single-host operation; multi-host orchestration MAY be considered in future scope.
- **FR-016**: The Wrapper MUST support configurable discovery intervals and backoff to avoid network saturation.
- **FR-017**: The Wrapper MUST record and expose correlation identifiers between devices and spawned Agent instances.
- **FR-018**: The Wrapper MUST default to secure settings and warn when default credentials are used.
- **FR-019**: The Wrapper MUST ensure crash-safe handling when finalizing recording segments (via the spawned Agents’ configuration), and SHOULD select defaults that favor data integrity.
- **FR-020**: The Wrapper MUST spawn Agent instances using [NEEDS CLARIFICATION: default mode - containerized or local process as default? Both SHOULD be supported via configuration].

### Key Entities

- **Device**: Represents an ONVIF-compatible camera discovered by the Wrapper. Key attributes: stable device ID, network address(es), capabilities, credential status, last seen.
- **AgentInstance**: Represents a spawned Kerberos Agent managing a single device. Key attributes: instance ID, device ID, mode (process/container), state, health, assigned ports, storage quota.
- **WrapperConfig**: Global/default configuration, discovery settings (intervals, methods, CIDRs), resource limits, naming conventions, credential sources, per-device overrides.
- **ReconcileEvent**: Events produced during reconciliation (device offline/online, agent start/stop/restart), with timestamps and correlation IDs.

## Success Criteria *(mandatory)*

<!--
  ACTION REQUIRED: Define measurable success criteria.
  These must be technology-agnostic and measurable.
-->

### Measurable Outcomes

- **SC-001**: Newly discovered eligible devices are provisioned with a healthy Agent instance within ≤ 30 seconds (median) from first discovery.
- **SC-002**: Discovery scans complete within ≤ 60 seconds per configured subnet without saturating host resources (observed CPU ≤ 70%, memory ≤ 70%).
- **SC-003**: 95% of eligible devices maintain a single healthy Agent instance without duplication during 24-hour operation.
- **SC-004**: 0 incidents of secrets/credentials appearing in logs or status outputs during test and production runs.
- **SC-005**: Wrapper health/readiness reports successful within ≤ 1 second p95 under normal load.
