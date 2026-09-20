# 135 - Dedicated ServiceAccount for KafkaProxy pods

Add an optional `KafkaProxy.spec.infrastructure.proxyContainer.serviceAccountName` field so users
can define a dedicated, user-managed Kubernetes ServiceAccount for KafkaProxy pods. The proposal
also adds optional `KafkaProxy.spec.infrastructure.deployment.strategy` controls.

## Current situation

Each `KafkaProxy` pod currently uses the `default` ServiceAccount in the proxy namespace.

## Non-goals

This proposal does not add:

- Operator creation, ownership, mutation, annotation management, or deletion of proxy
  ServiceAccounts.
- Operator management of Roles, ClusterRoles, RoleBindings, or ClusterRoleBindings for proxy
  workloads.
- Cross-namespace ServiceAccount references.
- Changes to automatic ServiceAccount token mounting for proxy pods.
- ServiceAccount watches or an existence preflight.

## Motivation

Users may need KafkaProxy pods to use a distinct Kubernetes identity when proxy configuration or an
external workload identity integration requires permissions or annotations that must not be shared
with every pod using the namespace's default ServiceAccount.

The feature also allows users to apply least-privilege RBAC. The operator should not need to own or
mutate those identity resources to provide this capability.

## Proposal

Add the optional ServiceAccount field to `KafkaProxy.spec`:

```yaml
apiVersion: kroxylicious.io/v1alpha1
kind: KafkaProxy
metadata:
  name: simple
  namespace: my-proxy
spec:
  infrastructure:
    proxyContainer:
      serviceAccountName: kroxylicious-proxy
```

The user creates and manages the referenced ServiceAccount separately:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kroxylicious-proxy
  namespace: my-proxy
```

### API semantics

- `serviceAccountName` is optional and is available on `KafkaProxy` under
  `infrastructure.proxyContainer`; no other CRD is changed.
- The value is a ServiceAccount `metadata.name`, not a `namespace/name` reference.
- Kubernetes resolves the name in the namespace of the `KafkaProxy` and its generated pod.
- The value is validated as a DNS-1123 subdomain with a maximum length of 253 characters.
- When set, the operator copies the value to the generated pod template. When omitted, the
  generated pod template leaves the field unset and Kubernetes uses the namespace's `default`
  ServiceAccount.
- The operator does not silently fall back to `default` when a configured account is unavailable.

The placement follows the existing infrastructure API shape, while the user-managed lifecycle
follows the Prometheus Operator's optional `spec.serviceAccountName` pattern.

### Ownership, permissions, and security boundary

Users manage the ServiceAccount, including its annotations, RBAC, token settings, and cloud
associations. The operator only copies its name to the generated Deployment; it does not manage,
read, or watch the account.

### Missing accounts and lifecycle

If the configured ServiceAccount is missing, replacement pods fail admission and Kubernetes does not
fall back to `default`. The generated Deployment reports `ReplicaFailure=True` with reason
`FailedCreate`, and its ReplicaSet event includes the missing account name.

The resulting `KafkaProxy` status includes:

```yaml
status:
  conditions:
    - type: Ready
      status: "False"
      reason: FailedCreate
      message: 'Error creating: ... serviceaccount "missing-proxy" not found'
```

The operator sets `KafkaProxy.status.conditions[Ready]` to `False` when the generated Deployment
reports `ReplicaFailure=True`, copying the Kubernetes reason and message. For a missing
ServiceAccount, KafkaProxy status is the primary diagnostic surface. This uses the operator's
existing Deployment observation and requires no ServiceAccount permissions. The condition returns to
`True` after the account is created and the rollout recovers.

To change accounts safely, create and configure the new account, update the `KafkaProxy`, wait for
rollout completion, then remove the old account.

### Rollout behavior

A ServiceAccount change updates the Deployment pod template. If the selected account is missing,
replacement pods cannot be created. Kubernetes' default rolling update can remove healthy old pods
before that failure settles, leaving fewer ready proxies. This proposal lets users choose the
availability and capacity trade-off through the generated Deployment strategy.

For example:

```yaml
spec:
  infrastructure:
    deployment:
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxUnavailable: 0
          maxSurge: 1
```

If the strategy is omitted, Kubernetes Deployment defaults apply.

### Validation evidence

Kind integration validation on Kubernetes v1.31.0 and v1.36.1 confirmed that a missing account
produces `ReplicaFailure=True`/`FailedCreate` and an event stating that the referenced
ServiceAccount was not found, that the operator surfaces the failure as `KafkaProxy Ready=False`, and
that creating the account restores `Ready=True` and completes the rollout. The v1.31.0 five-replica
experiment also showed the default 25%/25% policy retaining 4/5 old pods while
`maxUnavailable: 0`/`maxSurge: 1` retained all 5. Focused operator tests covered configuring,
changing, removing, and recovering accounts, plus CRD validation of valid and invalid names.

## Affected/not affected projects

**Affected:**

- `kroxylicious-kubernetes/kroxylicious-kubernetes-api` — add the optional CRD field and validation.
- `kroxylicious-kubernetes/kroxylicious-operator` — copy the field to generated proxy Deployments,
  apply the optional rollout settings, set `KafkaProxy Ready=False` from Deployment
  `ReplicaFailure`, and add tests.
- `kroxylicious-docs` — document creation, configuration, security, lifecycle, ServiceAccount
  changes, and troubleshooting.

**Not affected:**

- The operator's own ServiceAccount, RBAC, and admission webhook.
- Non-Kubernetes deployments and existing proxy/filter behavior.

## Compatibility

The API fields are additive and optional. Existing `KafkaProxy` resources that omit them retain the
current default ServiceAccount and Kubernetes Deployment rollout behavior. Removing the account
field removes the explicit pod-template value on the next reconciliation and returns to Kubernetes
default selection.

Existing user-managed ServiceAccounts are not adopted, owner-referenced, or modified. No additional
operator ServiceAccount permissions are required for the reference itself.

When configured, the rollout strategy applies to all proxy updates. Omitting it preserves Kubernetes
defaults.

## Rejected alternatives

### Operator-created ServiceAccounts

Rejected because the operator would need ownership, deletion, annotation, conflict, and RBAC
semantics. Those permissions are unnecessary for selecting a user-managed identity and would make
provider-specific identity configuration operator-owned.

### A creation or RBAC-management boolean

Rejected because a name already provides an explicit opt-in. A second API would introduce larger
lifecycle and security semantics outside this feature.

### No ServiceAccount preflight or watch

Not included because it would require additional RBAC and would still be subject to races before pod
admission. The generated Deployment already exposes admission failures, which the operator uses to
set the `KafkaProxy` Ready condition to `False`.

### A namespace in the reference

Rejected because `PodSpec.serviceAccountName` is a name resolved in the pod namespace. Accepting a
namespace would imply cross-namespace pod ServiceAccount semantics that Kubernetes does not provide.

## Verification criteria

The implementation is complete when tests and documentation demonstrate that:

- ServiceAccount settings render the expected pod template;
- omitted rollout settings leave Kubernetes Deployment defaults in effect, while configured strategy
  and rolling-update values render on the generated Deployment;
- valid DNS-1123 names are accepted and empty or invalid names are rejected by the CRD;
- changing the name replaces proxy pods through a normal Deployment rollout;
- a missing account never falls back to `default`, appears as Deployment `ReplicaFailure`, and is
  surfaced as `KafkaProxy Ready=False` with recovery after the account is restored;
- a failed replacement rollout retains existing ready proxy pods and recovers after the account is
  restored when the configured strategy sets `maxUnavailable: 0`;
- the operator does not create, mutate, adopt, bind, delete, read, or watch the referenced account.

This proposal addresses [issue #3758](https://github.com/kroxylicious/kroxylicious/issues/3758), follows
the Prometheus Operator API pattern, and uses Kubernetes Deployment strategy semantics.
