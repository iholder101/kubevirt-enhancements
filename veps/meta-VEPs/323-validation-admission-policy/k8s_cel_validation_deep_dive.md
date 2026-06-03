# Deep Dive: Understanding CRD-Level CEL Validation (`x-kubernetes-validations`) Without ValidatingAdmissionPolicy

This document clarifies the structural and architectural distinctions between Custom Resource Definition (CRD) schema validation powered by the Common Expression Language (CEL) and cluster-wide validation via `ValidatingAdmissionPolicy` (VAP). It maps how tools like Kubebuilder integrate these rules natively into the Kubernetes API request pipeline.

---

## 1. The Kubernetes Request Lifecycle: Schema vs. Admission

The confusion surrounding this topic usually stems from the fact that Kubernetes leverages the Common Expression Language (CEL) across multiple distinct subsystems. While `x-kubernetes-validations` (CRD-level validation) and `ValidatingAdmissionPolicy` (VAP) both execute CEL expressions, they reside in completely separate processing domains within the Kubernetes API server request pipeline. One does not rely on the other.

When a user submits a resource mutation (such as a `POST`, `PUT`, or `PATCH` request via `kubectl apply`), the payload transits through specific, ordered stages:

```
[ Incoming Request ]
         │
         ▼
 1. Authentication & Authorization
         │
         ▼
 2. Mutating Admission (Webhooks)
         │
         ▼
 3. Schema Validation & Pruning ──► (Where x-kubernetes-validations/CEL executes)
         │
         ▼
 4. Validating Admission ─────────► (Where VAP and standard webhooks execute)
         │
         ▼
[ Etcd Storage ]
```

### 1.1 Schema Validation (Stage 3)
This stage is handled natively by the `apiextensions-apiserver` module. Before checking if an object conforms to cluster wide rules, the API server must ensure that the object's JSON/YAML structural syntax is legal. It parses the resource's OpenAPI v3 schema, executes unknown field pruning, and compiles and runs any embedded `x-kubernetes-validations` CEL rules. If a value breaks a rule here, the request is dropped immediately before reaching the general admission subsystem.

### 1.2 Validating Admission (Stage 4)
If the resource passes schema validation, it enters the generic admission loop. This is a decoupled phase designed for cluster administrators to apply broad, cross-cutting rules or security guardrails across multiple resource types using `ValidatingAdmissionPolicy` or external validation webhooks.

---

## 2. How Kubebuilder Integrates CEL Natively

When building operators with Kubebuilder, you act as an API author modeling a specialized, self-contained domain object. Kubebuilder acts as a compiler that bridges Go structure code and the OpenAPI v3 specifications native to the Kubernetes API server. 

The pipeline from a simple code comment to native server execution operates as follows:

### Step A: Declaring the Kubebuilder Marker
In your Go definitions (e.g., `api/v1/mykind_types.go`), you write standard Go structs but decorate target fields with specialized `XValidation` tags:

```go
type DatabaseSpec struct {
    // +kubebuilder:validation:XValidation:rule="self.minReplicas <= self.replicas",message="replicas cannot be less than minReplicas"
    Replicas    int32 `json:"replicas"`
    MinReplicas int32 `json:"minReplicas"`
}
```

### Step B: Compilation to YAML via `controller-gen`
When running `make manifests`, Kubebuilder invokes the `controller-gen` configuration tool. This utility extracts your inline Go string rules and translates them directly into a standard OpenAPI v3 extension block (`x-kubernetes-validations`) in your generated CRD manifest:

```yaml
# config/crd/bases/mygroup.domain_mykinds.yaml
properties:
  replicas:
    type: integer
  minReplicas:
    type: integer
x-kubernetes-validations:
  - message: replicas cannot be less than minReplicas
    rule: self.minReplicas <= self.replicas
```

### Step C: Native API Server Execution
When this CRD manifest is applied to a cluster, the `apiextensions.k8s.io` engine parses the schema and buffers the compiled CEL rule directly into the API server's local type-cache. 

There are **no intermediate or external policy resources created**, nor does it touch the admission registration components. When a client submits a payload, the API server automatically projects the object fields into the CEL runtime context, mapping the current state to the `self` variable and the prior persisted state to `oldSelf`.

---

## 3. Structural Comparison: CRD-Level Validation vs. VAP

The two CEL implementations are designed for entirely distinct personas, architectural scopes, and operational blast radiuses:

| Architectural Metric | CRD Validation (`x-kubernetes-validations`) | Validating Admission Policy (VAP) |
| :--- | :--- | :--- |
| **API Group Location** | `apiextensions.k8s.io` (Built straight into the CRD) | `admissionregistration.k8s.io` (Decoupled configuration asset) |
| **Target Persona** | **Application / Operator Developers** defining immutable validation boundaries for their apps. | **Cluster Administrators / Platform Teams** enforcing cluster-wide security baselines or guardrails. |
| **Blast Radius** | Strictly localized to the specific Custom Resource type where it is declared. | Multi-resource capable; can simultaneously intercept Pods, Services, Deployments, or any CRD. |
| **CEL Context Variables** | Restricted to the data fields under evaluation: `self` and `oldSelf`. | Broader context access: `object`, `oldObject`, `request`, `namespace`, and policy `params`. |
| **Distribution Method** | Shipped transparently inside the CRD bundle. Installing the operator auto-enforces the constraints. | Created separately as twin infrastructure configurations: a policy object and a matching binding object. |

---

## Summary

Kubebuilder uses code-generation markers to weave CEL rules directly into the core data type contract of your custom kinds. The Kubernetes API server parses this and executes the validation natively as part of its baseline type-checking schema step. As a result, you can enforce rich, multi-field rules completely independent of `ValidatingAdmissionPolicy` configurations.
