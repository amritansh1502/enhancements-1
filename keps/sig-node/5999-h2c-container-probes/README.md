<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

Follow the guidelines of the [documentation style guide].
In particular, wrap lines to a reasonable length, to make it
easier for reviewers to cite specific portions, and to minimize diff churn on
updates.

[documentation style guide]: https://github.com/kubernetes/community/blob/master/contributors/guide/style-guide.md

To get started with this template:

- [ ] **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking
  up. KEPs should not be checked in without a sponsoring SIG.
- [ ] **Create an issue in kubernetes/enhancements**
  When filing an enhancement tracking issue, please make sure to complete all
  fields in that template. One of the fields asks for a link to the KEP. You
  can leave that blank until this KEP is filed, and then go back to the
  enhancement and add the link.
- [ ] **Make a copy of this template directory.**
  Copy this template into the owning SIG's directory and name it
  `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [ ] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "Title", "Authors", "Owning-sig",
  "Status", and date-related fields.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [ ] **Create a PR for this KEP.**
  Assign it to people in the SIG who are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the KEP clarified and merged quickly. The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a KEP is merged does not mean it is complete or approved. Any KEP
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing KEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR
with suggested changes.

One KEP corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new KEP to move from beta to GA, for example. If
new details emerge that belong in the KEP, edit the KEP. Once a feature has become
"implemented", major changes should get new KEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/keps/NNNN-kep-template/README.md).

**Note:** Any PRs to move a KEP to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the KEP approvers.
If none of those approvers are still appropriate, then changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross-cutting KEPs).
-->
# KEP-5999: HTTP/2 cleartext (h2c) container probes

<!--
This is the title of your KEP. Keep it short, simple, and descriptive. A good
title can help communicate what the KEP is and should be considered as part of
any review.
-->

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
  - [User Stories](#user-stories)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [API Design](#api-design)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
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

HTTP/2 cleartext (h2c) is widely deployed where TLS terminates at the edge
(load balancers, ingress) while workloads still speak plain HTTP/2 on the pod
network—including gRPC and other HTTP/2 stacks. The wire format is defined in
HTTP/2 specifications from the Internet Engineering Task Force (IETF), including
cleartext use with prior knowledge of HTTP/2 on the connection, and mainstream
HTTP implementations support h2c when configured. Kubernetes probes should use
that protocol rather than forcing a separate HTTP/1.1-only listener.

This KEP adds `h2cGet` as a new probe handler alongside the existing `httpGet`,
`grpc`, `tcpSocket`, and `exec` handlers. The new handler performs an HTTP GET
over HTTP/2 cleartext (h2c) to the pod's IP and a numeric port. Operators are
no longer forced to run a second HTTP/1.1-only probe port or fall back to a
`tcpSocket` probe that only confirms the socket is open rather than a valid
HTTP-level response.

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this KEP.  Describe why the change is important and the benefits to users. The
motivation section can optionally provide links to [experience reports] to
demonstrate the interest in a KEP within the wider Kubernetes community.

[experience reports]: https://github.com/golang/go/wiki/ExperienceReports
-->

Community interest and prior discussion: [kubernetes/kubernetes#125599](https://github.com/kubernetes/kubernetes/issues/125599).

HTTP/2 cleartext is defined in IETF HTTP/2 (including cleartext with prior
knowledge of HTTP/2 on the connection) and implemented in widely used HTTP
libraries, so the protocol is mature—not an ad hoc encoding. Together with
widespread in-cluster use when TLS terminates outside the pod, that supports
treating h2c as a first-class probe target.

The kubelet already offers first-class `httpGet`, `https`, `tcpSocket`, `grpc`,
and `exec`, but none performs HTTP/2 on the workload's cleartext listening port.
Operators are currently forced to either add an extra HTTP/1.1-only listener or
accept a `tcpSocket` probe that provides no HTTP-level health signal.

Unlike gRPC probes—where part of the case was that gRPC-related stack was
already in the kubelet—first-class h2c in the HTTP probe path can add or deepen
HTTP/2 client dependency. That cost is easier to accept when it matches how
services already listen, parallels HTTP/2 already reachable via `https` probes
after TLS, and keeps health checks declarative instead of pushing workloads
toward `exec` to approximate HTTP-level checks on the real port.

### Goals

<!--
List the specific goals of the KEP. What is it trying to achieve? How will we
know that this has succeeded?
-->

Enable HTTP/2 cleartext (h2c) container probes so apps are not forced to add a
separate HTTP/1.1-only probe port or rely on TCP probes that do not check a
real HTTP response on the workload port.

### Non-Goals

<!--
What is out of scope for this KEP? Listing non-goals helps to focus discussion
and make progress.
-->

- This KEP does not alter gRPC probes or ingress behavior.
- This KEP does not add h2c support to `httpGet`; that is intentionally a
  separate handler to avoid inheriting `httpGet` footguns (named ports, host
  override) in the h2c path.

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real nitty-gritty.
-->

Add a new `h2cGet` probe handler to `ProbeHandler`. This is modeled after the
existing `grpc` probe type: it uses a numeric-only port, connects directly to
pod IP (no host override), and the protocol is fixed (h2c—not configurable).
Behavior is gated by the `H2CContainerProbe` feature gate on both
kube-apiserver (API validation) and kubelet (probe execution).

The alternative of extending `httpGet` with an `http2Cleartext` boolean was
considered and rejected; see [Alternatives](#alternatives) for details.

### User Stories

<!--
Detail the things that people will be able to do if this KEP is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

As a platform engineer operating Kubernetes behind a TLS-terminating load
balancer, I want to configure liveness/readiness probes that speak HTTP/2
cleartext (h2c) to my app's main port, so that I don't have to run a second
HTTP/1.1-only port (with extra ingress rules and hardening) just to make probes
succeed.

#### Story 2

As an application developer whose service listens with HTTP/2 without TLS inside
the cluster, I want the kubelet to perform a real HTTP health check
(status code on a path) over h2c, so that I am not forced to use a `tcpSocket`
probe that only proves the port is open and does not confirm a valid HTTP
response.

### Notes/Constraints/Caveats (Optional)

- `h2cGet` uses HTTP/2 with prior knowledge (RFC 7540 §3.4), not the HTTP/1.1
  Upgrade mechanism. This is correct for services that exclusively listen on h2c.
- Named ports are not supported; numeric port only (same constraint as `grpc`).
- There is no `host` field. The kubelet always connects to `status.podIP:port`.
  A custom `Host` / `:authority` header can be passed via `httpHeaders` if needed.

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->

| Risk | Mitigation |
|------|-----------|
| Silent HTTP/1.1 downgrade if app does not speak h2c | The h2c client uses prior-knowledge mode; the connection will fail or return a non-2xx response — not silently succeed as HTTP/1.1. Probe failure is explicit. |
| Feature gate skew (apiserver on, kubelet off) | Kubelet must explicitly fail the probe with a clear error when `h2cGet` is set and the gate is off, not silently fall back. |
| New HTTP/2 client dependency in kubelet | `golang.org/x/net/http2` is already transitively present in the kubelet this adds a direct, documented use. |

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

### API Design

A new `h2cGet` field is added to `ProbeHandler` (in `core/v1`), alongside the
existing `exec`, `httpGet`, `tcpSocket`, and `grpc` fields. Exactly one of
these fields must be set per probe.

**New struct `H2CGetAction`:**

```go
// H2CGetAction describes an HTTP GET request over HTTP/2 cleartext (h2c)
// to the pod's IP address. The kubelet connects to status.podIP:port.
// There is no host field—use httpHeaders if a custom Host / :authority
// header is required.
type H2CGetAction struct {
    // Port number on the container. Must be in the range 1 to 65535.
    // Named ports are not supported (unlike httpGet).
    Port int32 `json:"port" protobuf:"varint,1,opt,name=port"`

    // Path to access on the HTTP server. Defaults to "/" if empty.
    // +optional
    Path string `json:"path,omitempty" protobuf:"bytes,2,opt,name=path"`

    // Custom headers to set in the request. HTTP allows repeated headers.
    // +optional
    // +listType=atomic
    HTTPHeaders []HTTPHeader `json:"httpHeaders,omitempty" protobuf:"bytes,3,rep,name=httpHeaders"`
}
```

**Updated `ProbeHandler`:**

```go
type ProbeHandler struct {
    Exec      *ExecAction      `json:"exec,omitempty" ...`
    HTTPGet   *HTTPGetAction   `json:"httpGet,omitempty" ...`
    TCPSocket *TCPSocketAction `json:"tcpSocket,omitempty" ...`
    GRPC      *GRPCAction      `json:"grpc,omitempty" ...`
    // H2CGet specifies an HTTP GET request over HTTP/2 cleartext (h2c).
    // +optional
    H2CGet    *H2CGetAction    `json:"h2cGet,omitempty" protobuf:"bytes,5,opt,name=h2cGet"`
}
```

**Example probe manifest:**

```yaml
readinessProbe:
  h2cGet:
    port: 8080
    path: /readyz
    httpHeaders:
      - name: Custom-Header
        value: my-value
  initialDelaySeconds: 5
  periodSeconds: 10
```

**Validation rules** (enforced only when `H2CContainerProbe` gate is on at
admission):

- `port` must be in [1, 65535]; named ports are rejected.
- `path` defaults to `/` if empty.
- `h2cGet` is mutually exclusive with `httpGet`, `tcpSocket`, `grpc`, `exec`.
- When the gate is off, any pod spec containing `h2cGet` is rejected at
  admission with a descriptive error.

**Kubelet behavior:**

- When executing an `h2cGet` probe and the `H2CContainerProbe` gate is off, the
  kubelet returns a probe failure with an explicit error message (no silent
  HTTP/1.1 downgrade).
- The h2c client uses HTTP/2 with prior knowledge (no HTTP/1.1 Upgrade
  negotiation), implemented via `golang.org/x/net/http2/h2c` transport.
- A 2xx response code is success; any other response or connection error is
  failure, consistent with `httpGet` probe semantics.
- Probe timeout and period settings apply identically to other probe types.

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*
The goal is to ensure that we don't accept enhancements with inadequate testing.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

- [ ] I/we understand the owners of the involved components may require updates
  to existing tests to make this code solid enough prior to committing the
  changes necessary to implement this enhancement.

##### Prerequisite testing updates

<!--
Based on reviewers feedback describe what additional tests need to be added prior
implementing this enhancement to ensure the enhancements have also solid foundations.
-->

No prerequisite test updates are expected. Existing probe unit tests in
`pkg/probe/http` and `pkg/kubelet/prober` cover the surrounding infrastructure
that this change builds on.

##### Unit tests

<!--
In principle every added code should have complete unit test coverage, so providing
the exact set of tests will not bring additional value.
However, if complete unit test coverage is not possible, explain the reason of it
together with explanation why this is acceptable.
-->

- `pkg/probe/h2c` (new package):
  - Build GET URL and headers correctly from `H2CGetAction`.
  - Probe a local h2c test server: 200 → success, 500 → failure, timeout →
    failure, connection refused → failure.
  - Verify HTTP/2 is actually used on the wire (not HTTP/1.1 downgrade).

- `pkg/apis/core/validation`:
  - Valid `h2cGet` with numeric port, path, and headers → allowed.
  - Port out of range, named port → rejected.
  - `h2cGet` combined with `httpGet` → rejected (mutually exclusive).
  - `H2CContainerProbe` gate off → `h2cGet` field rejected at admission.

- `pkg/kubelet/prober`:
  - Gate off + `h2cGet` set → probe returns explicit failure (not silent HTTP/1.1).
  - `newProber` exposes h2c prober when gate is on.
  - Gate on/off toggle: object admitted with gate on, gate turned off → kubelet
    fails the probe clearly on next execution.

<!--
Additionally, for Alpha try to enumerate the core package you will be touching
to implement this enhancement and provide the current unit coverage for those
in the form of:
- <package>: <date> - <current test coverage>
The data can be easily read from:
https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit
-->

- `pkg/probe/http`: `<date>` - `<current test coverage>`
- `pkg/kubelet/prober`: `<date>` - `<current test coverage>`
- `pkg/apis/core/validation`: `<date>` - `<current test coverage>`

##### Integration tests

N/A 
##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.
-->

For Alpha, the following e2e tests will be added.



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

[feature gate]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/
-->

#### Alpha

- `H2CContainerProbe` feature gate implemented (off by default).
- `H2CGetAction` API type added; admission rejects `h2cGet` when gate is off.
- Kubelet executes h2c probes when gate is on; fails explicitly when gate is off.
- Unit tests passing (validation, probe execution, gate toggle).
- Initial e2e tests completed and passing.

#### Beta

- Feature gate on by default.
- No major bugs reported during Alpha.
- e2e tests in Testgrid, stable for at least one release.
- Downgrade tests covering gate-on → gate-off transition.
- All monitoring requirements completed (probe_type label in existing metrics
  covers h2cGet).
- All security review items resolved.
- Feedback gathered from early adopters.

#### GA

- Feature gate removed (behavior always on).
- Stable for at least two releases after Beta.
- No major issues reported.
- Conformance tests added covering `h2cGet` probe lifecycle (Ready -> Unhealthy ->
  restarted).

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

**Upgrade:** No action is required for existing workloads. The `h2cGet` field is
opt-in; pods that do not set it behave exactly as before. To use the feature,
operators must enable the `H2CContainerProbe` feature gate on both
kube-apiserver and kubelet and then update their pod specs.

**Downgrade:** If the feature gate is disabled after pods with `h2cGet` were
admitted, the kubelet will fail those probes explicitly (no silent fallback to
HTTP/1.1). Operators must remove `h2cGet` from pod specs before disabling the
gate, or accept that those pods will report probe failures until the gate is
re-enabled or the pod specs are updated.

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

| Scenario | Behavior |
|----------|----------|
| Old apiserver, new kubelet | `h2cGet` field is unknown to old apiserver -> rejected or dropped at admission before reaching kubelet. No silent HTTP/1.1 execution. |
| New apiserver (gate on), old kubelet (gate off or lacks the type) | Pod is admitted. Old kubelet does not recognize `h2cGet` → probe handler is not dispatched → kubelet must return an explicit probe failure. Recommended: upgrade kubelets before enabling the apiserver gate or before deploying pods with `h2cGet`. |
| New apiserver (gate off), any kubelet | `h2cGet` field is rejected at admission with a validation error. No pods with `h2cGet` reach kubelets. |
| Both new, gate on apiserver / gate off kubelet | Same as "new apiserver, old kubelet (gate off)" above — explicit probe failure, not silent HTTP/1.1 downgrade. |
| Both new, gate on everywhere | Full feature operational. |

**Recommended upgrade order:** enable the `H2CContainerProbe` gate on kubelets
first (or simultaneously), then on kube-apiserver. This prevents a window where
admitted pods have unfailing h2c specs on nodes that cannot execute them.

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



###### Does enabling the feature change any default behavior?



###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?



###### What happens if we reenable the feature if it was previously rolled back?



###### Are there any tests for feature enablement/disablement?

<!--
The e2e framework does not currently support enabling or disabling feature
gates. However, unit tests in each component dealing with managing data, created
with and without the feature, are necessary. At the very least, think about
conversion tests if API types are being modified.

Additionally, for features that are introducing a new API field, unit tests that
are exercising the `switch` of feature gate itself (what happens if I disable a
feature gate after having objects written with the new field) are also critical.
You can take a look at one potential example of such test in:
https://github.com/kubernetes/kubernetes/pull/97058/files#diff-7826f7adbc1996a05ab52e3f5f02429e94b68ce6bce0dc534d1be636154fded3R246-R282
-->



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
missing a bunch of machinery and tooling and can't do that now.
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



###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?



###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

- [x] Metrics
  

###### Are there any missing metrics that would be useful to have to improve observability of this feature?



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
-->

No external cluster services are required.

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?



###### Will enabling / using this feature result in introducing new API types?

Yes — `H2CGetAction`, embedded in `ProbeHandler`. Not a standalone API resource.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?



###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?



###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?



###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?



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

### Option B: Extend `httpGet` with an `http2Cleartext` boolean

This approach would add an optional `http2Cleartext *bool` field to the existing
`HTTPGetAction` struct:

```go
type HTTPGetAction struct {
    Path           string             `json:"path,omitempty" ...`
    Port           intstr.IntOrString `json:"port" ...`
    Host           string             `json:"host,omitempty" ...`
    Scheme         URIScheme          `json:"scheme,omitempty" ...`
    HTTPHeaders    []HTTPHeader       `json:"httpHeaders,omitempty" ...`
    // +optional
    HTTP2Cleartext *bool              `json:"http2Cleartext,omitempty" ...`
}
```

**Why it was not chosen:**

1. `httpGet` carries footguns (named ports, `host` override) that the h2c use
   case does not need and that have caused confusion in production. A new probe
   type gives us a clean namespace with strict validation from the start, the
   same rationale used when `grpc` was added as a dedicated probe type rather
   than an extension of `httpGet`.
2. Validation is combinatorially more complex: `http2Cleartext: true` combined
   with `scheme: HTTPS` is undefined; named ports with h2c have no clear
   semantics; `host` override with h2c could direct the probe away from the pod
   IP. All of these require explicit validation rules that grow the test matrix.
3. `postStart`/`preStop` hooks already support `httpGet`. Extending `httpGet`
   for h2c would implicitly make h2c available in lifecycle hooks without a
   deliberate decision, complicating the scope.

This approach may be revisited in the future if user feedback indicates that
having a separate handler name is a usability burden, but for the initial
implementation the new dedicated handler is the safer choice.



## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->

No new infrastructure is needed.
