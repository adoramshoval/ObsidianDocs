#### Uninstall
- Delete the subscription
- Delete the CSV
- Delete the `nvidia-gpu-oprator` namespace
- Delete the ClusterPolicies.nvidia.com `clusterpolicies.nvidia.com gpu-cluster-policy`
- Drain and reboot the node in order to completely remove the nvidia driver
	- `lsmod | grep -i nvidia`

#### Installation
- Create cluster policy and specify version v23.3 in the subscription: https://gitlab.med.one/compute/ocpbm-cluster-config/-/merge_requests/335/diffs
- Sync namespace
- Sync operator group
- Sync subscription
- InstallPlan should be created - approve it - make sure the install plan wants to install the desired version
```
$ oc patch installplans.operators.coreos.com install-n5728 --type merge --patch '{"spec": {"approved":true}}'
```
- Wait for CSV to get in Success state
- Watch how the gpu-operator pod gets created inside the nvidia-gpu-operator
```
$ oc get nodes -o json -w | jq '.metadata | select(.labels."nvidia.com/gpu.present"?) | select(.labels."nvidia.com/gpu-driver-upgrade-state"?) | "Node name: \(.name), Upgrade state: \(.labels."nvidia.com/gpu-driver-upgrade-state"), Availability zone: \(.labels."topology.kubernetes.io/zone")"' -r
```
- Create the ClusterPolicy
- Watch how pods get deployed

- Delete the NodeFeatureDiscovery instance
- Remove labels related to NFD from GPU nodes

```
$ oc patch clusterpolicies.nvidia.com gpu-cluster-policy --type=merge -p '{"spec":{"driver":{"image":"nvcr.io/nvidia/driver@sha256:7c2df95df9ed4d16ad3b3c84079ccdd161f3639527ac1d90b106217f9f0a3aad"}}}' -n nvidia-gpu-operator
```

- p28  from deep dive - `devicePlugin` setting failed us ...

#### Upgrade

- Create a MR for subscription update and ClusterPolicy update with latest compatible driver image: https://gitlab.med.one/compute/ocpbm-cluster-config/-/merge_requests/343/diffs
- Driver version: `nvcr.io/nvidia/driver@sha256:351578cd0e4b94cc7738556e4ddcc2d70634414e089934d5e9d754bd868fb395`
- Sync the subscription - wait for install plan creation
- approve the install plan
- Watch the CSV enter success state
- Let the gpu-operator pod restart
- Let the GPU operator upgrade the driver version to the basic compatible version for 23.9 - it does it automatically - the operator will cordon nodes along the way automatically. 
- Wait for cordon to go down.
- Check with nvidia-smi the new installed driver version.
- Sync the cluster policy with the new driver image
- Watch for upgrade instructions through the node label
```
$ oc get nodes -o json -w | jq '.metadata | select(.labels."nvidia.com/gpu.present"?) | select(.labels."nvidia.com/gpu-driver-upgrade-state"?) | "Node name: \(.name), Upgrade state: \(.labels."nvidia.com/gpu-driver-upgrade-state"), Availability zone: \(.labels."topology.kubernetes.io/zone")"' -r
```

##### Testing driver upgrade 

![[Pasted image 20240924161428.png]]
- Sync cluster policy
```
  Normal  NodeNotSchedulable         103m (x3 over 4h23m)   kubelet              Node phmowrk-166018-14.medone-1.med.one status is now: NodeNotSchedulable
  Normal  Starting                   98m                    kubelet              Starting kubelet.
  Normal  NodeAllocatableEnforced    98m                    kubelet              Updated Node Allocatable limit across pods
  Normal  NodeHasSufficientPID       98m (x7 over 98m)      kubelet              Node phmowrk-166018-14.medone-1.med.one status is now: NodeHasSufficientPID
  Normal  NodeHasSufficientMemory    98m (x8 over 98m)      kubelet              Node phmowrk-166018-14.medone-1.med.one status is now: NodeHasSufficientMemory
  Normal  NodeHasNoDiskPressure      98m (x8 over 98m)      kubelet              Node phmowrk-166018-14.medone-1.med.one status is now: NodeHasNoDiskPressure
  Normal  ConfigDriftMonitorStarted  98m                    machineconfigdaemon  Config Drift Monitor started, watching against rendered-compute-az-b-6b7ea523ccb90ee16277804afcff3dc8
  Normal  GPUDriverUpgrade           90m                    nvidia-gpu-operator  Successfully updated node state label to [upgrade-done]
  Normal  GPUDriverUpgrade           30m                    nvidia-gpu-operator  Successfully updated node state label to [validation-required]
  Normal  GPUDriverUpgrade           30m                    nvidia-gpu-operator  Successfully updated node annotation to [nvidia.com/gpu-driver-upgrade-validation-start-time 1727183014]=%!s(MISSING)
  Normal  GPUDriverUpgrade           20m                    nvidia-gpu-operator  Successfully updated node state label to [upgrade-failed]
  Normal  GPUDriverUpgrade           20m                    nvidia-gpu-operator  Successfully updated node annotation to [nvidia.com/gpu-driver-upgrade-validation-start-time null]=%!s(MISSING)
  Normal  GPUDriverUpgrade           20m                    nvidia-gpu-operator  Successfully updated node state label to [uncordon-required]
  Normal  GPUDriverUpgrade           20m                    nvidia-gpu-operator  Successfully updated node state label to [upgrade-done]
  Normal  NodeSchedulable            20m (x3 over 98m)      kubelet              Node phmowrk-166018-14.medone-1.med.one status is now: NodeSchedulable
  Normal  GPUDriverUpgrade           18m (x2 over 34m)      nvidia-gpu-operator  Successfully updated node state label to [pod-restart-required]
  Normal  GPUDriverUpgrade           18m (x2 over 34m)      nvidia-gpu-operator  Successfully updated node state label to [wait-for-jobs-required]
  Normal  GPUDriverUpgrade           18m (x2 over 34m)      nvidia-gpu-operator  Successfully updated node state label to [upgrade-required]
  Normal  GPUDriverUpgrade           18m (x2 over 34m)      nvidia-gpu-operator  Successfully updated node state label to [cordon-required]
  Normal  GPUDriverUpgrade           18m (x2 over 34m)      nvidia-gpu-operator  Successfully updated node state label to [pod-deletion-required]
  Normal  NodeNotSchedulable         18m (x4 over 98m)      kubelet              Node phmowrk-166018-14.medone-1.med.one status is now: NodeNotSchedulable
  Normal  GPUDriverUpgrade           14m (x2 over 14m)      nvidia-gpu-operator  (combined from similar events): Successfully updated node annotation to [nvidia.com/gpu-driver-upgrade-validation-start-time 1727183986]=%!s(MISSING)
  Normal  NodeNotReady               8m46s (x21 over 110d)  node-controller      Node phmowrk-166018-14.medone-1.med.one status is now: NodeNotReady
  Normal  Starting                   4m47s                  kubelet              Starting kubelet.
  Normal  NodeAllocatableEnforced    4m47s                  kubelet              Updated Node Allocatable limit across pods
  Normal  NodeHasSufficientPID       4m46s (x7 over 4m47s)  kubelet              Node phmowrk-166018-14.medone-1.med.one status is now: NodeHasSufficientPID
  Normal  NodeHasSufficientMemory    4m45s (x8 over 4m47s)  kubelet              Node phmowrk-166018-14.medone-1.med.one status is now: NodeHasSufficientMemory
  Normal  NodeHasNoDiskPressure      4m45s (x8 over 4m47s)  kubelet              Node phmowrk-166018-14.medone-1.med.one status is now: NodeHasNoDiskPressure
  Normal  NodeUnresponsive           4m40s (x16 over 62d)   node-controller      virt-handler is not responsive, marking node as unresponsive
  Normal  ConfigDriftMonitorStarted  3m27s                  machineconfigdaemon  Config Drift Monitor started, watching against rendered-compute-az-b-6b7ea523ccb90ee16277804afcff3dc8
  Normal  GPUDriverUpgrade           59s                    nvidia-gpu-operator  Successfully updated node annotation to [nvidia.com/gpu-driver-upgrade-validation-start-time null]=%!s(MISSING)
  Normal  GPUDriverUpgrade           59s                    nvidia-gpu-operator  Successfully updated node state label to [upgrade-failed]
```

```
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: cordon-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: cordon-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: wait-for-jobs-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-deletion-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-deletion-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-failed, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-failed, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: uncordon-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: uncordon-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: cordon-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: cordon-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: wait-for-jobs-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-deletion-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: pod-restart-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: validation-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-failed, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-failed, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-failed, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-failed, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-failed, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-failed, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-failed, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: uncordon-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: uncordon-required, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
Node name: phmowrk-166018-14.medone-1.med.one, Upgrade state: upgrade-done, Availability zone: az-b
```

- Node went through a reboot for some reason ...
	- IMPORTANT - add a step for node drain before starting upgrade procedure
	- This is bad as it requires as to potentially drain passthrough nodes with VMs on them 
	- Where will GPU workload deploy while all GPU nodes are cordoned.

- Upgrade failed - gpu-feature-discovery could not get deployed
- Trying to upgrade to recommended driver version `550.54.15-rhcos4.12`

#### Common Errors
##### From `nvidia-dcgm-exporter`
```
  Warning  Failed     2m5s                 kubelet            Error: container create failed: time="2024-09-26T09:22:02Z" level=error msg="runc create failed: unable to start container process: error during container init: error running hook #0: error running hook: exit status 1, stdout: , stderr: nvidia-container-cli.real: initialization error: driver rpc error: timed out\n"
```

Usually NFD related
##### From `nvidia-device-plugin-validator`
```
 Warning  UnexpectedAdmissionError  3m25s  kubelet  Allocate failed due to device plugin GetPreferredAllocation rpc failed with err: rpc error: code = Unknown desc = error getting list of preferred allocation devices: unable to get device link information: error getting NVLink for devices (0, 1): failed to get nvlink remote pci info: failed to get nvlink state: Unknown Error, which is unexpected
```


