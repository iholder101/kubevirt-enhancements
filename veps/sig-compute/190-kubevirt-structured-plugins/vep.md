# VEP #190: Kubevirt Structured Plugins

Owners:
- @iholder101
- @vladikr

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue created, which links to VEP dir in [kubevirt/enhancements] (not the initial VEP PR)
- [ ] (R) Target version is explicitly mentioned and approved
- [ ] (R) Graduation criteria filled

## Table of contents

- [Release Signoff Checklist](#release-signoff-checklist)
- [Overview](#overview)
- [Motivation](#motivation)
- [Goals](#goals)
- [Non Goals](#non-goals)
- [Definition of Users](#definition-of-users)
- [User Stories](#user-stories)
- [Repos](#repos)
- [Design](#design)
  - [The Domain Hook](#the-domain-hook)
    - [Domain Hook modes](#domain-hook-modes)
    - [Domain Hook API exmaples](#domain-hook-api-exmaples)
  - [The Node Hook](#the-node-hook)
    - [Node hook points](#node-hook-points)
    - [Node hook API examples](#node-hook-api-examples)
  - [Mutation Policies and Webhooks](#mutation-policies-and-webhooks)
    - [Mutating policy / webhook API examples](#mutating-policy--webhook-api-examples)
  - [Plugin metadata](#plugin-metadata)
    - [Basic fields](#basic-fields)
    - [Versioning and Upgrade Path](#versioning-and-upgrade-path)
    - [multi-plugin support](#multi-plugin-support)
  - [Benefits to this design](#benefits-to-this-design)
- [API Examples (full version)](#api-examples-full-version)
- [Alternatives](#alternatives)
- [Scalability](#scalability)
- [Update/Rollback Compatibility](#updaterollback-compatibility)
- [Functional Testing Approach](#functional-testing-approach)
- [Implementation History](#implementation-history)
- [Graduation Requirements](#graduation-requirements)
  - [Alpha](#alpha)
  - [Beta](#beta)
  - [GA](#ga)

## Overview

KubeVirt currently allows for virtual machine (VM) customization through a sidecar model that modifies raw libvirt XML.
While functional, this approach is very limited (see below), and untrivial to create complex features with.

This proposal is building on the philosophy of frameworks like NRI (Node Resource Interface).
It introduces a structured hooking-based plugin mechanism to replace the existing "sidecar hook" model.
The new mechanism will enable a safer, holistic, maintainable and powerful way of integrating out-of-tree functionality
into KubeVirt.

Not only will this plugin mechanism be able to modify the domain XML in a more structured way, but this proposal adds
node-level hooks to perform operations by virt-handler, allowing privileged modifications to resources like
housekeeping CPUs, PRlimit, or node sockets.

Additionally, we propose integrating Kubernetes' Mutating Admission Policy to extend customization to pod and vmi
specifications, enhancing the flexibility of virtual machine deployments.

## Motivation

Often, KubeVirt features must be implemented through external extensions rather than in-tree changes.
See user stories below for detailed cases where this is needed.

The current hooking mechanism in KubeVirt allows sidecars to directly modify libvirt XML, which poses several
challenges:
- **Validation Bypass**: Modifications are not validated against KubeVirt's schema, risking invalid configurations.
- **Version Incompatibility**: XML changes may break with updates to KubeVirt or libvirt.
- **Lack of Auditing**: Changes are not tracked, making debugging difficult.
- **Resource and Migrability Ambiguity**: Hooks do not declare resource needs or impacts on VM migratability.
- **Limited scope**: Only modifies XML, with no ability to perform node-level operations, control other pod aspects
  (e.g. add a volume), etc.

This proposal addresses these issues by introducing a `Plugin` Custom Resource Definition (CRD) that allows:
- Structured modifications to the `DomainSpec` in a safe way.
- Node-level privileged operations to run via virt-handler.
- Integrating Kubernetes' Mutating Admission Policy, we further enable pod-level and vmi-level customizations,
  providing a comprehensive solution for VM and pod configuration.

## Goals

- Introduce a `Plugin` CRD that centralizes all plugin-related configuration and components.
- Support a structured way of modifying the `DomainSpec`.
- Support two hook modes: simple JSON/CEL-based and advanced plugin-based.
- Ensure hooks are validated, version-compatible, and auditable via Kubernetes events.
- Support modifying pod attributes like resources, migratability, etc.
- Support Mutating Admission Policy to allow vmi or pod level modifications.
- Support node-level privileged operations in virt-handler.
- Provide a way of versioning the plugin and provide KubeVirt version contraints.
- Provide a way to plan an upgrade path in advance.
- Support multi-plugins: dependencies, conflict detection, etc.

## Non Goals

- Allowing arbitrary, unvalidated XML modifications.
- Provide a helper library/framework for simplifying Plugin development (this should be addressed in a follow-up VEP).

## Definition of Users

- Enterprise: A company that develops KubeVirt and uses it in production. Prioritizes customer demands and business needs.
- KubeVirt Developer: aims to develop features for KubeVirt.
- Cluster administrator: in charge of managing clusters in production.

## User Stories

- As an enterprise, it prioritizes rapid delivery over backward-compatibility, community support and architectural
  stability, but is not interested in forking KubeVirt due to the huge maintenance overhead.
  It wishes to deliver fast by developing a Plugin.
- As an enterprise, it wants to design a feature for an advanced customer that wants to test it quickly and provide
  feedback.
  With a quick feedback loop, the company is able to make quick changes.
  After the customer is satisfied, the company can file a mature VEP with a POC and feedback from real users.
- As a developer of a very complex feature, I understand that a VEP would be challenging to review and comprehend.
  To help the community understand my intentions, I wish to build a high-quality POC that demonstrates a limited-scoped
  working feature to serve as a base ground for an enhancement proposal discussion.
- As a feature developer, I want to be able to develop a feature that the core 
KubeVirt community does not aim to maintain for at least one of these reasons:
  - A niche specialized behavior is needed that does not make sense upstream.
  - Too complex / dangerous / for most use-cases.
  - No expertise in this subject among the community members (no one to maintain it).

## Repos

kubevirt/kubevirt

## Design

The architecture of the new Plugin mechanism will consist of these different components:
**domain hook**, **node hook**, **mutating policies** and **plugin metadata**.

Let's describe them one by one,
then explain the overall CRD structure that centralizes all plugin-related configuration and components.

### The Domain Hook

The proposed mechanism focuses on modifying the `DomainSpec`, a Go struct generated from the VirtualMachineInstance
(VMI) spec, which serves as the internal representation of a VM's desired configuration before conversion to libvirt XML.
Hooks operate as sidecar containers within the `virt-launcher` pod, executing after the initial `DomainSpec` generation
but before the final XML is created.
KubeVirt's `virt-launcher` acts as a gatekeeper, validating and applying hook-proposed changes to ensure safety and
compatibility.

The current KubeVirt `DomainSpec` struct only includes a subset of the fields available in the full libvirt `DomainSpec`.
To allow the plugin complete control over the generated XML (similar to the legacy Sidecar Hook),
it will be replaced it with the complete, fully-expanded libvirt DomainSpec struct.

#### Domain Hook modes

The domain hook would support two types of hooks:

**Simple hooks**: These are JSON or CEL based hooks.
These hooks are valuable for simple XML changes that do not require complex logic.
These hooks are easily deployable, eliminating the need to write code, provide a container image,
running another sidecar container, etc.

**Advanced (plugin-based) hooks**: In this mode, a sidecar container would run inside `virt-launcher`.
This sidecar container would listen to a defined socket, waiting for requests by virt-launcher.
In pre-defined hook points (e.g. before the domain XML is applied) virt-launcher would send a request to the socket,
providing the `DomainSpec` struct generated by virt-launcher alongside a `DomainHookContext` struct that will contain
extra information.

Currently, we think of a single hook point, right after the domain XML is generated but before it's handed to libvirt.
Moving forward we can support more hook points if needed.

#### Domain Hook API exmaples

<ins>Simple Hooks</ins>:

JsonPatch based simple hook:
```yaml
domainHooks:
  - type: JsonPatch
    operations:
      - op: add
        path: /spec/devices/disks
        value:
          name: monitoring-disk
          disk:
            bus: virtio
```

CEL based simple hook:
```yaml
domainHooks:
  - type: ApplyConfiguration
    expression: |
      object.spec.devices.disks + [
        {
          "name": "monitoring-disk",
          "disk": {
            "bus": "virtio"
          }
        }
      ]
```

<ins>Advanced Hooks</ins>:

```yaml
domainHooks:
  - type: Plugin
    socketPath: "/var/run/kubevirt-plugin/my-plugin-socket.sock"
```

Note: The socket would need to reside under `/var/run/kubevirt-plugin`,
which will be defined as a shared volume with the compute container.

### The Node Hook

To support node-level customizations, hooks can also define operations that run in virt-handler on the node.
These are invoked via gRPC (or, ttRPC, that’s used by NRI) calls from virt-handler to a plugin server
(a DaemonSet running on each node). 
node hooks run in virt-handler on the host node, allowing operations that affect node resources.
These hooks are invoked at specific points in virt-handler's controllers.
Node hooks cannot modify `DomainSpec` or XML directly but can influence runtime behavior.
Plugins can implement both domain and node hooks for cohesive functionality.

Similarly to the domain hooks, `virt-handler` will communicate with the relevant socket at pre-defined hook points.
It would provide a `NodeHookContext` struct, and ask for a function to run by virt-handler.

#### Node hook points

- **PreVMStart**: Before VM launch (e.g., setup node devices/network; aligns with vmUpdateHelperDefault).
- **PostVMStart**: After VM is running (e.g., verify node resources).
- **PreMigrationSource**: Before migration from source node (e.g., prepare sockets; aligns with migrateVMI in migration-source.go).
- **PostMigrationSource**: After migration completes on source (e.g., cleanup; aligns with handleSourceMigrationProxy).
- **PreMigrationTarget**: Before migration to target node (e.g., setup target sockets; aligns with prepareMigrationTarget in migration-target.go).
- **PostMigrationTarget**: After migration completes on target (e.g., repair/verify; aligns with finalizeMigration).
- **OnVMStop**: When VM stops (e.g., cleanup node resources; aligns with processVmUpdate on shutdown).
- **OnVMStop**: Called periodically.

The plugin would have to explicitly mention which hook points it wishes to register for.

#### Node hook API examples

```yaml
nodeHooks:
  - mode: Plugin
    socket: /var/run/my-node-socket.sock
    permittedHooks:
    - PreStartVm
    - PostMigrationSource
selector:
  matchLabels:
    use-plugin: my-plugin
  matchFields:
    - key: metadata.name
      operator: Prefix
      value: vm-
  conditions: # CEL-based
    - "vmi.status.phase == 'Running'"
    - "vmi.status.conditions.exists(c, c.type == 'Ready' && c.status == 'True')"
    - "vmi.status.conditions.exists(c, c.type == 'LiveMigratable' && c.status == 'True')"
```

### Mutation Policies and Webhooks

In order to provide pod or VMI level modifications,
the new plugin mechanism would be able to deploy and manage Kubernetes
[Mutating Admission Policies](https://kubernetes.io/docs/reference/access-authn-authz/mutating-admission-policy/).
By utilizing these mutation policies, we would support providing a declarative policy to modify VMIs and pods
which would be performed on the API server's side.

When Mutating Admission Policies are used, the plugin mechanism will be responsible for deploying, managing and
garbage-collecting the specified policies.

Since mutating policies are so easy to manage, deploy and understand - it would be the encouraged mechanism to perform
pod/vmi level changes.
However, since mutating policies are purely CEL-based, they are more limited than mutating webhooks.
Therefore, in complex scenarios, a mutating webhook would also be supported but in a limited way -
instead of fully managing and doploying them (like with mutating policies), the plugin will only verify that the webhooks
exist and ready.

#### Mutating policy / webhook API examples:

<ins>Mutating policies</ins>:

The below examples are using CEL's `ApplyConfiguration`, but `JSONPatch` type should also be supported.

```yaml
podHooks:
  MutatingAdmissionPolicies:
    - matchConditions:
        - expression: "!object.spec.volumes.exists(v, v.name == 'mesh-config')"
      mutations:
        - patchType: ApplyConfiguration
          applyConfiguration:
            expression: >
              Object{
                spec: Object.spec{
                  volumes: Object.spec.volumes + [
                    {
                      name: "mesh-config",
                      image: {
                        image: "mesh/config-bundle:v1",
                        pullPolicy: "IfNotPresent"
                      }
                    }
                  ]
                }
              }

vmiHooks:
  MutatingAdmissionPolicies:
    - matchConditions:
        - expression: "object.spec.domain.features.hyperv.tlbflush == null"
      mutations:
        - patchType: ApplyConfiguration
          applyConfiguration:
            expression: >
              Object{
                spec: Object.spec{
                  domain: Object.spec.domain{
                    features: Object.spec.domain.features{
                      hyperv: Object.spec.domain.features.hyperv + {
                        tlbflush: { enabled: true },
                        vpindex:  { enabled: true }
                      }
                    }
                  }
                }
              }
```

<ins>Mutating webhooks</ins>:

Mutating webhooks are also supported, but will not be deployable and managable by the Plugin mechanism.
Instead, it will only validate that these webhooks exist and ready on the cluster.

```yaml
podHooks:
  MutatingAdmissionWebhooks:
    - name: "my-webhook-1"
    - name: "my-webhook-2"
```

### Plugin metadata

Each plugin would have to provide some basic metadata about itself.

The main idea here goes beyond specifying a unique name and identify to the plugin,
but also provide an upgrade path, versioning, multi-plugin support, dependency between plugins and more.

#### Basic fields

These are the basic fields every plugin would need to populate:
- name:  a unique identifier for the hook.
- Version: Specifies the hook’s version.

#### Versioning and Upgrade Path

With the following fields the plugin system will enable to properly define version constraints and a planned upgrade path:
- kubeVirtVersionCondition: a CEL based condition constraint for KubeVirt's version.
- libvirtVersionCondition: a CEL based condition constraint for Libvirt's version.
- upgradePaths: This field will be a list of values mapping a KubeVirt version to an image tag.
  - For example: `KubeVirtVersion: 1.7 -> pluginImageTag: 1.7, KubeVirtVersion: 1.8 -> pluginImageTag: 1.8`, etc.
  - In the above example, picture the v1.7 image existing and used in production, while the 1.8 image does not yet exist.
However, when v1.8 is realsed, an image with a 1.8 tag would be added to a remote registry, allowing the Plugin owner
To plan an upgrade path in advance.

In order to update the Plugin itself, a new Plugin CR instance can be created on the cluster with different version conditions.

#### multi-plugin support

The plugin system architecture aims to allow multiple plugins running on the same cluster.
In addition, plugins could provide dependencies between one another.
The plugin system should also validate that plugins do not override each other's values.

The following fields will be defined for this purpose:
- dependsOn: a list of plugins that have to run before this plugin.
- preservePaths: paths to exempt from modifications (NRI-inspired for conflict avoidance).

In addition, behind the scenes, modifications to the `DomainSpec` would be converted to a list of JSON patches.
This would allow multiple plugins to work on separate set of fields and allow separation of concerns.
In addition, this will allow validating that there is no conflict between different plugins so that they don't
override each other's data.

Only if a plugin is dependent on another plugin, both can edit the same fields, because now a clear ordering
is defined between them. This way one plugin can do some work that another plugin would finalize.

### Benefits to this design

- **Enhanced Flexibility**: Administrators can customize both the VM’s configuration and the pod’s specification, enabling use cases like monitoring sidecars or volume injection.
- **Centralized Policy Management**: Mutating Admission Policies enforce cluster-wide consistency for pod configurations.
- **Streamlined Experience**: Compared to custom Mutating webhooks, integration with VirtualMachineHook simplifies configuration.
- **Security Enhancements**: Policies can enforce security contexts or resource limits on pods.

Security and Testing:
- **RBAC**: Restrict VirtualMachineHook and Mutating Admission Policy creation to cluster administrators.
- **Validation**: Ensure strict validation of hooks and policies to prevent invalid configurations.
- **Testing** Plan: Include unit, integration, and end-to-end tests to verify DomainSpec and pod modifications.
- **Auditing**: Log all modifications via Kubernetes events for traceability.

## API Examples (full version)

```yaml
apiVersion: kubevirt.io/v1
kind: Plugin
metadata:
  name: monitoring-hook
spec:
  PluginMetadata:
    name: "best-plugin-ever"
    version: "0.0.1"
    kubeVirtVersionCondition: "version > 1.5"
    libvirtVersionCondition: "version > 9.0.0"
    upgradePaths:
      - KubeVirtVersion: 1.7
        pluginImageTag: 1.7
      - KubeVirtVersion: 1.8
        pluginImageTag: 1.8
    dependsOn:
      - "another-cool-plugin"
    preservePaths: "domainspec.devices.disks"
    
  domainHooks:
  - type: JsonPatch
    operations:
      - op: add
        path: /spec/devices/disks
        value:
          name: monitoring-disk
          disk:
            bus: virtio
  - type: Plugin
    socketPath: "/var/run/kubevirt-plugin/my-plugin-socket.sock"

  podHooks:
    MutatingAdmissionPolicies:
      - matchConditions:
          - expression: "!object.spec.volumes.exists(v, v.name == 'mesh-config')"
        mutations:
          - patchType: ApplyConfiguration
            applyConfiguration:
              expression: >
                Object{
                  spec: Object.spec{
                    volumes: Object.spec.volumes + [
                      {
                        name: "mesh-config",
                        image: {
                          image: "mesh/config-bundle:v1",
                          pullPolicy: "IfNotPresent"
                        }
                      }
                    ]
                  }
                }
    MutatingAdmissionWebhooks:
      - name: "my-webhook-1"
  
  vmiHooks:
    MutatingAdmissionPolicies:
      - matchConditions:
          - expression: "object.spec.domain.features.hyperv.tlbflush == null"
        mutations:
          - patchType: ApplyConfiguration
            applyConfiguration:
              expression: >
                Object{
                  spec: Object.spec{
                    domain: Object.spec.domain{
                      features: Object.spec.domain.features{
                        hyperv: Object.spec.domain.features.hyperv + {
                          tlbflush: { enabled: true },
                          vpindex:  { enabled: true }
                        }
                      }
                    }
                  }
                }

  nodeHooks:
    - mode: Plugin
      socket: /var/run/my-node-socket.sock
      permittedHooks:
      - PreStartVm
      - OnVMStop
      - Reconcile
  selector:
    matchLabels:
      use-plugin: my-plugin
    matchFields:
      - key: metadata.name
        operator: Prefix
        value: vm-
    conditions: # CEL-based
      - "vmi.status.phase == 'Running'"
      - "vmi.status.conditions.exists(c, c.type == 'Ready' && c.status == 'True')"
      - "vmi.status.conditions.exists(c, c.type == 'LiveMigratable' && c.status == 'True')"
```

## Alternatives

One of the obvious alternatives to this approach is forking KubeVirt in order to provide out-of-tree functionality.

While a fork is possible, the maintenance price of using it is huge,
mainly because the core KubeVirt is not aware of the made changes,
hence the added features can go through rough rebase conflicts, which is bad,
or be completely broken by new KubeVirt logic, which is worse.

## Scalability

From the Plugin mechanism perspective we don't envision any scalability concerns.

However, obviously the plugin authors would have the responsibility to keep their implementation scalable.

## Update/Rollback Compatibility

See `Versioning and Upgrade Path` section above.

## Functional Testing Approach

<!--
An overview on the approaches used to functional test this design)
-->

## Implementation History

<!--
For example:
01-02-1921: Implemented mechanism for doing great stuff. PR: <LINK>.
03-04-1922: Added support for doing even greater stuff. PR: <LINK>.
-->

## Graduation Requirements

<!--
The requirements for graduating to each stage.
Example:
### Alpha
- [ ] Feature gate guards all code changes
- [ ] Initial implementation supporting only X and Y use-cases

### Beta
- [ ] Implementation supports all X use-cases

It is not necessary to have all the requirements for all stages in the initial VEP.
They can be added later as the feature progresses, and there is more clarity towards its future.

Refer to https://github.com/kubevirt/community/blob/main/design-proposals/feature-lifecycle.md#releases for more details
-->

### Alpha

### Beta

### GA
