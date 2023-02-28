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
last-updated: 2023-02-28
tracking-link:
  - Feature: "https://issues.redhat.com/browse/OCPBU-22"
  - Epic: "https://issues.redhat.com/browse/WINC-802"
see-also:
  - "https://github.com/openshift/enhancements/blob/master/enhancements/proxy/global-cluster-egress-proxy.md"
  - Cluster-wide Egress Proxy Initiative doc: "https://docs.google.com/document/d/12bBF7GTgscW8B3apVU2WagtQpWh-PWk2tOWiD0dsdfU/edit"
  - Proxy Bootstrap Workflow: "https://docs.google.com/document/d/1y0t0yEOSnKc4abxsjxEQjrFa1AP8iHcGyxlBpqGLO08/edit#heading=h.y6ieif41wmlc"
  - Operator Proxy Support Dev Workflow: "https://docs.google.com/document/d/1otp9v5KkoOgq5vhN7ieBpdFhRIIy6z2yuaS_0rjm6wQ/edit#heading=h.dzcnuz4qv07o"
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
protocol for publishing [cluster-wide proxy](https://docs.openshift.com/container-platform/4.12/networking/enable-cluster-wide-proxy.html) 
settings, which is consumed by different OpenShift components (Linux worker nodes and infra nodes, CVO and OLM managed
operators) but Windows worker nodes do not currently consume or respect proxy settings. This effort will work to plug
feature disparity by making the [Windows Machine Config Operator](https://github.com/openshift/windows-machine-config-operator)
(WMCO) proxy aware at install time and reactive during runtime.

## Motivation

The motivation here is to expand the Windows containers production use case, enabling users to add Windows nodes and run
workloads easily and successfully in a proxied cluster. This is an extremely important ask for customer environments
where Windows nodes are often tasked with pulling images private registries secured behind the client's proxy server, or
making requests to off-cluster services, and those that use a [custom public key infrastructure](https://docs.openshift.com/container-platform/4.12/networking/configuring-a-custom-pki.html).

### Goals

* Create an automated mechanism for WMCO to consume global egress proxy config from existing platform recources.
  Includes:
  + [proxy connection information](https://docs.openshift.com/container-platform/4.12//rest_api/config_apis/proxy-config-openshift-io-v1.html#spec)
    * http/https URLs
    + list of networks that should bypass the proxy (no-proxy hostnames, domains, IP addresses or other network CIDRs)
  + [additional certificate authorities](https://docs.openshift.com/container-platform/4.12//rest_api/config_apis/proxy-config-openshift-io-v1.html#spec-trustedca) required to validate the proxy's certificate
* Configure the proxy settings in WMCO-managed components on Windows nodes (kubelet, containerd runtime)
* React to changes to the cluster-wide proxy settings during cluster runtime
* Maintain normal functionality in non-proxied clusters

### Non-Goals

* First-class support/enablement of proxy utilization for user-provided applications
* Anything to do with *ingress* or reverse proxy settings is out of scope
* Monitor cert expiration dates or automatically replace expired CAs in the cluster's trust bundle

## Proposal

There are two major undertakings:
- Adding proxy environment variables (`NO_PROXY`, `HTTP_PROXY`, and `HTTPS_PROXY`) to Windows nodes and WMCO-managed Windows services.
- Adding the proxy’s trusted CA certificate bundle to Windows nodes' local trust store.

Since WMCO is a day 2 operator, it will pick up proxy settings during runtime regardless of if proxy settings were set
during cluster install time or at some point during the cluster's lifetime. When global proxy settings are updated, WMCO will react by:
- overriding proxy vars on the instance with the new values
- copying over the new trust bundle to Windows instances (old one should be removed) and updating each instance's local trust store

All changes detailed in this enhacement proposal will be limited to the Windows Machine Config Operator.

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
The latter 2 scenarios occur through changing the [`Proxy` object named `cluster`](https://docs.openshift.com/container-platform/4.12/networking/enable-cluster-wide-proxy.html#nw-proxy-configure-object_config-cluster-wide-proxy)
or by modifying certificates present in their [trustedCA ConfigMap](https://docs.openshift.com/container-platform/4.12/security/certificates/updating-ca-bundle.html#ca-bundle-replacing_updating-ca-bundle).

In each case, Windows nodes can be joined to the cluster after altering proxy settings in which case proxy settings would
be applied initial during node configuration. In the latter 2 scenarios, Windows nodes may already be existing on the
cluster in which case they would need to be updated.

### Risks, Drawbacks, and Mitigations

The risks and mitigations are similar to those on the [Linux side of the cluster-wide proxy](https://github.com/openshift/enhancements/blob/master/enhancements/proxy/global-cluster-egress-proxy.md#risks-and-mitigations).
Although cluster infra resources already do a best effort validation on the user-provided proxy URL schema and CAs,
a user could provide non-functional proxy settings/certs. This would be propogated to their Windows nodes and workloads,
taking down existing application connectivity and preventing new Windows nodes from being bootstrapped.

The only drawbacks are the increased complexity of WMCO and the potential complexity of debugging customer cases that
involve a proxy setup, since it would be extremely difficult to set up an accurate replicatation environment.
This can be mitigated by proactively getting the development team, QE, and support folks familiar with the expected
behavior of Windows nodes/workloads in proxied clusters, and comfortable spinning up their own proxied clusters.

### API Extensions

N/A, as no CRDs, admission and conversion webhooks, aggregated API servers, or finalizers will be added or modified.
Only the WMCO will be extended which is an optional operator with its own lifecycle and SLO/SLAs, a 
[tier 3 OpenShift API](https://docs.openshift.com/container-platform/4.12/rest_api/understanding-api-support-tiers.html#api-tiers_understanding-api-tiers).

### Operational Aspects of API Extensions

N/A

#### Failure Modes & Support Procedures

In general the support procedures for WMCO will remain the same. There are two underlying mechanisms we rely on, the
publishing of proxy config to cluster resources and the comsuming of the published config. If either of the underlying
mechanisms fail, Windows nodes will become proxy unaware. This could involve an issue with the user-provided proxy
settings, the cluster network operator, OLM, or WMCO. This would result in all future egress traffic circumventing the
proxy, which could affect inter-pod communication, existing application availablity, and security. Also, the pause image
may not be able to be fetched, preventing new Windows nodes from being bootstrapped. This would require manual
intervention from the cluster admin or a new release fixing whatever bug is causing the problem.

## Design Details

### Configuring Proxy Environment Variables

As it stands today, the source of truth for cluster-wide proxy settings is the `Proxy` resource with the name `cluster`.
The contents of the resource are both user-defined, as well as adjusted by the cluster network operator (CNO). Some
platforms require instances to access certain endpoints to retreive metadata for bootstrapping. CNO has logic to inject
additional `no-proxy` entries such as `169.254.169.254` and `.${REGION}.compute.internal` into the `Proxy` resource.

OLM is a subscriber to these `Proxy` settings -- it forwards the settings to the CSV of managed operators, so the WMCO
container will automatically get the required `NO_PROXY`, `HTTP_PROXY`, and `HTTPS_PROXY` environment variables on startup.
In fact, OLM will update and restart the operator pod with proper environment variables if the `Proxy` resource changes.

On the Linux side this is achieved through `MachineConfigs` -- when global proxy settings are changed, the ignition
config is re-rendered by MCO (for whom proxy info is injected by CVO) to include a systemd unit 
`EnvironmentFile=/etc/mco/proxy.env`, with proxy vars set in this file.

WMCO will retrieve the proxy variables by watching the `rendered-worker` `MachineConfig` for changes and parse info from
the `proxy.env` file. This could be an expensive operation, but since we already parse ignition from the `MachineConfig`
as part of node bootstrap, the technical cost and complexity is reduced.

Then, once it has the required values, the operator will set the 3 proxy environment variables on the Windows instance
during node configuration/reconciling through Powershell. Since Windows services read environment from the system 
registry and the registry does not update until a shutdown/startup cycle, this may require rebooting. 
```powershell
$Env:<variable-name> = "<new-value>"
```
We may have to separately enable [Powershell to use the proxy vars](https://martin.hoppenheit.info/blog/2015/using-powershell-behind-proxy/).


### Configuring Custom Trusted Certificates

1. Add the trusted CA injection request ConfigMap to WMCO's manifests directory. 
   This is a resource that will be updated by CNO based on the content of the global `Proxy` resource.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
    release.openshift.io/create-only: "true"
  labels:
    config.openshift.io/inject-trusted-cabundle: "true"
  name: trusted-ca
  namespace: openshift-windows-machine-config-operator
```

2. Read the trusted CA ConfigMap data and use it to update the local trust store of Windows nodes during configuration.

3. Reconcile when the custom CA bundle changes. This will be done through a Kubernetes controller that watches
   the `trusted-ca` ConfigMap for create/update/delete events. On change, copy the new trust bundle to Windows instances,
   deleting old certificates (i.e. not present in the current trust bundle) off the instance and importing new ones.

How-to references:
* [import cert via powershell](https://docs.microsoft.com/en-us/powershell/module/pki/import-certificate?view=windowsserver2019-ps)
* [delete cert via powershell](https://stackoverflow.com/questions/37228851/delete-certificate-from-computer-store)

---

### Test Plan & Infrastructure Needed

In addition to unit testing individual WMCO packages and controllers, an e2e job will be added to the release repo for
WMCO's master/release-4.14 branches. This can replace the current vSphere job to avoid growing the number of required jobs.

We create a new workflow to test on vSphere. The [indivudial steps](https://github.com/openshift/release/tree/master/ci-operator/step-registry/ipi/conf/vsphere/proxy/https) 
are already present in the release repo and used by other jobs; we will string them together as needed to satisfy all
our testing requirements (OVN and HTTPS proxy secured by additional certs). The workflow will have the following steps:
```yaml
Pre:
- ipi-conf-vsphere
- ipi-conf-vsphere-proxy-https
- ovn-conf
- ovn-conf-hybrid-manifest-with-custom-vxlan-port
- ipi-install-vsphere
Test:
- windows-e2e-operator-test-with-custom-vxlan-port
Post:
- gather-network
- gather-proxy
- ipi-vsphere-post
```

We will run regression testing to make sure the addition of the egress proxy feature does not break existing functionality.
When we release a community  offering with this feature, we will add a similar CI job using a cluster-wide proxy on OKD.
QE may want to cover all platforms though when validating this feature.

### Graduation Criteria

A community version of WMCO 8 or 9 will be released with incremental additions to Windows proxy support
functionality, giving users an opportunity to get an early preview of the feature using OKD/OCP 4.13 or 4.14.
It will also allow us to collect feedback to troubleshoot common pain points and learn if there are any shortcomings.

The feature associated with this enhacement is targeted to land in the offical Red Hat operator version of WMCO 9.0.0
within OpenShift 4.14 timeframe. The normal WMCO release process will be followed as the functionality described in this
enhancement is integrated into the product.

An Openshift docs update announcing Windows cluster-wide proxy support will be required as part of GA. The new docs
should list any Windows-specific info, but linking to existing docs should be enough for overarching proxy/PKI details.

#### Removing a deprecated feature

N/A, as this is a new feature that does not supercede an existing one

### Upgrade / Downgrade Strategy

The relevant upgrade path is from WMCO 8.y.z in OCP 4.13 to WMCO 9.y.z in OCP 4.14. There will be no changes to the
current WMCO upgrade strategy. Once customers are on WMCO 9.0.0, they can configure one and the Windows nodes will be
automatically updated by the operator to use the `Proxy` settings for egress traffic.

When deconfiguring Windows instances, proxy settings will be cleared from the node. This involves undoing some node
config steps i.e. unsetting proxy variables and deleting additional certificates from the machine's local trust store.
This scenario will occur when upgrading both BYOH and Machine-backed Windows nodes.

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

## Alternatives & Justification

* A workaround that would deliver the same value proposed by this enhancement would be to validate and provide guidance
  to make cluster administrators responsible for manually propogating proxy settings to each of their Windows nodes, and
  underlying OpenShift managed components. This is not a feasible alternative as even manual node changes can be
  ephemeral. WMCO would reset config changes to OpenShift managed Windows services in the event of a node reconcilition.
* There is another possible way for WMCO to [retrieve the proxy variables](#configuring-proxy-environment-variables). WMCO
  can read the environment variables from its own pod spec (vars are injected + updated by OLM). The difficulty of this
  approach comes from figuring out when we need to update node's env vars. Ideally such reconfiguring happens only when
  the values change in the pod spec, but how can we detect if the proxy env vars changed or the operator just restarted 
  for some other reason? We want to avoid kicking off reconfigurations of all nodes every time the operator restarts.
* In order to avoid a system reboot after setting node environment variables, we can reconcile services by setting their
  environment variables, and then restarting the services. These changes are ephemeral, though, if the instance reboots.
  ```powershell
  [string[]] $envVars = @("HTTP_PROXY=http://<username>:<pswd>@<ip>:<port>", "NO_PROXY=123.example.com,10.88.0.0/16")
  Set-ItemProperty HKLM:SYSTEM\CurrentControlSet\Services\<$SERVICE_NAME> -Name Environment -Value $envVars
  Restart-Service <$SERVICE_NAME>
  ```
* Also note that there is another way to get the [trusted CA data](#configuring-custom-trusted-certififactes) required
  rather than accessing the ConfigMap directly, but I prefer not to follow it. It leaves open the same concern about
  unnecessary reconciliations -- how to detect if the operator restart was due to a trust bundle file change or the pod
  just restarted for another reason? For completeness, I will list it out the apporach here.
  * Update the operator’s Deployment to support trusted CA injection by mounting the trusted CA ConfigMap
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: windows-machine-config-operator
    namespace: openshift-windows-machine-config-operator
    annotations:
      config.openshift.io/inject-proxy: windows-machine-config-operator
  spec:
      ...
        containers:
          - name: windows-machine-config-operator
            volumeMounts:
            - name: trusted-ca
              mountPath: /etc/pki/ca-trust/extracted/pem
              readOnly: true
        - name: trusted-ca
          configMap:
            name: trusted-ca
            items:
              - key: ca-bundle.crt
                path: tls-ca-bundle.pem
  ...
  ```
  * Create a file watcher that watches changes to the mounted trust bundle that kills the main operator process and
    allows the backing k8s Deployment to start a new Pod that mounts the updated trust bundle. 
    Implementation example: [cluster-ingress-operator](https://github.com/openshift/cluster-ingress-operator/pull/334)
* Instead of adding a new vSphere job, we can leverage an [existing proxy test workflow on AWS](https://steps.ci.openshift.org/workflow/openshift-e2e-aws-proxy).
  However, this workflow does not test an HTTPs proxy requiring an additional trust bundle, so we would need to make 
  improvements to the pre- steps. Since vSphere is our most used platform, I'd rather test this on vSphere, which 
  already has the required config steps anyway.
