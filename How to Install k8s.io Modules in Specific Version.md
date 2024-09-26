##### Prerequisite
- Delete the existing `go.sum` file
- Based on the Kubernetes version the cluster is running go the to https://github.com/kubernetes/kubernetes/blob/master/go.mod
	- Go to the branch corresponding to the Kubernetes version you are running
	- Search for the `go.opentelemetry.io/otel` module version. This is the compatible OpenTelemetry version with the Kubernetes version.
		- EXAMPLE: `go.opentelemetry.io/otel v0.20.0`

##### Procedure
- Create the base go.mod file and 
	- modify Y and Z to the desired Kubernetes version. For example, k8s version 1.25.12 will be Y=25, Z=12.
	- Use the `go.opentelemetry.io/otel` module version you acquired previously 
```
module node-e2e

go 1.23.1

require (
	github.com/onsi/ginkgo v1.16.5
	github.com/sirupsen/logrus v1.9.3
	k8s.io/api v0.Y.Z
	k8s.io/apimachinery v0.Y.Z
	k8s.io/client-go v0.Y.Z
	k8s.io/kubernetes v1.Y.Z
)

require go.opentelemetry.io/otel vA.B.C

replace (
	//
	// k8s.io/kubernetes depends on these k8s.io packages, but unversioned
	//
	k8s.io/api => k8s.io/api v0.Y.Z
	k8s.io/apiextensions-apiserver => k8s.io/apiextensions-apiserver v0.Y.Z
	k8s.io/apimachinery => k8s.io/apimachinery v0.Y.Z
	k8s.io/apiserver => k8s.io/apiserver v0.Y.Z
	k8s.io/cli-runtime => k8s.io/cli-runtime v0.Y.Z
	k8s.io/client-go => k8s.io/client-go v0.Y.Z
	k8s.io/cloud-provider => k8s.io/cloud-provider v0.Y.Z
	k8s.io/cluster-bootstrap => k8s.io/cluster-bootstrap v0.Y.Z
	k8s.io/code-generator => k8s.io/code-generator v0.Y.Z
	k8s.io/component-base => k8s.io/component-base v0.Y.Z
	k8s.io/component-helpers => k8s.io/component-helpers v0.Y.Z
	k8s.io/controller-manager => k8s.io/controller-manager v0.Y.Z
	k8s.io/cri-api => k8s.io/cri-api v0.Y.Z
	k8s.io/cri-client => k8s.io/cri-client v0.Y.Z
	k8s.io/csi-translation-lib => k8s.io/csi-translation-lib v0.Y.Z
	k8s.io/dynamic-resource-allocation => k8s.io/dynamic-resource-allocation v0.Y.Z
	k8s.io/endpointslice => k8s.io/endpointslice v0.Y.Z
	k8s.io/kube-aggregator => k8s.io/kube-aggregator v0.Y.Z
	k8s.io/kube-controller-manager => k8s.io/kube-controller-manager v0.Y.Z
	k8s.io/kube-proxy => k8s.io/kube-proxy v0.Y.Z
	k8s.io/kube-scheduler => k8s.io/kube-scheduler v0.Y.Z
	k8s.io/kubectl => k8s.io/kubectl v0.Y.Z
	k8s.io/kubelet => k8s.io/kubelet v0.Y.Z
	k8s.io/legacy-cloud-providers => k8s.io/legacy-cloud-providers v0.Y.Z
	k8s.io/metrics => k8s.io/metrics v0.Y.Z
	
	// TODO: replace with latest once https://github.com/ceph/ceph-csi/issues/4633 is fixed
	k8s.io/mount-utils => k8s.io/mount-utils v0.Y.Z
	k8s.io/pod-security-admission => k8s.io/pod-security-admission v0.Y.Z
	k8s.io/sample-apiserver => k8s.io/sample-apiserver v0.Y.Z
)
```
- Get the `go.opentelemetry.io/otel` module:
```
$ go get go.opentelemetry.io/otel vA.B.C
```
- Tidy:
```
$ go mod tidy
```

Make sure the `k8s.io` modules were not modified to a different version than the desired one.


##### References
- CephCSI GitHub page: https://github.com/ceph/ceph-csi/blob/devel/go.mod