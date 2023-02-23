---
title: windows-node-cluster-wide-egress-proxy
authors:
  - "@saifshaikh48"
reviewers: # Include a comment about what domain expertise a reviewer is expected to bring and what area of the enhancement you expect them to focus on. For example: - "@networkguru, for networking aspects, please look at IP bootstrapping aspect"
  - "@openshift/openshift-team-windows-containers"
  - TBD -- sdn?
approvers:
  - "@aravindhp"
  - TBD
api-approvers: # In case of new or modified APIs or API extensions (CRDs, aggregated apiservers, webhooks, finalizers). If there is no API change, use "None"
  - None
creation-date: 2023-02-16
last-updated: 2023-03-02
tracking-link:
  - Feature: [OCPBU-22: Support cluster-wide proxy on Windows Containers feature](https://issues.redhat.com/browse/OCPBU-22)
  - Epic: [WINC-802: Windows Node Proxy Configuration](https://issues.redhat.com/browse/WINC-802)
see-also:
  - "/enhancements/proxy/global-cluster-egress-proxy.md"
---

# Windows Node Cluster-Wide Egress Proxy

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Operational readiness criteria is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

The goal of this enhancement proposal is to allow Windows nodes to consume and use global egress proxy configuration
when making external requests outside the cluster's internal network. OpenShift customers may require that external
traffic is passed through a proxy for security reasons, and Windows instances are no exception. There already exists a
protocol for publishing [cluster-wide proxy](https://docs.openshift.com/container-platform/4.12/networking/enable-cluster-wide-proxy.html) settings, which is consumed by different OpenShift components (Linux worker nodes and infra nodes, CVO and OLM managed
operators); Windows worker nodes do not currently consume or respect proxy settings. This effort will work to plug this
feature disparity by making the [Windows Machine Config Operator](https://github.com/openshift/windows-machine-config-operator)
(WMCO) proxy aware at install time and reactive during runtime.

## Motivation

The motivation here is to expand the Windows containers production use case, enabling users to add Windows nodes and run
workloads easily and successfully in a proxied cluster. This is an extremely important ask for customer environments
where Windows nodes are often tasked with pulling images from secure registries or making requests to off-cluster
services, and those that use a custom public key infrastructure.

### Goals

* Create an automated mechanism for WMCO to consume global egress proxy config from existing platform recources.
  Includes:
  + proxy connection information (http/https URLs)
  + additional certificate authorities required to validate the proxy's certificate
  + list of networks that should bypass the proxy (no-proxy hostnames, domains, IP addresses or other network CIDRs)
* Configure the proxy settings in WMCO-managed components on Windows nodes (kubelet, containerd runtime)
* Pick up changes to the cluster-wide proxy settings during cluster runtime
* Maintain normal functionality in non-proxied clusters

### Non-Goals

* Anything to do with *ingress* or reverse proxy settings is out of scope
* Monitor or automatically replace expired CAs in the cluster's trust bundle or Windows components

## Proposal

<!-- This is where we get down to the nitty gritty of what the proposal
actually is. Describe clearly what will be changed, including all of
the components that need to be modified and how they will be
different. Include the reason for each choice in the design and
implementation that is proposed here, and expand on reasons for not
choosing alternatives in the Alternatives section at the end of the
document. -->

As it stands today, the source of truth for cluster-wide proxy settings is the `Proxy` resource with the name `cluster`.
The spec of this resource is both user-defined, as well as adjusted by the cluster network operator (CNO). OLM is a
subscriber to these settings -- it forwards the settings to the CSV of managed operators, so the WMCO container will
automatically get the required `NO_PROXY`, `HTTP_PROXY`, and `HTTPS_PROXY` environment variables on startup. In fact,
OLM will update and restart the operator pod with proper environment variables if the `Proxy` resource changes.

Since WMCO is a day 2 operator, it will pick up proxy settings during runtime regardless of if proxy settings were set
during cluster install time or at some point during the cluster's lifetime.



All changes will be limited to the Windows Machine Config Operator.

On the Linux side this is achieved through `MachineConfigs` -- when global proxy settings are changed, the ignition
config is re-rendered by MCO to include a systemd unit `EnvironmentFile=/etc/mco/proxy.env`, with proxy vars set in this file.
WMCO could watch the MachineConfig for changes and copy over this file as part of Windows node configuration.

### User Stories

User stories can also be found within the node proxy epic: [WINC-802](https://issues.redhat.com/browse/WINC-802)

#### Story 1 - [WINC-637: Set cluster-wide proxy environment variables on Windows instance](https://issues.redhat.com/browse/WINC-637)

As an administrator of a network with strict traffic egress policies,
I want Windows nodes on my OpenShift cluster to successfully make proxied
external requests for images and other external service interactions.
This includes updating proxy settings when the cluster-wide config changes.

#### Story 2 - [WINC-633: Support custom CA certificates for cluster-wide proxy](https://issues.redhat.com/browse/WINC-633)

As an administrator of a network using a man-in-the-middle proxy which
is performing traffic decryption/re-encryption, I want to provide valid
CAs that can trust my proxy's certificate to Windows nodes so they and
their workloads will trust my proxy.

#### Story 3 - [WINC-688: Update Proxy certs on Windows instances when the certs are rotated](https://issues.redhat.com/browse/WINC-688)

As an OpenShift administrator, I want user-provided proxy CA certificates
to be automatically updated on Windows nodes when I manually rotate them
in cluster-wide resources so that Windows nodes use the latest trust bundle
and outbound traffic continues to respect proxy settings.

### Workflow Description and Variations

**cluster creator** is a human user responsible for deploying a cluster.
**cluster administrator** is a human user responsible for managing cluster settings including network egress policies.

There are 3 different workflows that affect the cluster-wide proxy use case.
1. A cluster creator specifies global proxy settings at install time
2. A cluster administrator introduces new global proxy settings during runtime in a proxy-less cluster
3. A cluster administrator changes or removes existing global proxy settings during cluster runtime

The first scenario would occur through their [install-config.yaml](https://docs.openshift.com/container-platform/4.12/networking/configuring-a-custom-pki.html#installation-configure-proxy_configuring-a-custom-pki).
The latter 2 scenarios occur through changing the []`Proxy` object named `cluster`](https://docs.openshift.com/container-platform/4.12/networking/enable-cluster-wide-proxy.html#nw-proxy-configure-object_config-cluster-wide-proxy)
or by modifying certificates present in their [trustedCA ConfigMap](https://docs.openshift.com/container-platform/4.12/security/certificates/updating-ca-bundle.html#ca-bundle-replacing_updating-ca-bundle).

In each case, Windows nodes can be joined to the cluster after altering proxy settings in which case proxy settings would
be applied initial during node configuration. In the latter 2 scenarios, Windows nodes may already be existing on the
cluster in which case they would need to be updated.

### Implementation Details/Notes/Constraints [optional]

Some infrastructures require instances to access certain endpoints to retreive their metadata for bootstrapping. In this
case, platform-specific logic could be introduced to inject additional `no-proxy` entries such as `169.254.169.254` and
`.${REGION}.compute.internal`. However, this may already be handled for us by components like the CNO.

### Risks and Mitigations

The risks and mitigations are similar to those on the [Linux side of the cluster-wide proxy](https://github.com/openshift/enhancements/blob/master/enhancements/proxy/global-cluster-egress-proxy.md#risks-and-mitigations).
Although cluster infra resources already do a best effort validation on the user-provided proxy URL schema and CAs,
a user could provide non-functional proxy settings/certs. This would be propogated to their Windows nodes and workloads,
taking down existing application connectivity and preventing new Windows nodes from being bootstrapped.

### Drawbacks

The only drawbacks are the increased complexity of WMCO and the potential complexity of debugging customer cases that
involve a proxy setup, since it would be extremely difficult to set up an accurate replicatation environment.
This can be mitigated by proactively getting the development team, QE, and support folks familiar with the expected
behavior of Windows nodes/workloads in proxied clusters, and comfortable spinning up their own proxied clusters.

### API Extensions

N/A, as no CRDs, admission and conversion webhooks, aggregated API servers, or finalizers will be added or modified.

### Operational Aspects of API Extensions

N/A

#### Failure Modes & Support Procedures

If either of the underlying mechanisms fail (those that publish proxy config to cluster resources or those that consume
these published config settings), Windows nodes will become proxy unaware. This could involve an issue with the cluster
network operator, OLM, WMCO, or the user-provided proxy settings. This would result in all future egress traffic
circumventing the proxy, which could affect inter-pod communication, exisitng application availablity, and security.
Also, the pause image may not be able to be fetched, preventing new Windows nodes from being bootstrapped. This would
require manual intervention from the cluster admin and/or a new release fixing whatever bug is causing the problem.

## Design Details

### Test Plan & Infrastructure Needed

In addition to unit testing individual WMCO packages and controllers, an e2e will be added to the WMCO repo for the
master/release-4.14 branches. We will leverage the [existing proxy workflow](https://github.com/openshift/release/tree/master/ci-operator/step-registry/ipi/conf/aws/proxy)
from the release repo and add a job to the WMCO repo's CI config to test with. We will run regression testing to make
sure the addition of the egress proxy feature does not break existing functionality. If we release a community offering
with this feature, we will need to add a CI job using a cluster-wide proxy on OKD.

### Open Questions [optional]

> 1. Can we use the existing [CI proxy job](https://github.com/openshift/release/tree/master/ci-operator/step-registry/ipi/conf/aws/proxy)
     to test an HTTPS proxy that requires custom certs to access, or can we only test HTTP and no-proxy using it? If the
     latter, we'll need to explore how to do this in EC2 and add improvments to the linked workflow.
>> 2. Is testing cluster-wide proxy in CI on __only AWS__ okay? This is consistent with what the Linux side does.
      QE may want to cover all platforms.
> 3. Do user-created Windows workloads need proxy variables set in their spec? Does this already happen? I don't think
     this should be WMCO's responsibility, so is this on another team or upon the user?

### Graduation Criteria

#### Dev Preview -> Tech Preview

Community WMCO 8.y.z or 9.y.z (or both) can be released with incremental additions to Windows proxy support
functionality, giving users an opportunity to get an early preview of the feature using OKD / OCP 4.13 or 4.14.
It will also allow us to collect feedback to troubleshoot common pain points and learn if there are any shortcomings.

#### Tech Preview -> GA

The feature associated with this enhacement is targeted to land in the offical Red Hat operator version of WMCO 9.0.0
within OpenShift 4.14 timeframe. The normal WMCO release process will be followed as the functionality described in this
enhancement is integrated into the product.

An Openshift docs update announcing Windows cluster-wide proxy support will be required as part of GA. The new docs
should list any Windows-specific info, but linking to existing docs should be enough for overarching proxy/PKI details.

#### Removing a deprecated feature

N/A, as this is a new feature that does not supercede an existing one

### Upgrade / Downgrade Strategy

The relevant upgrade path is from WMCO 8.y.z in OCP 4.13 to WMCO 9.y.z in OCP 4.14. There will be no changes to the
current WMCO upgrade strategy. Once customers are on WMCO 9.0.0:
* if they have an existing cluster-wide proxy configured, their Windows nodes will begin utilizing it for egress traffic
* otherwise, they can configure one and the Windows nodes will be automatically updated to use it by the operator

In both relevant OCP versions, OLM consumes global proxy settings and injects them into managed operators like WMCO.

Downgrades are generally [not supported by OLM](https://github.com/operator-framework/operator-lifecycle-manager/issues/1177),
which manages WMCO. In case of breaking changes, please see the
[WMCO Upgrades](https://github.com/openshift/enhancements/blob/master/enhancements/windows-containers/windows-machine-config-operator-upgrades.md#risks-and-mitigations)
enhancement document for guidance.

### Version Skew Strategy

N/A. There will be no version skew since this work is all within 1 product subcomponent (WMCO). The 8.y.z version of the
official Red Hat operator will not have cluster-wide egress proxy support for Windows enabled. Then, when customers move
to WMCO 9.0.0, the proxy support will be available.

## Implementation History

The implementation history can be tracked by following the associated work items in Jira and source code improvements in
the WMCO Github repo.

Planned GA of the feature: WMCO v9.0.0 in OCP 4.14

## Alternatives

* A workaround that would deliver the same value proposed by this enhancemnt would be to validate and provide guidance
  to make cluster administrators responsible for manually propogating proxy settings to each of their Windows nodes, and
  underlying OpenShift managed components. This is not a feasible alternative as even manual node changes can be
  ephemeral. WMCO would reset config changes to OpenShift managed Windows services in the event of a node reconcilition.
