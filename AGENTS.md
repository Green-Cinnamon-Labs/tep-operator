# CLAUDE.md

This file provides guidance to OpenAI Codex when working with code in this repository.

## Architecture

This is a **Kubebuilder operator** that acts as a supervisory controller for the Tennessee Eastman Process (TEP) digital twin. It bridges Kubernetes-native policy (CRDs) with a remote gRPC plant simulation.

### Control flow

```
PLCMachine CR (spec/status)
       ↕ controller-runtime
PLCMachineReconciler (internal/controller/)
       ↕ gRPC
PlantService (tep-plant, Rust, port 50051)
```

The reconciler follows an **Observe → Evaluate → Decide → Act → Record** loop, requeuing every `spec.monitoringInterval.baseMs` ms (default 2 s) or `transientMs` (default 200 ms) during transients.

### Current implementation phase

- **Phase 1 (done):** Passive observation — every reconcile reads all 41 XMEAS + 12 XMV from plant, stores full snapshot in `status.observation`, evaluates `spec.operatingRanges`, and sets phase to `Stable`.
- **Phase 2 (pending):** Supervisory logic — evaluate `spec.responseRules`, call `UpdateController` via gRPC when variables exit ranges, record `status.lastAction`.

### Key files

| File                                                     | Role                                                                                     |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `api/v1alpha1/plcmachine_types.go`                       | CRD spec and status structs; all `+kubebuilder:` markers                                 |
| `internal/controller/plcmachine_controller.go`           | Reconciliation loop                                                                      |
| `internal/grpc/client.go`                                | `PlantClient` — wraps gRPC dial, `GetPlantStatus`, `ListControllers`, `UpdateController` |
| `internal/grpc/gen/tepv1/`                               | Protobuf-generated stubs (do not edit manually)                                          |
| `proto/tep/v1/plant.proto`                               | Source of truth for plant gRPC API                                                       |
| `cmd/main.go`                                            | Manager bootstrap (flags, scheme registration, health probes)                            |
| `config/samples/infrastructure_v1alpha1_plcmachine.yaml` | Reference CR for local dev                                                               |

### CRD design

`PLCMachine` spec encodes **supervisory policy**:
- `plantAddress` — gRPC endpoint of the TEP plant
- `operatingRanges` — acceptable bounds per XMEAS index (0-based; XMEAS(i+1) in TEP notation)
- `responseRules` — actions to take when a range is violated (`above_max`/`below_min` → adjust controller parameter)
- `monitoringInterval` — base/transient polling cadence

`PLCMachine` status encodes **operator memory**:
- `phase` — `Pending | Stable | Transient | Alarm | Shutdown`
- `observation` — full passive snapshot (all 41 XMEAS, 12 XMV, `derivNorm`)
- `variables` — per-range evaluation results (value, trend, in-range flag)
- `lastAction` — most recent supervisory intervention

### gRPC client pattern

`PlantClient.Connect` opens a new connection per reconcile with a 5 s dial timeout (insecure credentials for in-cluster). Each call gets the context passed from the reconciler; the client is closed with `defer client.Close()`.

### Kubernetes deployment

- Namespace: `tep-operator-system`
- Metrics endpoint: HTTPS on port 8443 (authn/authz enabled)
- Health probes: `/healthz` and `/readyz` on port 8081
- Leader election: disabled by default locally, enabled in `config/manager/manager.yaml`
- Docker targets in Makefile are commented out; builds are done locally with `make build`, image loaded into Kind manually.

## Windows / Codespace constraint

The local development machine is Windows. Do not assume that all Kubebuilder generation commands can run locally.

`make generate` and `make manifests` should be run in a Linux environment, preferably GitHub Codespace, when local Windows execution is unreliable.

After generating files in Codespace, pull the generated changes back into the local workspace before continuing.


## Docs

`docs/` contains Portuguese-language design docs covering overview, file anatomy, CRD spec, and reconciliation logic. Consult before making structural changes.

## Agent Rules

- Do not change `api/v1alpha1/` types or `+kubebuilder:` markers without stating the CRD impact.
- Any change to CRD types or markers requires `make generate && make manifests` before commit.
- Do not edit generated files under `internal/grpc/gen/` manually.
- Do not change `proto/tep/v1/plant.proto` without checking impact on `tep-plant` and `tep-ihm`.
- Keep reconciliation logic inside `internal/controller/plcmachine_controller.go`.
- Keep gRPC access isolated in `internal/grpc/client.go`; do not spread gRPC calls through the reconciler.
- Preserve the operator loop: Observe → Evaluate → Decide → Act → Record.
- Do not introduce supervisory actions before confirming how they affect `status.lastAction`, `phase`, and `variables`.
- Do not run `kubectl`, Kind, Docker, deploy, or cluster-changing commands unless explicitly authorized.
- Prefer suggesting validation commands instead of executing environment-changing commands.
- Before editing, provide a short plan and list the files to be modified.
- For cross-repository decisions, follow the root `CLAUDE.md` / `AGENTS.md` propagation rules.
