# tep-operator

Kubernetes operator that acts as the supervisory controller for the Tennessee Eastman Process (TEP) plant. Monitors plant variables via gRPC, evaluates whether they are within acceptable ranges, and adjusts controller parameters when needed.

## Context

This operator is part of the **TEP CPS Lab** — a lab where Kubernetes acts as the supervisory system for a simulated chemical plant (a Cyber-Physical System, not a "digital twin" — there is no physical reference plant). The plant runs on its own, with PID controllers running and random disturbances. The operator doesn't push config — it observes, evaluates, decides, and acts.

```
kubectl apply -f plcmachine.yaml
  -> operator reads the supervisory policy (ranges, rules)
    -> connects to the plant via gRPC
      -> reads XMEAS (measured variables)
        -> if something leaves its range, adjusts a controller
          -> records the read state as memory (status)
```

## Quick start

```bash
# Prerequisites: Go 1.25+, Docker, Kind, kubectl

# Generate code and manifests
make generate manifests

# Build the image
make docker-build IMG=controller:latest

# Create a Kind cluster and deploy
kind create cluster --name tep-lab
kind load docker-image controller:latest --name tep-lab
make install      # CRDs
make deploy IMG=controller:latest

# Create a PLCMachine
kubectl apply -f config/samples/infrastructure_v1alpha1_plcmachine.yaml
kubectl get plcmachines
```

## CRD: PLCMachine

The operator's single resource. `.spec` defines the **supervisory policy** — acceptable ranges and response rules. `.status` is the operator's **memory** — latest readings, trends, actions taken.

```yaml
apiVersion: infrastructure.greenlabs.io/v1alpha1
kind: PLCMachine
metadata:
  name: tep-baseline
spec:
  plantAddress: "te-plant.default.svc:50051"
  operatingRanges:
    - name: reactor_pressure
      xmeasIndex: 6          # XMEAS(7) — reactor pressure
      min: 2600.0
      max: 2800.0
  responseRules:
    - name: pressure_high
      watchRef: reactor_pressure
      condition: above_max
      controllerID: pressure_reactor
      parameter: kp
      adjustValue: 0.15
  monitoringInterval:
    baseMs: 2000
    transientMs: 200
```

Full details in [docs/03-crd-plcmachine.md](docs/03-crd-plcmachine.md).

## Documentation

| Doc                                                        | Content                                                           |
| ---------------------------------------------------------- | ------------------------------------------------------------------ |
| [01 — Overview](docs/01-visao-geral.md)                    | What this repo is, where it fits, current state                  |
| [02 — Project anatomy](docs/02-anatomia-do-projeto.md)     | File map: what to edit vs. what Kubebuilder generates             |
| [03 — CRD PLCMachine](docs/03-crd-plcmachine.md)           | Spec, status, phases, gRPC mapping                                |
| [04 — Reconciliation](docs/04-reconciliacao.md)            | How the reconciler works (design, flow, idempotency)              |

## Lab repositories

| Repo                                                                                    | What it does                       |
| --------------------------------------------------------------------------------------- | ----------------------------------- |
| [spec-tennessee-eastman](https://github.com/Green-Cinnamon-Labs/spec-tennessee-eastman) | Issues, specs, decisions            |
| [tep-plant](https://github.com/Green-Cinnamon-Labs/tep-plant)   | TEP plant (Rust) + gRPC server    |
| **tep-operator**                                                            | **This repo** — K8s operator (Go)  |
| [tep-supervisor](https://github.com/Green-Cinnamon-Labs/tep-supervisor)         | Cluster infra (Kind, manifests) |

## Structure

```
api/v1alpha1/          <- CRD types (PLCMachine spec/status)
internal/controller/   <- Reconciler (supervisory logic)
cmd/main.go            <- Manager entry point
config/                <- Kustomize: CRD, RBAC, deployment
docs/                  <- Project documentation
```

> To understand what each file does and what is Kubebuilder boilerplate, see [docs/02-anatomia-do-projeto.md](docs/02-anatomia-do-projeto.md).

## Development status

- [x] Kubebuilder scaffold
- [x] PLCMachine CRD with supervisory policy (#37)
- [ ] Reconciler with gRPC client (#38)
- [ ] Kind cluster setup (#39)
- [ ] Deploy plant + operator (#40)
- [ ] E2E test: CR -> disturbance -> reconciliation (#41)
