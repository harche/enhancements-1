---
title: worker-latency-profiles
authors:
  - "@harche"
reviewers:
  - "@rphillips"
  - "@sttts"
  - "@soltysh"
approvers:
  - "@rphillips"
  - "@sttts"
  - "@soltysh"
creation-date: 2021-12-06
last-updated: 2021-12-06
status: implementable
see-also:
  - https://github.com/kubernetes-sigs/kubespray/blob/master/docs/kubernetes-reliability.md
  - https://github.com/Azure/aks-engine/blob/master/docs/topics/clusterdefinitions.md#controllermanagerconfig
replaces:
superseded-by:
---

# WorkerLatencyProfile (Day-0 only support) (WIP)

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)


## Summary

To make sure the Openshift cluster keeps running optimally where network latency between the control plane and the worker nodes may not always be at its best (e.g. Edge cases), we can tweak the frequency of the status updates done by the `Kubelet` and the corresponding reaction times of the `Kube Controller Manager` and `Kube API Server`.

## Motivation

To accommodate the higher network latency (such as Edge cases) cluster may experience, we need to adjust how frequently the `Kubelet` updates its status and how components such as `Kube Controller Manager` and `Kube API Server`react to those status updates.

The main motivation of this enhancement is to allow setting relevant arguments for these critical components in a more controlled manner instead of letting the users directly modify them manually which will make the cluster unsupported. This will also eliminate any room for manual errors that could lead to unscheduled downtime.

### Goals

* Enable Openshift cluster to fine tune the reliability in medium to high latency scenarios by specifying simple `WorkerLatencyProfile`.

### Non-Goals

* Modify the existing `Kubelet`, `Kube Controller Manager` or `Kube API Server` code in any way.
* Allow any kind of different latency scenarios between masters

### User Stories

* User wants to fine tune the cluster reliability for their node latency scenario while making sure they are protected from setting parameters that could potentially break the cluster.

## Proposal

* The option to set `WorkerLatencyProfile` will have to reside in a centralized location. The [OpenShift Infrastructure config object](https://github.com/openshift/api/blob/master/config/v1/types_infrastructure.go#L28) contains information describing how a cluster functions including cloud config and platform specification for each cloud. Setting the  `WorkerLatencyProfile` is an infrastructure setting.
* [MCO](https://github.com/openshift/machine-config-operator) set the appropriate value of the `Kubelet` flag `--node-status-update-frequency`
* [KCMO](https://github.com/openshift/cluster-kube-controller-manager-operator) set the appropriate value of the `Kube Controller Manager` flag `--node-monitor-grace-period`
* [CKAO](https://github.com/openshift/cluster-kube-apiserver-operator)set the appropriate value of the `Kube API Server` flags `--default-not-ready-toleration-seconds` and `--default-unreachable-toleration-seconds`


### Graduation Criteria

#### Dev Preview -> Tech Preview
* Succeffully set/update the relavent arguments of the `Kubelet`, `Kube Controller Manager` and `Kube API Server` that correspond to the user specified `WorkerLatecyProfile`
* End user documentation

#### Tech Preview -> GA
* More testing (upgrade, downgrade, scale)

#### Removing a deprecated feature

N/A

## Design Details

### API Extensions

```go
type WorkerLatencyProfileType string

const (
    // Medium Update and Average Reaction
    MediumUpdateAndAverageReaction WorkerLatencyProfileType = "MediumUpdateAndAverageReaction"

    // Low Update and Slow Reaction
    LowUpdateAndSlowReaction WorkerLatencyProfileType = "LowUpdateAndSlowReaction"

    // Default values of relavent Kubelet, Kube Controller Manager and Kube API Server
    Default WorkerLatencyProfileType = "Default"
)

type Node struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// +required
	Spec NodeSpec `json:"spec"`
}


type NodeSpec struct {
  WorkerLatencyProfile WorkerLatencyProfileType `json:"workerLatencyProfile,omitempty"`

  // an eventual additional option might be crun in the future. This explains
  //   why a new struct may be necessary
  // CgroupMode CgroupMode `json:"cgroupMode,omitempty"`
  //
  // CrunEnabled bool ...
}

```

### Operational Aspects of API Extensions


#### Default Update And Default Reaction
Openshift ships with following default configuration which works quite well in most cases.

By default `Kubelet` updates it's status every 10 seconds (`--node-status-update-frequency`), while `Kube Controller Manager` checks the statuses of `Kubelet` every 5 seconds (`--node-monitor-period`).
Before considering the `Kubelet` unhealthy `Kube Controller Manager` will wait for 40 seconds (`--node-monitor-grace-period`) to hear from the `Kubelet`. Once the `Kubelet` is considered unhealthy the node is given `node.kubernetes.io/not-ready` or `node.kubernetes.io/unreachable` taints.
Pods with `NoExecute` taint will get executed as per [tolerationSeconds](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions), but pods without such taint will get evicted in 300 seconds (`--default-not-ready-toleration-seconds` and `--default-unreachable-toleration-seconds` settings of the `Kube API Server`)

| Component               | Flag Name                                | Flag Value  |
| -----------             | -----------                              | ----------- |
| Kubelet                 | --node-status-update-frequency           |    10s      |
| Kube Controller Manager | --node-monitor-grace-period              |    40s      |
| Kube API Server         | --default-not-ready-toleration-seconds   |    300s     |
| Kube API Server         | --default-unreachable-toleration-seconds |    300s     |


#### Medium Update And Average Reaction

While the default configuration works well in most cases, sometimes the worker nodes may find themselves in a network with slightly higher than usual latency.

In this scenario, we will reduce the frequency of `Kubelet` updates to every 20 seconds (`--node-status-update-frequency`) and change the node monitor grace period of the `Kube Controller Manager` to 2 minutes (`--node-monitor-grace-period`). `--default-not-ready-toleration-seconds` and `--default-unreachable-toleration-seconds` of the Kube API Server will be set to 60 seconds.

`Kube Controller Manager` will wait for 2 minutes to consider unhealthy status for the node and since `--default-not-ready-toleration-seconds` and `--default-unreachable-toleration-seconds` of the Kube API Server are set to 60 seconds the total time will be 3 minutes before eviction process starts.


| Component               | Flag Name                                | Flag Value  |
| -----------             | -----------                              | ----------- |
| Kubelet                 | --node-status-update-frequency           |    20s      |
| Kube Controller Manager | --node-monitor-grace-period              |    2m       |
| Kube API Server         | --default-not-ready-toleration-seconds   |    60s      |
| Kube API Server         | --default-unreachable-toleration-seconds |    60s      |

#### Low Update and Slow reaction

Worker nodes may find themselves in a network with extremely high latency and/or bad reliability.
In this scenario, we will reduce the frequency of `Kubelet` updates further to every 1 minute (`--node-status-update-frequency`) and change the node monitor grade period of the `Kube Controller Manager` to every 5 minutes (`--node-monitor-grace-period`). `--default-not-ready-toleration-seconds` and `--default-unreachable-toleration-seconds` of the Kube API Server are set to 60 seconds.

`Kube Controller Manager` will wait for 5 minutes to consider unhealthy status for the node and since `--default-not-ready-toleration-seconds` and `--default-unreachable-toleration-seconds` of the `Kube API Server` are set to 60 seconds the total time will be 6 minutes before eviction process starts.


| Component               | Flag Name                                | Flag Value  |
| -----------             | -----------                              | ----------- |
| Kubelet                 | --node-status-update-frequency           |    1m       |
| Kube Controller Manager | --node-monitor-grace-period              |    5m       |
| Kube API Server         | --default-not-ready-toleration-seconds   |    60s      |
| Kube API Server         | --default-unreachable-toleration-seconds |    60s      |


#### Failure Modes
1. In case of failure `WorkerLatencyProfileStatus` should point towards failed component(s)


#### Support Procedures

### Test Plan


### Version Skew Strategy

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?

  This functionality only modifies the existing arguments of the `Kubelet`, `Kube Controller Manager` and `Kube API Server`. As long as components keep the respective flags in place, version skew should not have any impact on this work.

- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?

  No.

- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet.

  No


### Risks and Mitigations



1. Any bug in [MCO](https://github.com/openshift/machine-config-operator), [KCMO](https://github.com/openshift/cluster-kube-controller-manager-operator) or [CKAO](https://github.com/openshift/cluster-kube-apiserver-operator) in setting the appropiate values for the respective flags might put the cluster at risk.

2. We are changing `--node-monitor-grace-period` argument of the `Kube Controller Manager` and `--default-not-ready-toleration-seconds` and `--default-unreachable-toleration-seconds` of arguments the `Kube API Server`.
Although we are targeting only worker nodes with this enhancement, both `Kube Controller Manager` and `Kube API Server` do not allow us to set those respective arguments only for the subset of the available nodes.
i.e. They are applied cluster wide including master nodes. We are only proposing `WorkerLatencyProfile` with this enhancement that slow down updates from the Kubelet on the worker nodes and the corresponding reaction time from the control plane.
So even though 100% reliable network connections amongst master nodes is assumed, when the user applies `WorkerLatencyProfile` there will be delays evicting the pods running on the master nodes than their usual default values.

### Upgrade / Downgrade Strategy

Since this feature is controlled using the `KubeletConfig` and `ConfigObserver`, upgrade/downgrade strategies applicable for the `KubeletConfig` and `ConfigObserver` are applicable here too.

## Drawbacks

## Alternatives

Since we don't want users to manually modify the arguments for the components involved, there isn't any viable alternative that could safely work.

## Implementation History


