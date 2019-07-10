---
title: Sidecar Containers
authors:
  - "@joseph-irving"
owning-sig: sig-apps
participating-sigs:
  - sig-apps
  - sig-node
reviewers:
  - "@fejta"
approvers:
  - "@enisoc"
  - "@kow3ns"
editor:
  - "@dixudx"
  - "@zhangxiaoyu-zidif"
creation-date: 2018-05-14
last-updated: 2019-07-10
status: implementable
---

# Sidecar Containers

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Jobs](#jobs)
  - [Startup](#startup)
  - [Shutdown](#shutdown)
- [Goals](#goals)
- [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
    - [API Changes:](#api-changes)
    - [Kubelet Changes:](#kubelet-changes)
      - [Shutdown triggering](#shutdown-triggering)
      - [Sidecars terminated last](#sidecars-terminated-last)
      - [Sidecars started first](#sidecars-started-first)
      - [PreStop hooks sent to Sidecars first](#prestop-hooks-sent-to-sidecars-first)
    - [PoC and Demo](#poc-and-demo)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
    - [Beta -&gt; GA Graduation](#beta---ga-graduation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Implementation History](#implementation-history)
- [Alternatives](#alternatives)
<!-- /toc -->

## Release Signoff Checklist

**ACTION REQUIRED:** In order to merge code into a release, there must be an issue in [kubernetes/enhancements] referencing this KEP and targeting a release milestone **before [Enhancement Freeze](https://github.com/kubernetes/sig-release/tree/master/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core Kubernetes i.e., [kubernetes/kubernetes], we require the following Release Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These checklist items _must_ be updated for the enhancement to be released.

- [ ] kubernetes/enhancements issue in release milestone, which links to KEP (this should be a link to the KEP location in kubernetes/enhancements, not the initial KEP PR)
- [X] KEP approvers have set the KEP status to `implementable`
- [X] Design details are appropriately documented
- [X] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [X] Graduation criteria is in place
- [X] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

**Note:** Any PRs to move a KEP to `implementable` or significant changes once it is marked `implementable` should be approved by each of the KEP approvers. If any of those approvers is no longer appropriate than changes to that list should be approved by the remaining approvers and/or the owning SIG (or SIG-arch for cross cutting KEPs).

**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://github.com/kubernetes/enhancements/issues
[kubernetes/kubernetes]: https://github.com/kubernetes/kubernetes
[kubernetes/website]: https://github.com/kubernetes/website

## Summary

To solve the problem of container lifecycle dependency we can create a new class of container: a "sidecar container" that behaves primarily like a normal container but is handled differently during termination and startup.

## Motivation

SideCar containers have always been used in some ways but just not formally identified as such, they are becoming more common in a lot of applications and as more people have used them, more issues have cropped up.

Here are some examples of the main problems:

### Jobs
 If you have a Job with two containers one of which is actually doing the main processing of the job and the other is just facilitating it, you encounter a problem when the main process finishes; your sidecar container will carry on running so the job will never finish.

The only way around this problem is to manage the sidecar container's lifecycle manually and arrange for it to exit when the main container exits. This is typically achieved by building an ad-hoc signalling mechanism to communicate completion status between containers. Common implementations use a shared scratch volume mounted into all pods, where lifecycle status can be communicated by creating and watching for the presence of files. This pattern has several disadvantages:

* Repetitive lifecycle logic must be rewritten in each instance a sidecar is deployed.
* Third-party containers typically require a wrapper to add this behaviour, normally provided via an entrypoint wrapper script implemented in the k8s container spec. This adds undesirable overhead and introduces repetition between the k8s and upstream container image specs.
* The wrapping typically requires the presence of a shell in the container image, so this pattern does not work for minimal containers which ship without a toolchain.

### Startup
An application that has a proxy container acting as a sidecar may fail when it starts up as it's unable to communicate until its proxy has started up successfully. Readiness probes don't help if the application is trying to talk outbound.

### Shutdown
Applications that rely on sidecars may experience a high amount of errors when shutting down as the sidecar may terminate before the application has finished what it's doing.

### (TODO) ADD PostSidecar User story
<Update Here>

## Goals

Solve issues so that they don't require application modification:
* [25908](https://github.com/kubernetes/kubernetes/issues/25908) - Job completion
* [65502](https://github.com/kubernetes/kubernetes/issues/65502) - Container startup dependencies

## Non-Goals

Allowing multiple containers to run at once during the init phase - this could be solved using the same principal but can be implemented separately. //TODO write up how we could solve the init problem with this proposal

## Proposal

Create a way to define containers as sidecars, this will be an additional field to the `container.lifecycle` spec: `Type` which can be either `Standard` (default), `PreSidecar` or `PostSidecar`.

e.g:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp
    image: myapp
    command: ['do something']
  - name: presidecar
    image: presidecar-image
    lifecycle:
      type: PreSidecar
    command: ["do something to help my app"]
  - name: postsidecar
    image: postsidecar-image
    lifecycle:
      type: PostSidecar
    command: ["do something else to help my app"]

```
PreSidecars will be started before normal containers but after init, so that they are ready before your main processes start.

PostSidecars will be started after normal containers, so that they could do something updation after your main processes start. For example, as for Java application, when we built a stable image for normal container, we find some war files need to be updated. we could do that normal container and postSidecar share one volume, when postSidecar started, it will update war file in the shared volume. That means, normal container's old war would have been updated without building new images. Some other similar examples are applied to update css files for web application.

This will change the Pod startup to look like this:
* Init containers start
* Init containers finish
* PreSidecars start
* PreSidecars become ready
* Containers start
* Containers become ready
* PostSidecars start
* PostSidecars become ready

During pod termination, `PreSidecar` and `PostSidecar` containers are considered equally and will be terminated last:
* Containers sent SIGTERM
* Once all Containers have exited: Sidecars sent SIGTERM



If Containers don't exit before the end of the TerminationGracePeriod then they will be sent a SIGKIll as normal, Sidecars will then be sent a SIGTERM with a short grace period of 5/10 Seconds (up for debate) to give them a chance to cleanly exit.

PreStop Hooks will be sent to sidecars before containers are terminated.
This will be useful in scenarios such as when your sidecar is a proxy so that it knows to no longer accept inbound requests but can continue to allow outbound ones until the the primary containers have shut down.

To solve the problem of Jobs that don't complete: When RestartPolicy!=Always if all normal containers have reached a terminal state (Succeeded for restartPolicy=OnFailure, or Succeeded/Failed for restartPolicy=Never), then all sidecar containers will be sent a SIGTERM.

Sidecars are just normal containers in almost all respects, they have all the same attributes, they are included in pod state, obey pod restart policy etc. The only differences are lifecycle related.

### Implementation Details/Notes/Constraints

The proposal can broken down into four key pieces of implementation that all relatively separate from one another:

* Shutdown triggering for sidecars when RestartPolicy!=Always
* Pre-stop hooks sent to sidecars before non sidecar containers
* Sidecars are terminated after normal containers
* Sidecars start before normal containers

#### API Changes:
As this is a change to the Container spec we will be using feature gating, you will be required to explicitly enable this feature on the api server as recommended [here](https://github.com/kubernetes/community/blob/master/contributors/devel/api_changes.md#adding-unstable-features-to-stable-versions).

New field `Type` will be added to the lifecycle struct:

```go
type Lifecycle struct {
  // Type
  // One of Standard, PreSidecar, PostSidecar.
  // Defaults to Standard
  // +optional
  Type LifecycleType `json:"type,omitempty" protobuf:"bytes,3,opt,name=type,casttype=LifecycleType"`
}
```
New type `LifecycleType` will be added with three constants:
```go
// LifecycleType describes the lifecycle behaviour of the container
type LifecycleType string

const (
  // LifecycleTypeStandard is the default container lifecycle behaviour
  LifecycleTypeStandard LifecycleType = "Standard"
  // LifecycleTypePreSidecar means that the container will start up before standard containers and be terminated after
  LifecycleTypePreSidecar LifecycleType = "PreSidecar"
  // LifecycleTypePostSidecar means that the container will start up after standard containers and be terminated after
  LifecycleTypePostSidecar LifecycleType = "PostSidecar"
)
```
Note that currently the `lifecycle` struct is only used for `preStop` and `postStop` so we will need to change its description to reflect the expansion of its uses.

#### Kubelet Changes:
Broad outline of what places could be modified to implement desired behaviour:

##### Shutdown triggering
Package `kuberuntime`

Modify `kuberuntime_manager.go`, function `computePodActions`. Have a check in this function that will see if all the non-sidecars had permanently exited, if true: return all the running sidecars in `ContainersToKill`. These containers will then be killed via the `killContainer` function which sends preStop hooks, sig-terms and obeys grace period, thus giving the sidecars a chance to gracefully terminate.

##### Sidecars terminated last
Package `kuberuntime`

Modify `kuberuntime_container.go`, function `killContainersWithSyncResult`. Break up the looping over containers so that it goes through killing the non-sidecars before terminating the sidecars.
Note that the containers in this function are `kubecontainer.Container` instead of `v1.Container` so we would need to cross reference with the `v1.Pod` to check if they are sidecars or not. This Pod can be `nil` but only if it's not running, in which case we're not worried about ordering.

##### PreSidecars started first
Package `kuberuntime`

Modify `kuberuntime_manager.go`, function `computePodActions`. If pods has `PreSidecars` it will return these first in `ContainersToStart`, until they are all ready it will not return the non-sidecars. Readiness changes do not normally trigger a pod sync, so to avoid waiting for the Kubelet's `SyncFrequency` (default 1 minute) we can modify `HandlePodReconcile` in the `kubelet.go` to trigger a sync when the `PreSidecars` first become ready (ie only during startup).

##### PreStop hooks sent to PreSidecars and PostSidecars first
Package `kuberuntime`

Modify `kuberuntime_container.go`, function `killContainersWithSyncResult`. Loop over sidecars and execute `executePreStopHook` on them before moving on to terminating the containers. This approach would assume that PreStop Hooks are idempotent as the sidecars would get sent the PreStop hook again when they are terminated.

##### PostSidecars started last
Package `kuberuntime`

Modify `kuberuntime_manager.go`, function `computePodActions`. If pods has `PostSidecars`, it will append these to the last of `ContainersToStart`. PostSidecars will not get created until standard containers are all ready. Readiness changes do not normally trigger a pod sync, so to avoid waiting for the Kubelet's `SyncFrequency` (default 1 minute) we can modify `HandlePodReconcile` in the `kubelet.go` to trigger a sync when the `PostSidecars` first become ready (ie only during startup).

#### PoC and Demo
There is a [PR here](https://github.com/kubernetes/kubernetes/pull/75099) with a working Proof of concept for this KEP, it's just a draft but should help illustrate what these changes would look like.

Please view this [video](https://youtu.be/4hC8t6_8bTs) if you want to see what the PoC looks like in action.

### Risks and Mitigations

You could set all containers to have `lifecycle.type: PreSidecar` or `lifecycle.type: PostSidecar`, this would cause strange behaviour in regards to shutting down the sidecars when all the non-sidecars have exited. To solve this the api could do a validation check that at least one container is not a sidecar.

Init containers would be able to have `lifecycle.type: PreSidecar` or `lifecycle.type: PostSidecar` applied to them as it's an additional field to the container spec, this doesn't currently make sense as init containers are ran sequentially. We could get around this by having the api throw a validation error if you try to use this field on an init container or just ignore the field.

Older Kubelets that don't implement the sidecar logic could have a pod scheduled on them that has the sidecar field. As this field is just an addition to the Container Spec the Kubelet would still be able to schedule the pod, treating the sidecars as if they were just a normal container. This could potentially cause confusion to a user as their pod would not behave in the way they expect, but would avoid pods being unable to schedule.

Shutdown ordering of Containers in a Pod can not be guaranteed when a node is being shutdown, this is due to the fact that the Kubelet is not responsible for stopping containers when the node shuts down, it is instead handed off to systemd (when on Linux) which would not be aware of the ordering requirements. Daemonset and static Pods would be the most effected as they are typically not drained from a node before it is shutdown. This could be seen as a larger issue with node shutdown (also effects things like termination grace period) and does not necessarily need to be addressed in this KEP , however it should be clear in the documentation what guarantees we can provide in regards to the ordering.

## Design Details

### Test Plan
* Units test in kubelet package `kuberuntime` primarily in the same style as `TestComputePodActions` to test a variety of scenarios.
* New E2E Tests to validate that pods with sidecars behave as expected e.g:
 * Pod with sidecars starts sidecars containers before non-sidecars
 * Pod with sidecars terminates non-sidecar containers before sidecars
 * Pod with init containers and sidecars starts sidecars after init phase, before non-sidecars
 * Termination grace period is still respected when terminating a Pod with sidecars
 * Pod with sidecars terminates sidecars once non-sidecars have completed when `restartPolicy` != `Always`
 * Pod phase should be `Failed` if any sidecar exits in failure when `restartPolicy` != `Always`
 * Pod phase should be `Succeeded` if all containers, including sidecars, exit with success when `restartPolicy` != `Always`


### Graduation Criteria
#### Alpha -> Beta Graduation
* Addressed feedback from Alpha testers
* Thorough E2E and Unit testing in place
* The beta API either supports the important use cases discovered during alpha testing, or has room for further enhancements that would support them

#### Beta -> GA Graduation
* Sufficient number of end users are using the feature
* We're confident that no further API changes will be needed to achieve the goals of the KEP
* All known blocking bugs have been fixed

### Upgrade / Downgrade Strategy
When upgrading no changes should be needed to maintain existing behaviour as all of this behaviour is optional and disabled by default.
To activate the feature they will need to enable the feature gate and mark their containers as sidecars in the container spec.

When downgrading `kubectl`, users will need to remove the sidecar field from any of their Kubernetes manifest files as `kubectl` will refuse to apply manifests with an unknown field (unless you use `--validate=false`).

### Version Skew Strategy
Older Kubelets should still be able to schedule Pods that have sidecar containers however they will behave just like a normal container.

## Implementation History

- 14th May 2018: Proposal Submitted
- 26th June 2019: KEP Marked as implementable
- 10th July 2019: Supports `PreSidecars` and `PostSidecars`

## Alternatives

One alternative would be to have a new field in the Pod Spec of `sidecarContainers:` where you could define a list of sidecar containers, however this would require more work in terms of updating tooling/kubelet to support this.

Another alternative would be to change the Job Spec to have a `primaryContainer` field to tell it which containers are important. However I feel this is perhaps too specific to job when this Sidecar concept could be useful in other scenarios.

A boolean flag of `sidecar: true` could be used to indicate which pods are sidecars, however this prevents additional ContainerLifecycles from being added in the future.
