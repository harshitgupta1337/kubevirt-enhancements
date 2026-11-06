# `virt-handler` Virtualization-Stack Dependency Analysis

## 1. Executive summary

`virt-handler` owns Kubernetes-facing, node-local VMI orchestration, but its
controlling runtime model is tightly coupled to the current Libvirt/QEMU
launcher stack. The shared `api.Domain` model is sufficiently general to remain
the handler-launcher representation of desired and observed VM state. The
coupling is in QEMU- and Libvirt-specific methods and fields around that model:
the launcher command API exposes QEMU version, SEV, dirty-rate, and checkpoint
operations; migration knows Libvirt ports and socket paths; resource adjustment
discovers QEMU and `virtqemud` processes; and node capability discovery consumes
Libvirt XML.

The existing `hypervisor.VirtRuntime` abstraction demonstrates a useful
extension point, but it covers only housekeeping and process resource
adjustment. Both its KVM and MSHV implementations still depend on QEMU,
`virtqemud`, and QEMU-specific thread handling. Supporting an independent stack
therefore requires typed capability extensions and a typed node-runtime adapter,
not replacement of `api.Domain` or only additional branches in `VirtRuntime`.

Core `virt-handler` should retain VMI reconciliation, privileged host
operations, safety validation, shutdown and migration policy, and status
reporting. `api.Domain` remains the normalized runtime contract. The selected
stack should translate its native VMM state into that contract and provide
lifecycle mechanics, named process and endpoint roles, migration transport
requirements, and versioned node capabilities.

## 2. Control flow

```text
Domain notification or VMI event
  -> VirtualMachineController.Execute / execute
    pkg/virt-handler/vm.go
    -> read VMI and shared api.Domain from local caches
    -> sync
      -> derive lifecycle action from api.Domain state and reason
      -> starting VMI
        -> mount storage and configure pod networking/devices/cgroups
        -> LauncherClient.SyncVirtualMachine
      -> running VMI
        -> VirtRuntime.HandleHousekeeping / AdjustResources
        -> LauncherClient CPU, memory, hotplug, guest-agent operations
      -> stopping VMI
        -> LauncherClient.ShutdownVirtualMachine / KillVirtualMachine
      -> update VMI status from api.Domain and Libvirt/QEMU statistics

Migration source or target event
  -> MigrationSourceController or MigrationTargetController
    -> inspect launcher process and api.Domain state
    -> create TCP-to-Unix migration proxies
    -> LauncherClient.SyncMigrationTarget / MigrateVirtualMachine /
      FinalizeVirtualMachineMigration
    -> repair passt sockets in the launcher mount namespace
```

The main reconciler is
[`VirtualMachineController.execute`](../pkg/virt-handler/vm.go#L312-L426), and
[`VirtualMachineController.sync`](../pkg/virt-handler/vm.go#L1320-L1497)
turns VMI and `api.Domain` state into lifecycle actions. The launcher command
boundary is [`LauncherClient`](../pkg/virt-handler/cmd-client/client.go#L82-L137).

## 3. Findings

| ID | Classification | Severity | Exact file and symbol | Assumption | Recommended disposition |
| --- | --- | --- | --- | --- | --- |
| H1 | Explicit current-stack dependency | Blocking | [`LauncherClient`](../pkg/virt-handler/cmd-client/client.go#L82-L137) | The common handler ABI mixes general lifecycle operations with QEMU version, dirty-rate, SEV, migration-tuning, and backend checkpoint operations. | Retain `api.Domain` and general lifecycle methods; move optional backend operations behind typed, versioned capability interfaces. |
| H2 | Reusable stack-neutral contract | Minor | [`VirtualMachineController.execute`](../pkg/virt-handler/vm.go#L312-L426), [`VirtualMachineController.sync`](../pkg/virt-handler/vm.go#L1320-L1497), [`DomainStatus`](../pkg/virt-launcher/virtwrap/api/schema.go#L124-L133) | Reconciliation uses `api.Domain` existence, lifecycle state, reason, metadata, and deletion state. These concepts apply across VMMs. | Preserve `api.Domain` as the shared model. Each launcher backend translates native state and reasons into it; add fields only when they remain meaningful across stacks. |
| H3 | Explicit current-stack dependency | Blocking for migration | [`GetMigrationPortsList`](../pkg/virt-handler/migration-proxy/migration-proxy.go#L36-L91), [`handleTargetMigrationProxy`](../pkg/virt-handler/migration-target.go#L739-L759) | Migration uses Libvirt ports 49152/49153 and proxies to launcher Unix sockets, including `virtqemud-sock`. | Core owns migration policy and phase transitions; the stack supplies a typed transport plan and executes backend migration operations. |
| H4 | Explicit current-stack dependency | Significant | [`KvmVirtRuntime.AdjustResources`](../pkg/hypervisor/kvm/runtime.go#L49-L95), [`MshvVirtRuntime.AdjustResources`](../pkg/hypervisor/mshv/runtime.go#L47-L93) | Resource adjustment searches for QEMU or `virtqemud` processes and changes their rlimits. MSHV retains the same process model. | Stack runtime identifies managed processes and returns bounded resource-adjustment targets; core applies authorized node policy. |
| H5 | Explicit current-stack dependency | Significant | [`KvmVirtRuntime.HandleHousekeeping`](../pkg/hypervisor/kvm/runtime.go#L97-L127), [`MshvVirtRuntime.HandleHousekeeping`](../pkg/hypervisor/mshv/runtime.go#L95-L112) | CPU housekeeping reads general CPU tuning intent from `api.Domain`, but implements it through QEMU vCPU, emulator-thread, and PIT-thread discovery. | Retain CPU tuning in `api.Domain`; move backend thread discovery and affinity mechanics behind the stack runtime. |
| H6 | Implicit stack-specific behavior | Significant | [`passthSocketDirOnHost`](../pkg/virt-handler/unsafepath.go#L27-L42), [`handleTargetMigrationProxy`](../pkg/virt-handler/migration-target.go#L739-L759) | Handler enters the launcher root through `/proc/<pid>/root` and knows `/var/run/libvirt/qemu/run/passt` and `/var/run/libvirt/virtqemud-sock`. | Isolation remains core infrastructure; plugins return named endpoint purposes, while core resolves and validates paths inside the launcher root. |
| H7 | Reusable stack-neutral contract | Minor | [`VirtualMachineController.processVmShutdown`](../pkg/virt-handler/vm.go#L1549-L1574), [`helperVmShutdown`](../pkg/virt-handler/vm.go#L1621-L1652) | Graceful shutdown timing, metadata, acknowledgement, and terminal lifecycle state are general VM concepts represented by `api.Domain`. | Keep grace-period policy and `api.Domain` metadata in core; require every launcher backend to implement the existing shutdown semantics. |
| H8 | Explicit current-stack dependency | Significant | [`virt-handler` capability loading](../cmd/virt-handler/virt-handler.go#L345-L374) | Startup requires a Libvirt `capabilities.xml` and derives machine and CPU information from Libvirt XML. | A node stack provider publishes a versioned capability document; core maps approved capability names to node labels. |
| H9 | Implicit stack-specific behavior | Significant | [`VolumeRenderer` runtime mount](../pkg/virt-controller/services/rendervolumes.go#L69-L96), [`handleTargetMigrationProxy`](../pkg/virt-handler/migration-target.go#L739-L759) | Handler assumes launcher runtime sockets exist at fixed Libvirt paths, even though it accesses them indirectly through the launcher process root. | Treat runtime endpoints as typed stack outputs, not handler constants. |
| H10 | Extension point for future backends | Blocking | [`VirtRuntime`](../pkg/hypervisor/virt-runtime.go#L33-L46) | The current abstraction covers only housekeeping and resource adjustment and defaults unknown names to KVM. Its `api.Domain` input is reusable, but its implementations encode backend process and thread assumptions. | Replace fallback with explicit selection and expand the adapter around process, thread, endpoint, capability, and migration mechanics while retaining `api.Domain`. |

## 4. Ownership boundary

The handler should remain responsible for Kubernetes and node orchestration:

- Watching VMIs and launcher pods, workqueue retries, events, conditions, and
  status updates.
- Detecting the launcher pod PID and safely resolving its mount and network
  namespaces.
- Applying authorized mounts, cgroup, device, security, and network operations
  from validated typed requirements.
- Enforcing shutdown deadlines and migration policy, coordinating source and
  target nodes, and reporting progress.

The selected stack implementation should provide:

- Translation between native VMM configuration/state and the shared
  `api.Domain` model, plus implementation of its lifecycle operations.
- Backend process roles and resource targets instead of QEMU and `virtqemud`
  process-name discovery in core paths.
- Runtime endpoint purposes and migration transport plans instead of fixed
  Libvirt socket paths and ports.
- Backend thread roles and affinity requirements instead of QEMU vCPU,
  emulator, and PIT-thread assumptions.
- Versioned node capabilities and compatibility checks instead of requiring
  Libvirt `capabilities.xml`.

The plugin must not receive unrestricted host access or arbitrary command
execution. It should return typed requirements; `virt-handler` validates those
requirements and performs privileged node operations. Backend mechanics that
cannot be represented declaratively should execute inside the stack-owned
launcher runtime through the versioned launcher ABI.

## 5. Current MSHV boundary

MSHV demonstrates partial backend selection, not handler independence. Both
[`KvmVirtRuntime`](../pkg/hypervisor/kvm/runtime.go#L35-L127) and
[`MshvVirtRuntime`](../pkg/hypervisor/mshv/runtime.go#L34-L112) locate QEMU or
`virtqemud` and implement QEMU-shaped CPU and process handling. Their shared
use of `api.Domain` is intentional and does not itself create coupling.
`GetVirtRuntime` also silently selects KVM for unknown names
([pkg/hypervisor/virt-runtime.go](../pkg/hypervisor/virt-runtime.go#L38-L46)).
The abstraction therefore varies a small amount of node housekeeping while the
controlling lifecycle and migration contracts remain Libvirt/QEMU-specific.

## 6. Tests that pin the current behavior

- The MSHV runtime test constructs a process named `virtqemud` and requires
  `AdjustResources` to apply the memlock limit to that PID
  ([pkg/hypervisor/mshv/runtime_test.go](../pkg/hypervisor/mshv/runtime_test.go#L124-L155)).
  This confirms that current MSHV selection does not remove the Libvirt process
  assumption.
- Migration proxy tests create a `virtqemud-sock`, identify direct migration as
  port 49152, and verify forwarding between the manager's TCP and Unix
  endpoints
  ([pkg/virt-handler/migration-proxy/migration-proxy_test.go](../pkg/virt-handler/migration-proxy/migration-proxy_test.go#L107-L168)).
- VM reconciliation tests express general lifecycle behavior with `api.Domain`
  states:
  a running orphan is killed, a crashed domain is deleted, and a running domain
  with grace-period metadata receives `ShutdownVirtualMachine`
  ([pkg/virt-handler/vm_test.go](../pkg/virt-handler/vm_test.go#L357-L405),
  [pkg/virt-handler/vm_test.go](../pkg/virt-handler/vm_test.go#L498-L530)).
- Node-labeller tests require host CPU model and supported machine-type labels
  derived from Libvirt XML model types
  ([pkg/virt-handler/node-labeller/node_labeller_test.go](../pkg/virt-handler/node-labeller/node_labeller_test.go#L129-L145)).

The VM reconciliation cases should remain core conformance tests for every
stack because they validate the shared `api.Domain` contract. The MSHV process
and migration proxy cases pin current-stack behavior and should become adapter
compatibility tests. New stack tests should use named process roles, named
endpoints, and stack capability documents around the same `api.Domain` model.

## 7. Prioritized refactorings

1. Formalize `api.Domain` as the versioned, stack-neutral handler-launcher
  contract and document which fields every stack must implement.
2. Split QEMU-, SEV-, dirty-rate-, checkpoint-, and migration-specific methods
  from the common `LauncherClient` into typed capability interfaces.
3. Replace fixed migration ports and socket paths with a validated transport
   plan containing named endpoints, direction, protocol, and security mode.
4. Expand `VirtRuntime` to identify backend process and thread roles, then move
   QEMU, `virtqemud`, vCPU, emulator, and PIT discovery into the current-stack
   implementation.
5. Replace `capabilities.xml` as the generic input with a versioned stack
   capability document and explicit stack lookup without KVM fallback.
6. Add conformance tests that drive the same handler lifecycle state machine
   through the current Libvirt adapter and a minimal non-Libvirt fake adapter.

## 8. Migration involvement and decoupling

### 8.1 How `virt-handler` carries out migration today

`virt-handler` coordinates migration on both nodes, but it does not copy guest
memory or device state itself. The source and target `virt-launcher` processes
perform the VMM operation. The handlers prepare node resources, establish the
network path between launchers, invoke launcher RPCs, observe `api.Domain`, and
publish migration status.

```text
virt-controller
  -> creates and schedules the target launcher pod
  -> initializes VMI migration state and hands work to both nodes

target virt-handler
  -> mounts target volumes and configures network and device ownership
  -> calls target launcher: SyncMigrationTarget
  -> starts TLS TCP listeners that forward to target launcher Unix sockets
  -> publishes target address and allocated ports in VMI migration status

source virt-handler
  -> waits for target address and ports
  -> starts local Unix listeners forwarding through TLS to target handler
  -> applies source migration policy and runs pre-migration hooks
  -> calls source launcher: MigrateVirtualMachine

source and target launchers
  -> perform the VMM-specific state transfer
  -> publish progress and completion through api.Domain migration metadata

source and target virt-handler
  -> observe domain state, handle aborts, transfer node ownership
  -> call FinalizeVirtualMachineMigration and clean up proxies and volumes
```

On the target, [`processVMI`](../pkg/virt-handler/migration-target.go#L856-L985)
mounts storage, configures the target network and device ownership, invokes
`SyncMigrationTarget`, adjusts process resources, and then creates migration
listeners. [`handleTargetMigrationProxy`](../pkg/virt-handler/migration-target.go#L739-L759)
collects the target launcher sockets. [`updateStatus`](../pkg/virt-handler/migration-target.go#L313-L420)
publishes the handler address and dynamically allocated TCP ports so the source
can connect.

On the source, [`handleSourceMigrationProxy`](../pkg/virt-handler/migration-source.go#L462-L484)
waits for those advertised endpoints and starts local Unix-to-remote-TCP
forwarding. [`migrateVMI`](../pkg/virt-handler/migration-source.go#L488-L604)
resolves migration policy such as bandwidth, downtime, post-copy, compression,
and stall detection, performs passt repair and node hooks, and invokes
`MigrateVirtualMachine` on the source launcher. Abort requests are forwarded to
the launcher through `CancelVirtualMachineMigration`.

The handlers then converge Kubernetes state from the shared domain model. The
source copies progress from `api.Domain` migration metadata and waits for the
target to report a ready domain
([pkg/virt-handler/migration-source.go](../pkg/virt-handler/migration-source.go#L147-L286)).
The target detects the received and active domain, transfers VMI ownership to
the target node, and invokes final CPU, memory, interface, and launcher
finalization
([pkg/virt-handler/migration-target.go](../pkg/virt-handler/migration-target.go#L278-L384),
[pkg/virt-handler/migration-target.go](../pkg/virt-handler/migration-target.go#L1205-L1244)).

### 8.2 Where migration is tied to Libvirt and QEMU

The high-level source/target sequencing is reusable. These details are not:

1. **Fixed endpoint vocabulary.** The handler defines Libvirt direct and block
  migration ports 49152 and 49153
  ([pkg/virt-handler/migration-proxy/migration-proxy.go](../pkg/virt-handler/migration-proxy/migration-proxy.go#L36-L91)).
  The target always includes `/var/run/libvirt/virtqemud-sock` and derives
  additional socket names from those port numbers. This assumes one Libvirt
  control connection, one QEMU memory-state stream, and optionally one block
  migration stream.
2. **Shared Libvirt path construction.** Both handler and launcher use
  `ConstructProxyKey` and `SourceUnixFile`. The source launcher converts those
  paths into `unix://` memory and disk URIs
  ([pkg/virt-launcher/virtwrap/live-migration-source.go](../pkg/virt-launcher/virtwrap/live-migration-source.go#L1077-L1125)),
  while the target launcher creates matching legacy TCP proxy sockets
  ([pkg/virt-launcher/virtwrap/live-migration-target.go](../pkg/virt-launcher/virtwrap/live-migration-target.go#L238-L264)).
3. **Libvirt migration invocation.** The source launcher creates
  `libvirt.DomainMigrateParameters`, migratable domain XML, QEMU migration
  flags, disk targets, and QEMU connection URIs, then calls
  `MigrateToURI3`
  ([pkg/virt-launcher/virtwrap/live-migration-source.go](../pkg/virt-launcher/virtwrap/live-migration-source.go#L1077-L1130),
  [pkg/virt-launcher/virtwrap/live-migration-source.go](../pkg/virt-launcher/virtwrap/live-migration-source.go#L1340-L1372)).
4. **QEMU-specific policy knobs.** Auto-converge, post-copy, parallel migration
  threads, dirty-rate stall detection, downtime tuning, and compression are
  sent through the common migration RPC even though another VMM may support a
  different subset or use different semantics
  ([pkg/virt-handler/cmd-client/client.go](../pkg/virt-handler/cmd-client/client.go#L61-L80),
  [pkg/virt-handler/migration-source.go](../pkg/virt-handler/migration-source.go#L520-L581)).
5. **Backend-specific node repair.** Handler migration directly repairs passt
  sockets under `/var/run/libvirt/qemu/run/passt` and adjusts the target QEMU
  or `virtqemud` process. Those are current-stack implementation details, not
  general migration stages.

`api.Domain` does not need to be removed. Its `MigrationMetadata` contains the
general orchestration facts the handlers need: migration UID, start and end
times, success or failure, abort status, failure reason, and mode
([pkg/virt-launcher/virtwrap/api/schema.go](../pkg/virt-launcher/virtwrap/api/schema.go#L455-L464)).
Each launcher backend can translate native VMM progress into those existing
fields. Backend-specific progress may be exposed through optional capability
interfaces without changing the core lifecycle model.

### 8.3 How migration should be decoupled

Keep the current responsibility split, but replace implicit Libvirt conventions
with a typed migration capability negotiated for the VMI's selected stack.

```go
type MigrationEndpointPurpose string

const (
   MigrationControl MigrationEndpointPurpose = "control"
   MigrationState   MigrationEndpointPurpose = "state"
   MigrationDisk    MigrationEndpointPurpose = "disk"
)

type MigrationEndpointRequirement struct {
   Name      string
   Purpose   MigrationEndpointPurpose
   Direction MigrationDirection
   Required  bool
}

type MigrationTransportPlan struct {
   StackID      string
   StackVersion string
   Transport    MigrationTransport
   Endpoints    []MigrationEndpointRequirement
}

type MigrationCapability interface {
   PlanTarget(vmi *v1.VirtualMachineInstance) (MigrationTransportPlan, error)
   SupportedOptions() MigrationOptionCapabilities 
}
```

The endpoint `Name` is an opaque identifier shared by the source and target
launcher adapters; `Purpose` lets core apply policy. A plugin must not return an
arbitrary host path or listening address. On each node, trusted handler code
resolves endpoint names beneath an approved launcher runtime directory, starts
TLS proxies, allocates host ports, and publishes endpoint-name-to-address
mappings. This preserves handler ownership of privileged host networking and
certificate policy.

The target sequence becomes:

1. Resolve the selected stack's migration capability and reject unsupported or
  incompatible source/target stack versions before preparation.
2. Keep core volume, network, device, hook, and status preparation.
3. Ask the stack for endpoint requirements and supported migration options.
4. Resolve and validate local endpoints, start core-owned TLS proxies, and
  publish named advertised endpoints.
5. Call `SyncMigrationTarget` with the resolved transport plan. The target
  launcher translates it into native VMM listeners and configuration.

The source sequence becomes:

1. Wait for the target's named advertised endpoints and verify that they match
  the negotiated plan.
2. Start core-owned source proxies and pass their resolved local endpoints to
  the source launcher.
3. Filter cluster migration policy through `SupportedOptions`; fail explicitly
  when a required option is unavailable instead of passing QEMU knobs to every
  stack.
4. Call `MigrateVirtualMachine` with general policy plus the resolved transport
  plan. The launcher adapter performs Libvirt `MigrateToURI3`, an OpenVMM
  transfer, or another backend operation.
5. Continue reading progress and completion from `api.Domain` migration
  metadata, then run the existing ownership transfer, finalization, and
  cleanup state machines.

The existing `migrationProxyManager` can remain core after its API changes from
port-derived socket keys to named endpoints. The first implementation should be
a Libvirt/QEMU adapter that emits `control`, `state`, and optional `disk`
requirements and reproduces today's paths and behavior. Conformance tests should
then run handler orchestration against a fake non-Libvirt plan, while existing
49152/49153, `virtqemud-sock`, and `MigrateToURI3` tests remain adapter-specific
compatibility tests.

---

This report is based on read-only static analysis of the repository as of
2026-08-27. Architectural recommendations are identified separately from
verified implementation facts.
