# KEP-4939: TLS Credentials in gRPC Probe

<!--
A table of contents is helpful for quickly jumping to sections of a KEP and for
highlighting any additional information provided beyond the standard KEP
template.

Ensure the TOC is wrapped with
  <code>&lt;!-- toc --&rt;&lt;!-- /toc --&rt;</code>
tags, and then generate with `hack/update-toc.sh`.
-->

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [API Design](#api-design)
  - [Feature Gate Behavior](#feature-gate-behavior)
  - [Kubelet Probe Execution](#kubelet-probe-execution)
  - [gRPC Transport Credentials](#grpc-transport-credentials)
  - [Test Plan](#test-plan)
    - [Prerequisite testing updates](#prerequisite-testing-updates)
    - [Unit tests](#unit-tests)
    - [Integration tests](#integration-tests)
    - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) within one minor version of promotion to GA
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

<!--
This section is incredibly important for producing high-quality, user-focused
documentation such as release notes or a development roadmap. It should be
possible to collect this information before implementation begins, in order to
avoid requiring implementors to split their attention between writing release
notes and implementing the feature itself. KEP editors and SIG Docs
should help to ensure that the tone and content of the `Summary` section is
useful for a wide audience.

A good summary is probably at least a paragraph in length.
-->


The new gRPC health probe enables developers to probe [gRPC health servers](https://github.com/grpc-ecosystem/grpc-health-probe) from the node. This allows them to stop using workarounds such as this grpc-health-probe paired with `exec` probes.

It allows natively running health checks on gRPC services without deploying additional binaries as well as other benefits outlined in [the announcement](https://kubernetes.io/blog/2022/05/13/grpc-probes-now-in-beta/).

A limitation in the current implementation is that it only supports gRPC servers that do not leverage TLS connections. Even if they are not concerned about certificate verification for the health check, a connection cannot be established at all if the server is expecting TLS and the client is not.

This enhancement aims to add configuration options to enable TLS on the gRPC probe.

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this KEP.  Describe why the change is important and the benefits to users. The
motivation section can optionally provide links to [experience reports] to
demonstrate the interest in a KEP within the wider Kubernetes community.

[experience reports]: https://github.com/golang/go/wiki/ExperienceReports
-->
We often deploy internal gRPC services on our cluster. These deployments provide internal services and it's simple to add the health server to them so they can be verified through a single interface.

It's also worth noting we have and internal CA that signs certs for communicating with these servers and all of them use TLS.

Currently, we are using the exec probe for readiness and liveness configured as:

```yaml
livenessProbe:
  exec:
    command:
      - "/bin/grpc_health_probe"
      - "-addr=:8443"
      - "-tls"
      - "-tls-no-verify"
```
We would really like to switch to the gRPC probes introduced in 1.24 but are unable to do so since there is no way to configure it to use a TLS connection when reaching out to the health server.

Instead we must continue to rely on the exec probe and cannot reap the benefits described in [the announcement](https://kubernetes.io/blog/2022/05/13/grpc-probes-now-in-beta/).


### Goals

<!--
List the specific goals of the KEP. What is it trying to achieve? How will we
know that this has succeeded?
-->
The primary goal is to support TLS connections when using the grpc probe. The probe will use TLS but not verify the certificate.

### Non-Goals

<!--
What is out of scope for this KEP? Listing non-goals helps to focus discussion
and make progress.
-->
It is not a goal of this KEP to support providing a certificate to verify the TLS connection.

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

Add a new optional `mode` field alongside `port` and `service` in the
[Probe GRPCAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#grpcaction-v1-core).
It indicates whether the probe should connect using TLS or plaintext, and
serves as a basis for future TLS-related probe functionality if desired.

### User Stories (Optional)

<!--
Detail the things that people will be able to do if this KEP is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

As a platform engineer running internal gRPC services with TLS enabled
(via an internal CA), I want to configure native gRPC liveness and readiness
probes with `mode: TLS` so that I can stop bundling `grpc_health_probe` in
every container image and relying on `exec` probes just to health-check a
TLS endpoint.

#### Story 2

As an application developer whose gRPC server only accepts TLS connections,
I want the kubelet's built-in gRPC probe to connect over TLS so that my
healthy containers are not marked unhealthy due to a TLS handshake failure.

### Notes/Constraints/Caveats (Optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->
Adds more code to kubelet and surface area to `Pod.Spec`.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

### API Design

A new optional `mode` field is added to the existing `GRPCAction` struct.
The field is a pointer to a `GRPCProbeMode` enum (`nil` preserves existing
plaintext behavior). Gated by the `GRPCContainerProbeTLS` feature gate on
both kube-apiserver (validation / field-dropping) and kubelet (probe execution).

**New enum `GRPCProbeMode`:**

```go
// +enum
type GRPCProbeMode string

const (
    GRPCProbeModePlaintext GRPCProbeMode = "Plaintext"
    GRPCProbeModeTLS       GRPCProbeMode = "TLS"
)
```

**Updated `GRPCAction`:**

```go
type GRPCAction struct {
    Port    int32    `json:"port" protobuf:"varint,1,opt,name=port"`
    Service *string  `json:"service" protobuf:"bytes,2,opt,name=service"`
    // +featureGate=GRPCContainerProbeTLS
    // +optional
    Mode *GRPCProbeMode `json:"mode,omitempty" protobuf:"bytes,3,opt,name=mode,casttype=GRPCProbeMode"`
}
```

**Pod spec example: TLS probe:**

```yaml
livenessProbe:
  grpc:
    port: 8443
    mode: TLS
```

**Pod spec example: explicit plaintext:**

```yaml
livenessProbe:
  grpc:
    port: 50051
    mode: Plaintext
```

**Pod spec example: default (nil mode, plaintext, backward compatible):**

```yaml
livenessProbe:
  grpc:
    port: 50051
```

### Feature Gate Behavior

When `GRPCContainerProbeTLS` is **disabled**:

- The `mode` field is **dropped** from new pods by `dropDisabledGRPCContainerProbeTLS`.
- Validation **rejects** pods that set `mode` with a `Forbidden` error.
- If an existing pod already has `mode` set (created while the gate was enabled),
  the field is **preserved** for backward compatibility.

### Kubelet Probe Execution

In `pkg/kubelet/prober/prober.go`, the kubelet reads the `Mode` field:

```go
useTLS := p.GRPC.Mode != nil && *p.GRPC.Mode == v1.GRPCProbeModeTLS
```

This boolean is passed to the gRPC prober.

### gRPC Transport Credentials

In `pkg/probe/grpc/grpc.go`, transport credentials are selected based on the
`useTLS` flag:

```go
var transportCreds credentials.TransportCredentials
if useTLS {
    transportCreds = credentials.NewTLS(&tls.Config{
        InsecureSkipVerify: true,
    })
} else {
    transportCreds = insecure.NewCredentials()
}
```

`InsecureSkipVerify: true` is used because the probe connects to the pod's own
IP / localhost where certificate verification is impractical.

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*
The goal is to ensure that we don't accept enhancements with inadequate testing.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

[ ] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

<!--
Based on reviewers feedback describe what additional tests need to be added prior
implementing this enhancement to ensure the enhancements have also solid foundations.
-->

##### Unit tests

<!--
In principle every added code should have complete unit test coverage, so providing
the exact set of tests will not bring additional value.
However, if complete unit test coverage is not possible, explain the reason of it
together with explanation why this is acceptable.
-->

<!--
Additionally, for Alpha try to enumerate the core package you will be touching
to implement this enhancement and provide the current unit coverage for those
in the form of:
- <package>: <date> - <current test coverage>
The data can be easily read from:
https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit

This can inform certain test coverage improvements that we want to do before
extending the production code to implement this enhancement.
-->

The following unit tests have been added:

- **`pkg/api/pod`**
  - `TestDropGRPCContainerProbeTLS`: Verifies that `mode` is stripped from all container types (regular, init, ephemeral) when the feature gate is disabled, and preserved when enabled or when the field is already persisted on an existing pod.
  - `TestGRPCContainerProbeTLSValidationOptions`: Verifies that `AllowGRPCContainerProbeTLS` is set correctly based on gate state and old pod spec.

- **`pkg/apis/core/validation`**
  - `TestValidateGRPCAction`: Verifies that `mode: TLS` and `mode: Plaintext` are accepted when the gate is enabled, rejected with `Forbidden` when disabled, and unsupported values like `"Verify"` are rejected with `NotSupported`.

- **`pkg/apis/core/v1`**
  - `TestSetDefaultProbeGRPCMode`: Verifies that `mode: TLS`, `mode: Plaintext`, and `nil` mode are all preserved through round-trip defaulting with no unwanted mutation.

- **`pkg/probe/grpc`**
  - `TestGrpcProber_Probe`: Existing plaintext probe tests updated to pass `useTLS=false`.
  - `TestGrpcProber_Probe_TLS`: Verifies TLS probe succeeds against a TLS server, plaintext probe fails against a TLS server, and TLS probe fails against a plaintext server.

##### Integration tests

<!--
Integration tests are contained in https://git.k8s.io/kubernetes/test/integration.
Integration tests allow control of the configuration parameters used to start the binaries under test.
This is different from e2e tests which do not allow configuration of parameters.
Doing this allows testing non-default options and multiple different and potentially conflicting command line options.
For more details, see https://github.com/kubernetes/community/blob/master/contributors/devel/sig-testing/testing-strategy.md

If integration tests are not necessary or useful, explain why.
-->

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, document that tests have been written,
have been executed regularly, and have been stable.
This can be done with:
- permalinks to the GitHub source code
- links to the periodic job (typically https://testgrid.k8s.io/sig-release-master-blocking#integration-master), filtered by the test name
- a search in the Kubernetes bug triage tool (https://storage.googleapis.com/k8s-triage/index.html)
-->

Integration tests will be added.

- [test name](https://github.com/kubernetes/kubernetes/blob/2334b8469e1983c525c0c6382125710093a25883/test/integration/...): [integration master](https://testgrid.k8s.io/sig-release-master-blocking#integration-master?include-filter-by-regex=MyCoolFeature), [triage search](https://storage.googleapis.com/k8s-triage/index.html?test=MyCoolFeature)

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, document that tests have been written,
have been executed regularly, and have been stable.
This can be done with:
- permalinks to the GitHub source code
- links to the periodic job (typically a job owned by the SIG responsible for the feature), filtered by the test name
- a search in the Kubernetes bug triage tool (https://storage.googleapis.com/k8s-triage/index.html)

We expect no non-infra related flakes in the last month as a GA graduation criteria.
If e2e tests are not necessary or useful, explain why.
-->

The following e2e tests have been added in `test/e2e/common/node/container_probe.go`:

- **`should *not* be restarted with a GRPC liveness probe with TLS mode`**: Creates a pod with a TLS-enabled gRPC health server and a liveness probe using `mode: TLS`. The probe should succeed and the restart count must remain zero.
- **`should be restarted with a GRPC liveness probe when not using TLS against a TLS server`**: Creates a pod with a TLS-enabled gRPC health server and a plaintext liveness probe (no `mode` set). The probe should fail the TLS handshake and the container must be restarted.

Both tests are gated by `framework.WithFeatureGate(features.GRPCContainerProbeTLS)`.

### Graduation Criteria

<!--
**Note:** *Not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, [feature gate] graduations, or as
something else. The KEP should keep this high-level with a focus on what
signals will be looked at to determine graduation.

Consider the following in developing the graduation criteria for this enhancement:
- [Maturity levels (`alpha`, `beta`, `stable`)][maturity-levels]
- [Feature gate][feature gate] lifecycle
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc
definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning)
or by redefining what graduation means.

In general we try to use the same stages (alpha, beta, GA), regardless of how the
functionality is accessed.

[feature gate]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

Below are some examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

-->

#### Alpha

- API field implemented and functional
- Unit tests passing
- Documentation available

#### Beta

- No major bugs reported during alpha
- Gather feedback from users

#### GA

- Stable for at least two releases
- No major issues reported


### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to maintain previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to make use of the enhancement?
-->

No special upgrade steps are required. The `mode` field defaults to `nil`,
which preserves the existing plaintext behavior. Existing pods are unaffected
on upgrade.

On downgrade (or if the `GRPCContainerProbeTLS` feature gate is disabled):

- The API server drops the `mode` field from new or updated pods.
- Existing pods that had `mode` set retain the field in etcd, but the kubelet
  on the older version ignores it and falls back to plaintext.
- Pods relying on `mode: TLS` to reach a TLS-only server will begin failing
  probes, which is the same behavior that existed before this feature.

No data migration is needed. The feature is purely additive and opt-in.

### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and nodes?
- How does an n-3 kubelet or kube-proxy without this feature available behave when this feature is used?
- How does an n-1 kube-controller-manager or kube-scheduler without this feature available behave when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->

This feature requires the `GRPCContainerProbeTLS` feature gate on both
kube-apiserver and kubelet.

- **API server newer than kubelet:** The API server accepts `mode: TLS`,
  but the older kubelet ignores the field and dials plaintext. TLS-only
  servers will fail probes, identical to pre-feature behavior.
- **Kubelet newer than API server:** The older API server drops the `mode`
  field, so the kubelet never sees it and dials plaintext.

Both components must have the gate enabled for TLS probes to function.
Partial enablement degrades gracefully to plaintext with no crashes or
unexpected behavior.

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the KEP to move to `implementable` status and be included in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.
-->

### Feature Enablement and Rollback

<!--
This section must be completed when targeting alpha to a release.
-->

###### How can this feature be enabled / disabled in a live cluster?

<!--
Pick one of these and delete the rest.

Documentation is available on [feature gate lifecycle] and expectations, as
well as the [existing list] of feature gates.

[feature gate lifecycle]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[existing list]: https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/
-->

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: `GRPCContainerProbeTLS`
  - Components depending on the feature gate:
    - `kube-apiserver` (validation, field dropping)
    - `kubelet` (probe execution)

###### Does enabling the feature change any default behavior?

No. The `mode` field defaults to `nil`, which preserves the existing plaintext
behavior. Only pods that explicitly set `mode: TLS` are affected.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Disabling the `GRPCContainerProbeTLS` feature gate and restarting
kube-apiserver and kubelet will cause the `mode` field to be dropped from new
or updated pods. Existing pods that had `mode` set retain the field in etcd,
but the kubelet will ignore it and fall back to plaintext. Pods relying on
`mode: TLS` to reach a TLS-only server will begin failing probes, which is
the same behavior that existed before this feature.

###### What happens if we reenable the feature if it was previously rolled back?

Pods that still have `mode: TLS` persisted in etcd will start using TLS
probes again. New pods can set `mode: TLS` as expected.

###### Are there any tests for feature enablement/disablement?

Yes. `TestDropGRPCContainerProbeTLS` verifies the `mode` field is dropped
when the gate is disabled and preserved when enabled or when the old pod
already uses it. `TestGRPCContainerProbeTLSValidationOptions` verifies
validation allows or rejects the field based on gate state.

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout or rollback fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some API servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->

###### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

- [ ] Events
  - Event Reason: 
- [ ] API .status
  - Condition name: 
  - Other field: 
- [ ] Other (treat as last resort)
  - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

<!--
This is your opportunity to define what "normal" quality of service looks like
for a feature.

It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99.9% of /health requests per day finish with 200 code

These goals will help you determine what you need to measure (SLIs) in the next
question.
-->

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

<!--
Think about both cluster-level services (e.g. metrics-server) as well
as node-level agents (e.g. specific version of CRI). Focus on external or
optional services that are needed. For example, if this feature depends on
a cloud provider API, or upon an external software-defined storage or network
control plane.

For each of these, fill in the following—thinking about running existing user workloads
and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?

<!--
Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
Focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)
-->

###### Will enabling / using this feature result in introducing new API types?

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

###### Will enabling / using this feature result in any new calls to the cloud provider?

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

<!--
Focus not just on happy cases, but primarily on more pathological cases
(e.g. probes taking a minute instead of milliseconds, failed pods consuming resources, etc.).
If any of the resources can be exhausted, how this is mitigated with the existing limits
(e.g. pods per node) or new limits added by this KEP?

Are there any tests that were run/should be run to understand performance characteristics better
and validate the declared limits?
-->

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?

###### What are other known failure modes?

<!--
For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.
-->

###### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

**Nested `tls` struct with `mode` field.** The
[Previous KEP proposal](https://github.com/kkoch986/enhancements/blob/0e2ba3bb95e73aaed31e5dfb60aa2061424de265/keps/sig-node/4939-tls-in-grpc-probe/README.md)
added a `tls` sub-struct to `GRPCAction` with a `mode` field (`NoVerify`).
The presence of the struct indicated TLS should be used. This was rejected by
reviewers because the probe will never validate certificates (it connects to
localhost/pod IP), so a nested struct adds unnecessary complexity. A flat
`mode` field on `GRPCAction` is simpler and sufficient.

**Boolean `tls` flag.** A simple `tls: true/false` boolean was
[discussed](https://github.com/kubernetes/enhancements/pull/5029#discussion_r1936341743).
This was rejected in favor of a string enum (`mode`) to allow explicit naming
of the connection type (`"TLS"`, `"Plaintext"`) and to leave room for future
values without a breaking API change.

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->