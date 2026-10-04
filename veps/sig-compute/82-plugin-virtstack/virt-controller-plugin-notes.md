# Virt-launcher Pod rendering plugin

## Purpose

Virt-controller must be able to create virt-launcher Pods for different
virtualization stacks. The rendering flow therefore separates common KubeVirt
Pod construction from virtualization-stack-specific completion.

Virt-controller first constructs a functional, stack-neutral base Pod. It then
invokes a launcher manifest renderer, referred to here as the virtualization
plugin, to add the fields required by a particular virtualization stack.
Virt-controller validates the completed Pod before using it.

The in-tree renderer API is intentionally shaped so that it can later be
represented by protobuf messages and invoked over gRPC without changing the
logical contract.

## Rendering flow

The high-level flow is:

1. Resolve controller-local data.
2. Construct the stack-neutral base Pod.
3. Build the plugin request.
4. Invoke the selected virtualization plugin.
5. Apply generic post-render policy.
6. Validate the completed Pod.
7. Return the Pod for creation.

Conceptually:

```go
basePod := renderBaseLauncherManifest(vmi, resolvedData, mode)

request := &LauncherManifestRenderRequest{
    VMI: vmi,
    Pod: basePod,
    Context: &LauncherManifestRendererContext{
        Configuration: effectiveKubeVirtConfiguration,
        Mode:          mode,
    },
}

if err := renderer.Render(ctx, request); err != nil {
    return nil, err
}

applyGenericPostRenderPolicy(request.Pod)
if err := validateCompletedLauncherPod(vmi, request.Pod); err != nil {
    return nil, err
}

return request.Pod, nil
```

An in-tree renderer updates the request Pod in place. A future gRPC plugin will
return the completed Pod explicitly.

## Base Pod responsibilities

Virt-controller owns all virtualization-stack-agnostic Pod construction. The
base Pod includes common KubeVirt behavior and all information that required
controller-local clients, informer stores, or resolved object state.

The base Pod contains:

- Pod metadata, ownership, common labels, and common annotations.
- The `compute` container skeleton.
- Common compute security context, probes, ports, capabilities, mounts, volume
  devices, and environment variables such as `POD_NAME`.
- User-specified CPU, memory, storage, device, and DRA resource requirements
  that are independent of the virtualization stack.
- VMI volumes after controller-side PVC and DataVolume resolution.
- ContainerDisk, kernel boot, VirtioFS, serial-console, and hook-sidecar
  containers and volumes.
- Image pull secrets.
- Service-account behavior.
- DNS, scheduler, priority, tolerations, topology spread, and resource claims.
- User and cluster node selectors.
- Generic KubeVirt scheduling constraints such as dedicated-CPU and realtime
  selectors.
- User affinity and generic persistent-reservation anti-affinity.

The base `compute` container deliberately does not contain:

- A virtualization-stack launcher image or image pull policy.
- A stack-specific command or arguments.
- Stack-specific environment variables.
- Stack-specific mounts, such as the libvirt runtime directory.
- Hypervisor resources or stack-specific memory overhead.
- Hypervisor capability selectors or affinity.

Controller-local resolution results such as image ID maps, backend-storage PVC
names, Kubernetes clients, informer stores, and `ClusterConfig` are not passed
to the plugin. Their effects must already be represented in the base Pod.

## Plugin request

The logical request contains four values:

| Field | Type | Description |
|---|---|---|
| `vmi` | `VirtualMachineInstance` | The VMI whose launcher Pod is being rendered. |
| `pod` | `Pod` | The stack-neutral base Pod to complete. |
| `configuration` | `KubeVirtConfiguration` | The effective, defaulted, serializable KubeVirt configuration. |
| `mode` | `LauncherRenderMode` | The lifecycle operation being rendered. |

Supported modes are:

- `launch`: render a normal virt-launcher Pod.
- `migration-target`: render the target virt-launcher Pod for migration.
- `provisioning`: render the temporary Pod used for storage provisioning.

Go's `context.Context` is passed separately for cancellation, deadlines,
tracing, and request metadata. It is not part of the serialized plugin payload.

The current in-tree API is:

```go
type LauncherManifestRenderRequest struct {
    VMI     *virtv1.VirtualMachineInstance
    Pod     *corev1.Pod
    Context *LauncherManifestRendererContext
}

type LauncherManifestRendererContext struct {
    Configuration *virtv1.KubeVirtConfiguration
    Mode          LauncherRenderMode
}

type LauncherManifestRenderer interface {
    Render(context.Context, *LauncherManifestRenderRequest) error
}
```

## Plugin responsibilities

The plugin completes the base Pod for one virtualization stack. It may replace
or extend stack-owned fields, but it must preserve the common KubeVirt behavior
already present in the base Pod.

A plugin typically supplies:

- The compute-container image and image pull policy.
- The compute command and arguments.
- Stack-specific environment variables.
- Stack-specific volumes and mounts.
- Hypervisor device resources.
- Stack-specific memory-overhead calculation and resource adjustment.
- Stack-specific resource claims or devices.
- Hypervisor capability node selectors.
- CPU model, CPU feature, machine type, firmware, and confidential-computing
  scheduling constraints.
- Stack-specific affinity.
- Stack-specific annotations, including a memory-overhead annotation when
  configured.

Plugin implementation dependencies, such as launcher image paths, runtime
directories, timeout values, hypervisor resource implementations, and memory
overhead calculators, belong to the plugin and are provided when the plugin is
constructed. They are not request fields.

The plugin must be self-contained. It must not call `TemplateService` or access
`ClusterConfig`, Kubernetes clients, informer stores, or controller caches.

## Plugin output

The logical output is:

| Field | Type | Description |
|---|---|---|
| `pod` | `Pod` | The completed virt-launcher Pod. |

For the in-tree API, the renderer mutates `request.Pod` and returns an error.
For a future gRPC API, the response should contain the completed Pod:

```proto
message RenderLauncherManifestResponse {
  k8s.io.api.core.v1.Pod pod = 1;
}
```

Errors must be returned explicitly. A plugin must not return a success-shaped
fallback Pod when it cannot render a valid manifest.

## Future protobuf and gRPC shape

The protobuf API should preserve the same logical boundary:

```proto
service LauncherManifestRenderer {
  rpc RenderLauncherManifest(RenderLauncherManifestRequest)
      returns (RenderLauncherManifestResponse);
}

message RenderLauncherManifestRequest {
  kubevirt.io.api.core.v1.VirtualMachineInstance vmi = 1;
  k8s.io.api.core.v1.Pod pod = 2;
  kubevirt.io.api.core.v1.KubeVirtConfiguration configuration = 3;
  LauncherRenderMode mode = 4;
}

message RenderLauncherManifestResponse {
  k8s.io.api.core.v1.Pod pod = 1;
}

enum LauncherRenderMode {
  LAUNCHER_RENDER_MODE_UNSPECIFIED = 0;
  LAUNCHER_RENDER_MODE_LAUNCH = 1;
  LAUNCHER_RENDER_MODE_MIGRATION_TARGET = 2;
  LAUNCHER_RENDER_MODE_PROVISIONING = 3;
}
```

The exact protobuf package and imported Kubernetes/KubeVirt message types will
be decided when the gRPC transport is introduced. The request and response
semantics should remain identical to the in-tree interface.

## Validation

Virt-controller treats plugin output as untrusted until it has been validated.
Validation occurs after the plugin returns and after generic post-render policy
has been applied.

Post-render validation must include:

- Verifying that the Pod and compute container are structurally complete.
- Verifying required common base-Pod fields were not removed or invalidated.
- Validating permitted host devices against the effective cluster policy.
- Validating resource and resource-claim consistency.
- Rejecting unsupported modes, missing output, malformed fields, and invalid
  stack-specific mutations.

Validation failures abort rendering and prevent Pod creation. Validation is
owned by virt-controller rather than individual plugins so all virtualization
stacks are subject to the same safety and policy checks.

## Ownership summary

| Stage | Owner | Responsibility |
|---|---|---|
| Pre-render resolution | Virt-controller | Resolve clients, stores, PVCs, image IDs, sidecars, namespace policy, and other controller-local state. |
| Base Pod construction | Virt-controller | Build common KubeVirt containers, volumes, resources, security, metadata, and scheduling. |
| Stack completion | Virtualization plugin | Add image, command, runtime, hypervisor resources, overhead, and stack capability scheduling. |
| Generic post-render policy | Virt-controller | Apply controller-owned policy that must be identical for every stack. |
| Final validation | Virt-controller | Validate structure, policy, devices, resources, and preservation of the base contract. |

