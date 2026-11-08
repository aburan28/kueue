# KEP-NNNN: Quota-relevant PodSet equivalence and declared injected containers

<!-- Draft extending kubernetes-sigs/kueue#15570. Code references are against main @ 6b79acb (2026-11-07). -->

## Summary

Kueue decides whether a Job still matches its Workload by deep-equality on the pod
template (`pkg/util/equality/podset.go: ComparePodTemplate`). Every field of every
container is compared, including `image`, `env`, `args`, `volumeMounts` and probes,
none of which affect quota or flavor assignment.

Issue #15570 proposes relaxing this for LeaderWorkerSet updates, limited to tolerations and
init containers whose requests and ports don't change. This KEP generalizes that idea
in two ways:

1. **Quota-relevant equivalence.** An opt-in comparison mode, available to every
   integration, that compares only the parts of a PodSpec that affect quota,
   flavor assignment or node fit. Containers are matched by name, not by position.
2. **Declared injected containers.** A configuration-level allowlist of containers
   that admission webhooks inject after Kueue builds the Workload (service-mesh
   proxies, log/metrics agents, credential helpers). Each entry carries a declared
   resource envelope, and Kueue charges that envelope at Workload construction.
   Equivalence then tolerates the injected container as long as it stays within the
   envelope.

The motivating case is MultiKueue. Today, a worker-side webhook that adds a sidecar to
the Job template makes the remote Workload `OutOfSync`, and the manager re-dispatches
it in a loop. The only workaround is to inject the same sidecar on the manager as well.

## Motivation

### Where equality is enforced today

| Site | Code | Compares | On mismatch |
|---|---|---|---|
| Job ↔ Workload (all integrations) | `jobframework.EquivalentToWorkload` → `equality.ComparePodSetSlices` → `ComparePodTemplate` | `DeepEqual` on `InitContainers`, `Containers`, pod-level `Resources`; `ResourceClaims` keyed by name; `Tolerations` until admitted | Workload recreated (non-prebuilt) or finished `OutOfSync` (prebuilt) |
| Prebuilt Workload on a MultiKueue worker | `jobframework.ensurePrebuiltWorkloadInSync` | as above | `Finish(..., WorkloadFinishedReasonOutOfSync, ...)` |
| Manager ↔ remote Workload | `multikueue.isRemoteSpecOutOfSync` | `DeepEqual` on the whole `WorkloadSpec` (only `PreemptionGates` excluded) | remote deleted |
| Remote `OutOfSync` seen by manager | `multikueue` workload reconciler, step 3 | n/a | check reset to `Retry` and the workload re-dispatched |

`ComparePodTemplate` itself carries a TODO saying the comparison should be revisited to
cover what actually affects scheduling.

### The MultiKueue sidecar loop

1. The manager builds the Workload from the user's Job and dispatches it.
2. A worker reserves quota for the remote Workload, and only then does `SyncJob` create
   the Job on that worker.
3. A mutating webhook on the worker (Istio, a Datadog or Vector agent, a Kyverno policy)
   adds a container to the Job template.
4. The worker's Job reconciler finds that the template no longer deep-equals the
   prebuilt Workload and finishes it `OutOfSync`.
5. The manager sees a sync failure, sets the check to `Retry`, and re-dispatches.
   Go to step 2.

Because injection happens after quota is reserved (step 2), the worker cannot simply
re-derive the Workload from the mutated Job: the sidecar's resources would never be
charged. That is why this KEP declares sidecar overhead up front instead.

### Goals

- Offer an opt-in comparison mode that works across integrations and ignores fields
  that don't affect quota or placement.
- Let platform operators declare injected containers and a resource envelope for each,
  so that:
  - quota is charged correctly, on the manager and on workers;
  - webhook injection never causes `OutOfSync`, recreation or a MultiKueue re-dispatch;
  - an injected container that exceeds its envelope is still caught.
- Keep `Strict` as the default, with today's behavior bit-for-bit.
- Pin the comparison mode a Workload was admitted under, so that upgrading Kueue or
  changing configuration doesn't make admitted Workloads `OutOfSync`. This is the
  risk raised in #14395.
- Subsume #15570's LeaderWorkerSet relaxation.

### Non-Goals

- Heterogeneous pods within one PodSet. PodSets stay homogeneous; that direction is
  being handled in #14201 and KEP-6012 (CompositePodGroup).
- Comparing `nodeSelector` or `affinity`. They aren't compared today; adding them is a
  separate behavior change.
- Changing quota-relevant fields after admission. Increasing requests after quota is
  reserved stays invalid.
- Containers injected at the Pod level, after the Job controller creates pods. These
  never reach the template comparison. Their accounting is covered by the same
  declared envelope, but they need no equivalence change.

## Proposal

### API (Kueue Configuration, v1beta2)

```yaml
workloadEquivalence:
  # Strict (default): today's behavior.
  # QuotaRelevant: compare only the projection described below.
  mode: QuotaRelevant
  # Containers that may appear in a Job template without being in the Workload.
  # Only honored in QuotaRelevant mode.
  injectedContainers:
  - name: istio-proxy
    kind: InitSidecar          # InitSidecar (restartPolicy: Always) | Container
    maxRequests:               # declared envelope, charged per pod
      cpu: 200m
      memory: 256Mi
    namespaceSelector: {}      # optional: only where the webhook actually injects
  - name: vector-agent
    kind: Container
    maxRequests: { cpu: 100m, memory: 128Mi }
  # Optional per-integration override of mode.
  integrations:
  - framework: leaderworkerset.x-k8s.io/leaderworkerset
    mode: QuotaRelevant
```

- Feature gate: `QuotaRelevantWorkloadEquivalence`, alpha, default off.
- The MultiKueue manager and every worker must run the same `workloadEquivalence`
  configuration. Validation compares a hash of it (see MultiKueue below).

### Quota-relevant projection

For a `PodSpec`, the projection `Q(spec)` is:

- **Containers and init containers keyed by name, ignoring order.** For each container:
  - `resources.requests` and `resources.limits` (and `claims`);
  - for init containers, `restartPolicy`, because a native sidecar changes how
    effective requests are summed;
  - the set of `hostPort`s, because it affects node fit.
- **The effective pod request**, from `resourcehelpers.PodRequests`, as Kueue already
  uses in `pkg/resources/factory.go`. This protects the core invariant even if
  per-container fields move between containers.
- **Pod-level** `resources`, `overhead` and `runtimeClassName`. RuntimeClass overhead
  is already resolved in `workload.ResolveAdjustmentInputs`.
- **`resourceClaims`**, using the existing order-independent comparison.
- **`tolerations`**: same rule as today (compared until admitted).

Everything else is ignored. That includes `image`, `command`, `args`, `env`,
`envFrom`, `volumeMounts`, probes, `lifecycle`, `securityContext`, `workingDir`,
`imagePullPolicy` and termination-message fields.

`ComparePodSets` keeps its existing checks on `count`, `minCount` and the topology
request. Only `ComparePodTemplate` gains the new mode.

### Declared injected containers

**At Workload construction**, on the manager or on any cluster where Kueue builds the
Workload: for each PodSet whose namespace matches an entry's `namespaceSelector`, add
that entry's `maxRequests` to the PodSet's effective resources. This is a new step in
`applyAdjustmentsToPodSpec`, alongside the existing RuntimeClass-overhead and
LimitRange steps. The template is left unchanged, so the manager Job and the manager
Workload still match.

**At equivalence time**, if the Job template has a container that is not in the
Workload template:

1. If its name matches an `injectedContainers` entry of the same `kind`, and its
   requests fit within `maxRequests` (and, for `InitSidecar`, it has
   `restartPolicy: Always`), drop it from the Job side before projecting.
2. Otherwise the comparison fails, as today. The finish condition gets a precise
   reason, `UndeclaredInjectedContainer` or `InjectedContainerExceedsEnvelope`, with
   the container name. This builds on #13090's message work.

### MultiKueue

- `isRemoteSpecOutOfSync` uses the same projection when the gate is enabled. The remote
  `WorkloadSpec` is never rewritten from the worker's Job, so manager and remote specs
  stay identical. The relaxation only matters if some other path (for example a future
  in-place update) introduces drift.
- Because the envelope is charged at construction, manager and worker charge the same
  amount. No undercounting, and no oversized manager quota needed to absorb it.
- The step-3 sync-failure reset (`OutOfSync` → `Retry`) is unchanged. It just stops
  firing for declared sidecars. Add a counter,
  `kueue_multikueue_sync_failure_redispatch_total{reason}`, so loops caused by
  anything else are visible.
- On config drift: the manager stamps the remote Workload with the annotation
  `kueue.x-k8s.io/equivalence-config-hash`. If a worker's hash differs, the worker
  refuses to admit, with a clear event, rather than looping. Exact placement of this
  check is an open question.

### Pinning the mode at admission

When quota is reserved, record the comparison mode and the config hash in the
Workload's status (field name to be decided). `EquivalentToWorkload` always compares
using the pinned mode. This addresses #14395: changing the configuration, or a Kueue
upgrade that adds fields to `Strict`, cannot flip an already-admitted Workload to
`OutOfSync` and release its quota while pods are running.

### Relationship to #15570

Issue #15570 targets the LeaderWorkerSet validating webhook's immutability check on
updates. With this KEP, that webhook can call the same `Q()` projection when the
integration's mode is `QuotaRelevant`:

- Toleration-only changes stay allowed, as #15570 proposes.
- Init-container changes are allowed when name-keyed requests, `restartPolicy` and
  host ports are unchanged.
- The shape-hash `matchConditions` alternative proposed there becomes a cheap
  pre-filter computed from `Q()`.

### User Stories

#### Story 1: MultiKueue with a service mesh on workers

As a platform operator, my worker clusters run Istio and a log agent that inject
containers into Job templates. My manager cluster runs neither. Today every MultiKueue
Workload loops between `OutOfSync` and re-dispatch unless I replicate the mesh and agent
injection on the manager. With this KEP I declare `istio-proxy` and the log agent with
their resource envelopes once. Quota is charged correctly on every cluster, and
injection no longer breaks equivalence.

#### Story 2: Fleet-wide changes that don't affect quota

As a platform operator, I roll out a new image tag for a sidecar, a toleration, or an
env var across running LeaderWorkerSets and Jobs. In `QuotaRelevant` mode these changes
no longer force Workload recreation or a delete-and-recreate of the LWS, because they
don't change requests, ports or overhead.

### Notes/Constraints/Caveats

Open questions for reviewers:

1. Should `injectedContainers` live in Kueue Configuration (operator-owned) or in a
   cluster-scoped CRD that platform teams manage per cluster?
2. Should the envelope support a fraction of the main container's requests as well as
   fixed amounts?
3. Should `nodeSelector` and `affinity` join the projection now, behind the same gate,
   since the TODO in `ComparePodTemplate` already calls for it?

### Risks and Mitigations

- **Under-reserved quota (#14255).** An undeclared injected container, or one beyond
  its envelope, still fails the comparison. The envelope is charged before admission,
  so quota is never silently short.
- **Over-reserved quota.** The envelope is charged even where the webhook doesn't
  actually inject. Use `namespaceSelector` to scope it. A label-based selector could be
  added later if needed.
- **Hiding a real user change.** Changing only an image or env var no longer triggers
  recreation in `QuotaRelevant` mode. That is the intended behavior, and why the mode
  is opt-in. Document that template changes not affecting resources must be rolled out
  by the integration's own mechanism.
- **Manager and worker configured differently.** Covered by the config-hash check.

## Design Details

### Code changes

- **`pkg/util/equality/podset.go`**
  - Add `WithQuotaRelevant(injected []InjectedContainerSpec)`, a
    `ComparePodSetsOption`.
  - Add `projectPodSpec`, which builds a normalized, name-keyed projection, and
    compare projections with `Semantic.DeepEqual`.
- **`pkg/controller/jobframework/reconciler.go`**
  - `EquivalentToWorkload` appends the option based on the pinned mode, or the
    configured mode before admission.
  - `ensurePrebuiltWorkloadInSync` passes the precise mismatch reason through to
    `Finish`.
- **`pkg/workload/effective_resources.go`**
  - Add `InjectedContainerOverheads` to `AdjustmentInputs`, resolved from config and
    the namespace selector.
  - Apply it after RuntimeClass overhead and before the LimitRange defaults.
- **`pkg/controller/admissionchecks/multikueue/workload.go`**
  - `isRemoteSpecOutOfSync` takes options.
  - Add the config-hash check and the new metric.
- **LeaderWorkerSet webhook:** call `projectPodSpec` behind the per-integration mode.

### Test Plan

#### Unit tests

- **Unit tests** for `projectPodSpec`. Table cases:
  - reordered containers;
  - `image`, `env`, `args` or `volumeMounts` differences (equal);
  - a requests delta (not equal);
  - a native sidecar's `restartPolicy` flipped (not equal);
  - a `hostPort` added (not equal);
  - an injected container within its envelope (equal);
  - an injected container over its envelope (not equal, with the reason);
  - an undeclared injected container (not equal, with the reason).
#### Integration tests

- **Integration tests:** with Strict mode, the existing golden tests must pass
  unchanged.
#### e2e tests

- **MultiKueue e2e** (kind, 1 manager + 2 workers), with a toy mutating webhook on one
  worker only that injects `istio-proxy`. Assert:
  - no `OutOfSync`;
  - zero redispatch-counter increments;
  - worker `ClusterQueue` usage equals the declared envelope plus the user's requests.

  Then raise the injected container's requests past its envelope and assert one
  `InjectedContainerExceedsEnvelope` finish and a bounded number of redispatches.
- **Upgrade test:** admit a Workload under `Strict`, switch the config to
  `QuotaRelevant` and restart Kueue. The Workload must not go `OutOfSync`. Repeat in
  the reverse direction.
- **Reproducibility:** run every e2e scenario 20 times and diff the condition
  sequences. Any nondeterminism blocks graduation.

### Graduation Criteria

- **Alpha:** the gate, both modes, injected containers, mode pinning, metrics, and the
  e2e tests above.
- **Beta:**
  - Production evidence from at least one MultiKueue deployment with mesh injection:
    redispatch rate before and after, and the gap between charged and observed
    requests.
  - The LeaderWorkerSet webhook migrated to `Q()`.
  - The config-hash check settled.
- **GA:** consider making `QuotaRelevant` the default for new installations.

## Implementation History

- 2026-11-08: Initial draft, extending #15570.

## Drawbacks

- Adds a second comparison mode that every integration and the MultiKueue spec check must honor, which increases test surface.
- Declared envelopes are an operator-maintained estimate. A wrong estimate either over-reserves quota or fails the comparison loudly.

## Alternatives

1. **Inject identical sidecars on the manager** (today's workaround). This couples the
   manager to every worker's mesh and agent setup and breaks when workers differ.
2. **Exclude MultiKueue Jobs from template-mutating webhooks and inject at the Pod
   level instead.** No Kueue change needed, but sidecar resources go uncharged unless
   the operator also uses RuntimeClass overhead (already counted by Kueue). That works,
   but forces a RuntimeClass per sidecar combination, and the scheduler counts both the
   overhead and the container's requests on the node unless the sidecar's own requests
   are set to zero.
3. **Let the worker re-derive the remote Workload from the mutated Job.** Not viable:
   the Job is created only after the remote Workload has reserved quota.
4. **A per-integration `CustomEquivalenceOptions` hook only.** That interface already
   exists (used by the Ray integrations), but it's code, not configuration, and it
   doesn't reach the MultiKueue spec comparison or resource accounting.
