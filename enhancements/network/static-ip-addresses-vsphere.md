---
title: static-ip-addresses-vsphere
authors:
  - rvanderp3
reviewers: 
  - JoelSpeed - machine API
  - elmiko - machine API
  - patrickdillon - installer
  - jcpowermac - vSphere
  - cybertron - networking
  - zaneb - networking
approvers:
  - JoelSpeed
  - patrickdillon
  - cybertron
api-approvers: 
  - JoelSpeed
creation-date: 2022-10-21
last-updated: 2022-11-01
tracking-link: 
- https://issues.redhat.com/browse/OCPPLAN-9654
see-also:
  - /enhancements/installer/vsphere-ipi-zonal.md
replaces:
superseded-by:
---

# Static IP Addresses for vSphere IPI

## Summary

Static IP addresses are emerging as a common requirement in environments where
the usage of DHCP violates corporate security guidelines.  Additionally, many 
users which require static IPs also require the use of the IPI installer. 
The proposal described in this enhacement discusses the implementation of
of assiging static IPs at both day 0 and day 2.

## Motivation

Users of OpenShift would like the ability to provision vSphere IPI clusters with static IPs.

- https://issues.redhat.com/browse/OCPPLAN-9654

### User Stories

As an OpenShift administrator, I want to provision nodes with static IP addresses so that I can comply with my organization's security requirements.

As an OpenShift administrator, I want to provision static IP addresses with the IPI installer so that I can reduce the complexity of certifying tools required to provision OpenShift.

As an OpenShift administrator, I want to scale nodes with static IPs so that I can meet capacity demands as well as respond to disaster recovery scenarios.

### Goals

- All nodes created during the installation are configured with static IPs.  
  Rationale: Many environments, due to security policies, do not allow DHCP.

- The IPI installation method is able to provide static IPs to the nodes
  Rationale: Some users must qualify each tool used in their environment. 
  Leveraging IPI greatly reduces the number of tools required to provision
  a cluster.

### Non-Goals

- OpenShift will not be responsible for managing IP addresses.

## Proposal

### Static IPs Configured at Installation

To faciliate the configuration of static IP address, network device configuration definitions are created for each node in the install-config.yaml. A `hosts`
slice will be introduced to the installer platform specification to allow network device configurations to be specified for a nodes.

For the bootstrap and control plane nodes, static IP configuration is passed to the node on the [kernel command line](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/networking_guide/sec-configuring_ip_networking_from_the_kernel_command_line) via the `guestinfo.afterburn.initrd.network-kargs` extraconfig parameter.  [Afterburn](https://github.com/coreos/afterburn/blob/main/src/providers/vmware/amd64.rs) recognizes this parameter when the node initially boots. 

When static network device configuration is required, Machines can not be created via MachineSets without an external controller fulfilling `IPAddressClaim`s. The installer will create the initial compute Machine manifests with the network configuration provided in the platform specification.

As with the installer, the vSphere [machine reconciler](https://github.com/openshift/machine-api-operator/blob/master/pkg/controller/vsphere/reconciler.go#L745-L755) 
will pass the static network device configuration via the `guestinfo.afterburn.initrd.network-kargs` extraconfig parameter.  


### Day 2 Static network device configuration

Machines with static IPs may be added after installation by either manually specifying the network
configuration in the providerSpec in a manually created `machine` resource, or by defining an `IPPool` in `addressesFromPool` in a `machineset` provider spec.

CAPI has introduced CRDs [IPAddressClaims](https://github.com/kubernetes-sigs/cluster-api/blob/main/config/crd/bases/ipam.cluster.x-k8s.io_ipaddressclaims.yaml) and [IPAddresses](https://github.com/kubernetes-sigs/cluster-api/blob/main/config/crd/bases/ipam.cluster.x-k8s.io_ipaddresses.yaml).  An implementation of IPAM support has merged in [CAPV](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/pull/1666).  The CAPV implementation relies upon experimental APIs but is available for use with workload clusters.

`machinesets` will be supported through the creation of an external controller([sample controller](https://github.com/rvanderp3/machine-ipam-controller)).  

#### Changes Required

##### Installer

1. Modify the `install-config.yaml` vSphere platform specification to support the definition of the 
~~~go

const (
	// IPAddressClaimedCondition documents the status of claiming an IP address
	// from an IPAM provider.
	IPAddressClaimedCondition clusterv1.ConditionType = "IPAddressClaimed"

	// WaitingForIPAddressReason (Severity=Info) documents that the VSphereVM is
	// currently waiting for an IP address to be provisioned.
	WaitingForIPAddressReason = "WaitingForIPAddress"

	// IPAddressInvalidReason (Severity=Error) documents that the IP address
	// provided by the IPAM provider is not valid.
	IPAddressInvalidReason = "IPAddressInvalid"

  // IPClaimFinalizer allows the reconciler to prevent deletion of an
	// IPAddressClaim that is in use.
	IPAddressClaimFinalizer = "vspherevm.infrastructure.cluster.x-k8s.io/ip-claim-protection"
)

// Hosts defines `Host` configurations to be applied to nodes deployed by the installer
type Hosts []Host

// Host defines the network device configuration to be applied for a node deployed by the installer
type Host struct {  
  // FailureDomain refers to the name of a FailureDomain as described in https://github.com/openshift/enhancements/blob/master/enhancements/installer/vsphere-ipi-zonal.md
  // +optional
  FailureDomain string `json: "failureDomain"`

  // Slice of NetworkDeviceSpecs to be applied
  // +kubebuilder:validation:Required
  NetworkDevice []NetworkDeviceSpec `json: "networkDevice"` 

  // Role defines the role of the node
  // +kubebuilder:validation:Enum="";bootstrap;control-plane;compute
  // +kubebuilder:validation:Required
  Role string `json: "role"`
}

type IPPoolRef struct {
  ApiGroup string `json: "apiGroup"`
  Kind string `json: "kind"`
  Name string `json: "name"`
}

type IPAddressClaim struct {
 	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

  PoolRef IPAMClusterIPPoolRef `json: "poolRef"`
}

type IPAddressClaimRef struct {
  Name string `json: "name"`
}

type IPAddress struct {
  Address string `json: "address"`

  ClaimRef IPAddressClaimRef `json: "claimRef"`
  
  Gateway string `json: "gateway"`

  PoolRef IPAMClusterIPPoolRef `json: "poolRef"`

  Prefix int64 `json: "prefix"`
}

// IPAMClusterIPPoolSpec defines the desired state of IPAMClusterIPPoolSpec.
type IPPoolSpec struct {
	// Subnet is the subnet to assign IP addresses from.
	// +optional
	Subnet string `json:"subnet,omitempty"`

  FailureDomain string `json:"failure-domain,omitempty"`
}

// IPPoolStatus defines the observed state of StubClusterIPPool.
type IPPoolStatus struct {
}

// IPAMClusterIPPool is the Schema for the StubClusterIPPool API.
type IPPool struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

  Spec   IPPoolSpec   `json:"spec,omitempty"`
	Status IPPoolStatus `json:"status,omitempty"`
}


// AddressesFromPool is an IPAddressPool that should be assigned to
// IPAddressClaims. The machine's cloud-init metadata will be populated with
// IPAddresses fulfilled by an IPAM provider.
type AddressesFromPool struct {  
  // APIGroup is the group for the resource
  // being referenced. If APIGroup is not
  // specified, the specified Kind must be in
  // the core API group. For any other
  // third-party types, APIGroup is required.
  ApiGroup string `json: "apiGroup"`
  
  // Kind is the type of resource being referenced
  Kind string `json: "kind"` 
  
  // Name is the name of resource being referenced
  Name string `json: "name"` 
}

type NetworkDeviceSpec struct {
	// gateway4 is the IPv4 gateway used by this device.
	// Required when DHCP4 is false.
	// +optional
	// +kubebuilder:validation:Format=ipv4
	Gateway4 string `json:"gateway4,omitempty"`

	// gateway4 is the IPv4 gateway used by this device.
	// Required when DHCP6 is false.
	// +kubebuilder:validation:Format=ipv6
	// +optional
	Gateway6 string `json:"gateway6,omitempty"`

	// ipaddrs is a list of one or more IPv4 and/or IPv6 addresses to assign
	// to this device.
	// Required when DHCP4 and DHCP6 are both Disabled.
	// + Validation is applied via a patch, we validate the format as either ipv4 or ipv6
	// +optional
	IPAddrs []string `json:"ipAddrs,omitempty"`

	// nameservers is a list of IPv4 and/or IPv6 addresses used as DNS
	// nameservers.
	// Please note that Linux allows only three nameservers (https://linux.die.net/man/5/resolv.conf).
	// +optional
	Nameservers []string `json:"nameservers,omitempty"`
  
  // addressesFromPools is a list of address pools which will fulfill requests for static IPs
  AddressesFromPools []AddressesFromPool `json:"addressesFromPool,omitempty"`
}

~~~

Example of a platform spec configured to provide static IPs for the bootstrap, control plane, and compute nodes:
~~~yaml
platform:
  vsphere:
    hosts:
    - role: bootstrap
      networkDevice:
        ipAddrs:
        - 192.168.101.240/24
        gateway4: 192.168.101.1
        nameservers:
        - 192.168.101.2
    - role: control-plane
      failureDomain: us-east-1a
      networkDevice:
        ipAddrs:
        - 192.168.101.241/24
        gateway4: 192.168.101.1
        nameservers:
        - 192.168.101.2
    - role: control-plane
      failureDomain: us-east-1b
      networkDevice:
        ipAddrs:
        - 192.168.101.242/24
        gateway4: 192.168.101.1
        nameservers:
        - 192.168.101.2
    - role: control-plane
      failureDomain: us-east-1c
      networkDevice:
        ipAddrs:
        - 192.168.101.243/24
        gateway4: 192.168.101.1
        nameservers:
        - 192.168.101.2
    - role: compute
      networkDevice:
        ipAddrs:
        - 192.168.101.244/24
        gateway4: 192.168.101.1
        nameservers:
        - 192.168.101.2
    - role: compute
      networkDevice:
        ipAddrs:
        - 192.168.101.245/24
        gateway4: 192.168.101.1
        nameservers:
        - 192.168.101.2
    - role: compute
      networkDevice:
        ipAddrs:
        - 192.168.101.246/24
        gateway4: 192.168.101.1
        nameservers:
        - 192.168.101.2
~~~
2. Add validation for the modified/added fields in the platform specification.
3. For compute nodes, produce machine manifests with associated network device configuration.  

Example of `machine` configured with network device configuration
~~~yaml
apiVersion: machine.openshift.io/v1beta1
kind: Machine
metadata:
  name: test-compute-1
spec:
  metadata: {}
  providerSpec:
    value:
      numCoresPerSocket: 2
      diskGiB: 60
      snapshot: ''
      userDataSecret:
        name: worker-user-data
      memoryMiB: 8192
      credentialsSecret:
        name: vsphere-cloud-credentials
      network:
        devices:
          - networkName: lab
            ipAddrs:
            - 192.168.101.244/24
            gateway4: 192.168.101.1
            nameservers:
            - 192.168.101.2            
      metadata:
        creationTimestamp: null
      numCPUs: 2      
      kind: VSphereMachineProviderSpec
      workspace:
        datacenter: testdc
        datastore: datastore-1
        folder: /testdc/vm/cluster-folder
        resourcePool: /testdc/host/cluster1/Resources
        server: test.vcenter.net
      template: vm-template-rhcos
      apiVersion: machine.openshift.io/v1beta1
~~~
4. For bootstrap and control plane nodes, modify vSphere terraform to convert network device configuration to a VM guestinfo parameter
for each VM to be created.

As the assets are generated for the control plane and compute nodes, the slice of `host`s for each 
node role will be used to populate network device configuration.  The number of `host`s must match the number of
replicas defined in the associated machine pool.

Additionally, each defined host may optionally define a failure domain.  This indicates that the associated `networkDevice` will be applied to a machine created in the indicated failure domain.


##### Machine API
- Modify vSphere machine controller to convert IP configuration to VM guestinfo parameter
- Introduce new types to facilitate IP allocation by a controller.
- Modify [types_vsphereprovider.go](https://github.com/openshift/api/blob/master/machine/v1beta1/types_vsphereprovider.go) to support network device configuration. 


###### network device configuration of Machines

IP configuration for a given network device may be derived from three configuration mechanisms:
1. DHCP
2. An external IPAM IP Pool
3. Static IP configuration defined in the provider spec

The machine API `VSphereMachineProviderSpec.Network` will be extended to include a subset of additional properties as defined in https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/blob/main/apis/v1beta1/types.go.  See [openshift/api#1338](https://github.com/openshift/api/pull/1338) for further details on the API extension to the provider specification.  


### Workflow Description

#### Installation
1. OpenShift administrator reserves IP addresses for installation.
2. OpenShift administrator constructs `install-config.yaml` to define an network device configuration for each node that will receive a static IP address.
3. OpenShift administrator initiates an installation with `openshift-install create cluster`.  
4. The installer will proceed to:
- provision bootstrap and control plane nodes with the specified network device configuration
- create machine resources containing specified network device configuration
5. Once the machine API controllers become active, the compute machine resources will be rendered with the specified network device configuration.

#### Scaling new Nodes without `machinesets`
1. OpenShift administrator reserves IP addresses for new nodes to be scaled up.
2. OpenShift administrator constructs machine resource to define a network device configuration for each new node that will receive a static IP address.
3. OpenShift administrator initiates the creation of new machines by running `oc create -f machine.yaml`.  
4. The machine API will render the nodes with the specified network device configuration.

#### Scaling new Nodes with `machinesets`
1. OpenShift administrator configures a machineset with `addressesFromPools` defined in the platform specification.

Example:
~~~yaml
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  name: static-machineset-worker
  namespace: openshift-machine-api
  labels:
    machine.openshift.io/cluster-api-cluster: cluster
spec:
  replicas: 0
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: cluster
      machine.openshift.io/cluster-api-machineset: static-machineset-worker
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: cluster
        machine.openshift.io/cluster-api-machine-role: worker
        machine.openshift.io/cluster-api-machine-type: worker
        machine.openshift.io/cluster-api-machineset: static-machineset-worker
    spec:          
      cloneMode: linkedClone
      datacenter: dc0
      datastore: sharedVmfs-0
      diskGiB: 25
      folder: folder0
      memoryMiB: 8192
      network:
        devices:
        - networkName: port-group-vlan-101          
          dhcp4: false                          
          addressesFromPools:                   
          - apiGroup: machine.openshift.io
            kind: IPPool
            name: example-pool
~~~

2. OpenShift administrator or machine autoscaler scales `n` machines
3. If `addressesFromPools` contains a `AddressesFromPool` definition, the machine controller will create an `IPAddressClaim`. The `IPAddressClaim` will:
- Set a finalizer on the `IPAddressClaim` called `ipam.cluster.x-k8s.io/ip-claim-protection`
- Set an owner reference to the associated `Machine`
- Set the status of the associated `Machine`
- Block the creation of the underlying machine in the infrastructure until all associated `IPAddressClaim`s are bound
4. An external controller will watch for `IPAddressClaim`s and obtain an IP address in accordance with the `AddressesFromPool` specification.
5. Once obtained, the external controller will create an `IPAddress` and bind it to its associated `IPAddressClaim`.
6. The machine controller will then create the virtual machine with the network configuration in the network device spec and the `IPAddress`.

~~~mermaid
sequenceDiagram
    machineset controller->>+machine: creates machine with<br> IPPool
    machine controller-->machine controller: create IPAddressClaim<br>and wait for claim<br>to be bound
    machine controller-->machine controller: IPAddressClaim ownerReference<br>refers to the machine
    IP controller-->IP controller: processes claim and<br>allocates IP address    
    IP controller-->IP controller: create IPAddress and bind IPAddressClaim
    machine controller-->machine controller: build guestinfo.afterburn.initrd.network-kargs<br>and clone VM
~~~

On scale down:
1. The machine controller will remove the finalizer on `IPAddressClaim` associated with a given `Machine` after the underlying virtual machine has been deleted.
2. The kubernetes API will garbage collect the `IPAddressClaim` and `IPAddress` formerly associated with the `Machine`.
3. Once the `IPAddress` associated with the machine is deleted, the external controller can reuse the address.

In this workflow, the controller is responsible for managing, claiming, and releasing IP addresses.  

A sample project [machine-ipam-controller](https://github.com/rvanderp3/machine-ipam-controller) is an example of a controller that implements this workflow.


#### Variation [optional]

### API Extensions

3 additional CRDs will be introduced which to `machine.openshift.io`.

- `ippool.machine.openshift.io` - IP pool which is fulfilled by an external controller
- `ipaddressclaim.machine.openshift.io` - IP address claim request which is created by the machine reconciler and fulfilled by an external controller
- `ipaddress.machine.openshift.io` - IP address fulfilled by an external controller

The CRDs `machines.machine.openshift.io` and `machinesets.machine.openshift.io` will be modified to allow the definition of `addressesFromPool` in the provider specification.

### Implementation Details/Notes/Constraints 

#### Eventual Migration to CAPI/CAPV

The definition of the CRDs above is intended to follow a similar pattern followed by [CAPV](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/pull/1210/files).  Migration would consist of migrating the `ipaddressclaim.machine.openshift.io` and `ipaddress.machine.openshift.io` CRDs to the analogous CAPI CRDs.

### Risks and Mitigations

### Drawbacks

- Scaling nodes will become more complex. This will require the OpenShift administrator to integrate network device configuration
  management to enable scaling of machine API machine resources.

- If a `machineset` is configured to specify an IPPool resource.  An external controller is responsible for fulfilling the resultant `IPAddressClaim` that is created during machine rendering.

- `install-config.yaml` will grow in complexity.

## Design Details





### Open Questions

#### `nmstate` API
Q: How should we introduce `nmstate` to the OpenShift API?  While we only need a subset of `nmstate` for this enhancement, `nmstate` may have broader applicability outside of vSphere.

A: In the November 10, 2022 cluster lifecycle arch call, it was decided to move to an [API consistent with CAPV](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/blob/main/apis/v1beta1/types.go).

### Test Plan

### Graduation Criteria

#### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers
- Enumerate service level indicators (SLIs), expose SLIs as metrics
- Write symptoms-based alerts for the component(s)

#### Tech Preview -> GA

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default
- Backhaul SLI telemetry
- Document SLOs for the component
- Conduct load testing
- User facing documentation created in [openshift-docs](https://github.com/openshift/openshift-docs/)

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

#### Removing a deprecated feature

### Upgrade / Downgrade Strategy

### Version Skew Strategy

### Operational Aspects of API Extensions

#### Failure Modes

#### Support Procedures

## Implementation History

## Alternatives

Lifecycle hook 

## Infrastructure Needed [optional]
