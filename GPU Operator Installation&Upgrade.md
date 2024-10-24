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

- Setting persistence mode to 0
```
$ nvidia-smi -i 0000:xx:00.0 -pm 0
```
- Changing drain state of the GPU, preventing it from being scheduled on:
```
$ nvidia-smi drain -p 000:xx:00.0 -m 1
```
#### Upgrade

- Create a MR for subscription update and ClusterPolicy update with latest compatible driver image: https://gitlab.med.one/compute/ocpbm-cluster-config/-/merge_requests/343/diffs
- Driver version: `nvcr.io/nvidia/driver@sha256:351578cd0e4b94cc7738556e4ddcc2d70634414e089934d5e9d754bd868fb395`
- Drain all GPU related nodes in an AZ by AZ to prepare them for post driver upgrade automatic reboot.
- Sync the subscription - wait for install plan creation
- approve the install plan
- Watch the CSV enter success state
- Let the gpu-operator pod restart
- The GPU operator will place upgrade state related labels on the GPU nodes, watch it
```
$ oc get nodes -o json -w | jq '.metadata | select(.labels."nvidia.com/gpu.present"?) | select(.labels."nvidia.com/gpu-driver-upgrade-state"?) | "Node name: \(.name), Upgrade state: \(.labels."nvidia.com/gpu-driver-upgrade-state"), Availability zone: \(.labels."topology.kubernetes.io/zone")"' -r
```
- All pods under the `nvidia-gpu-operator` namespace will go through a restart
- Wait for `upgrade-done` label

- Check with `nvidia-smi` the new installed driver version.
- Sync the cluster policy with the new driver image
- Watch for upgrade instructions through the node label
- Wait for all pods under the `nvidia-gpu-operator` to go through a restart
- Wait for the driver get installed by running `nvidia-smi`
	- Some pods may not be able to get deployed:
		- gpu-feature-discovery
		- nvidia-dcgm-exporter
		- nvidia-device-plugin-validator
	- If the driver in the new version was indeed installed, reboot the nodes in an AZ by AZ manner

- Test deployment of a GPU workload:```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cuda-gpu-deployment
  namespace: nvidia-gpu-operator
  labels:
    app: cuda-gpu-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cuda-gpu-app
  template:
    metadata:
      labels:
        app: cuda-gpu-app
    spec:
      containers:
        - name: cuda-container
          image: nvcr.io/nvidia/cuda:11.8.0-runtime-centos7
          resources:
            limits:
              nvidia.com/gpu: 1  # Request 1 GPU
          command: ["/bin/bash", "-c", "--"]
          args: ["while true; do sleep 30; done;"]
      nodeSelector:
        nvidia.com/gpu.present: "true"
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


- nvcr.io/nvidia/driver@sha256:afc0062898f1ceae3b5e65012a18294184386fd533a0d66a50f339e08bb5a9f5 - `NVIDIA 535.183.06` - GPU Driver version 23.9.2
- nvcr.io/nvidia/driver@sha256:523742bfeeee5975a344098a8dadba3bb3554023f50c89dda7f420ecd987c94a - `NVIDIA 470.223.02` - GPU Drvier version 23.3.2

#### From `gpu-operator` pod logs:
```
$ oc logs gpu-operator-5849c97b7d-sssdx --all-containers | grep controllers.Upgrade

{"level":"info","ts":"2024-10-09T14:51:23Z","logger":"controllers.Upgrade","msg":"Node hosting a driver pod","node":"phmowrk-166018-14.medone-1.med.one","state":"upgrade-failed"}
```

#### Force upgrade
```
$ oc label node phmowrk-166018-14.medone-1.med.one nvidia.com/gpu-driver-upgrade-state=upgrade-required --overwrite
```

#### If a card has an ERROR for some reason and the dcgm exporter pod is unable to be deployed
- Use `nvidia-smi` in order to find out which card has an error
- Use the following command in order to remove the card with the error:
```
$ oc exec nvidia-driver-daemonset-412.86.202309190812-0-jmtnb -- nvidia-smi drain -p 0000:61:00.0 -m 1
```

### If the DCGM-Exporter fails with profiling metrics error
- Run `oc exec nvidia-driver-daemonset-412.86.202309190812-0-jmtnb -- dmesg | grep -i 119 -C 10`
```
[ 3947.590120] NVRM: Xid (PCI:0000:61:00): 119, pid=123417, name=nv-hostengine, Timeout after 6s of waiting for RPC response from GPU0 GSP! Expected function 10 (FREE) (0xc0000005 0x0).
[ 3953.591118] NVRM: Xid (PCI:0000:61:00): 119, pid=123417, name=nv-hostengine, Timeout after 6s of waiting for RPC response from GPU0 GSP! Expected function 10 (FREE) (0xc0000002 0x0).
[ 3959.592099] NVRM: Rate limiting GSP RPC error prints for GPU at PCI:0000:61:00 (printing 1 of every 30).  The GPU likely needs to be reset.
[ 3964.293978] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 3964.295103] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 3967.060943] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 3967.061984] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 3969.822649] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 3969.823806] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 3972.607108] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 3972.608209] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 4131.289367] NVRM: Xid (PCI:0000:61:00): 119, pid=126121, name=nvidia-smi, Timeout after 6s of waiting for RPC response from GPU0 GSP! Expected function 76 (GSP_RM_CONTROL) (0x2080a068 0x4).
[ 4136.586179] mimir invoked oom-killer: gfp_mask=0x6000c0(GFP_KERNEL), order=0, oom_score_adj=999
[ 4136.586186] CPU: 156 PID: 128187 Comm: mimir Tainted: P           OE    --------- -  - 4.18.0-372.73.1.el8_6.x86_64 #1
[ 4136.586189] Hardware name: Cisco Systems Inc UCSC-C245-M6SX/UCSC-C245-M6SX, BIOS C245M6.4.2.3c.0.1104222046 11/04/2022
[ 4136.586191] Call Trace:
[ 4136.586195]  dump_stack+0x41/0x60
[ 4136.586202]  dump_header+0x4a/0x1df
[ 4136.586207]  oom_kill_process.cold.32+0xb/0x10
[ 4136.586209]  out_of_memory+0x1bd/0x4e0
[ 4136.586212]  mem_cgroup_out_of_memory+0xec/0x100
[ 4136.586215]  try_charge+0x64f/0x690
--
[ 4860.928675] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 4860.929763] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 4917.684420] NVRM: GPU 0000:61:00.0: Device is currently unavailable
[ 4917.684433] NVRM: GPU 0000:61:00.0: Device is currently unavailable
[ 4918.519515] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 4918.520765] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 4921.282177] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 4921.283284] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 4952.437873] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 4952.438994] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 4955.227119] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 4955.228225] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 4986.745290] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 4986.746221] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 4990.517041] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 4990.517983] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 4997.176887] NVRM: GPU 0000:61:00.0: Device is currently unavailable
[ 4997.178342] NVRM: GPU 0000:61:00.0: Device is currently unavailable
[ 5016.636398] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 5016.637528] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 5022.403460] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
--
               workingset_refault_anon 0
               workingset_refault_file 453262
               workingset_activate_anon 0
               workingset_activate_file 2
               workingset_restore_anon 0
               workingset_restore_file 0
               workingset_nodereclaim 10481
               pgfault 1816000
               pgmajfault 0
               pgrefill 163
               pgscan 1195039
               pgsteal 1186391
               pgactivate 407
               pgdeactivate 163
               pglazyfree 0
               pglazyfreed 0
               thp_fault_alloc 6025
               thp_collapse_alloc 0
[ 5787.539781] Tasks state (memory values in pages):
[ 5787.539781] [  pid  ]   uid  tgid total_vm      rss pgtables_bytes swapents oom_score_adj name
[ 5787.539782] [ 155586]     0 155586    35965      599   172032        0         -1000 conmon
--
[ 5988.899857] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 5988.900915] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 6052.703598] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 6052.704787] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 6055.477627] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 6055.478747] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 6064.334309] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 6064.335990] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 6067.094569] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 6067.095851] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 6119.559556] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 6119.560845] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 6124.336163] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 6124.337354] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 6128.057424] mimir invoked oom-killer: gfp_mask=0x6000c0(GFP_KERNEL), order=0, oom_score_adj=999
[ 6128.057431] CPU: 47 PID: 160296 Comm: mimir Tainted: P           OE    --------- -  - 4.18.0-372.73.1.el8_6.x86_64 #1
[ 6128.057434] Hardware name: Cisco Systems Inc UCSC-C245-M6SX/UCSC-C245-M6SX, BIOS C245M6.4.2.3c.0.1104222046 11/04/2022
[ 6128.057436] Call Trace:
[ 6128.057440]  dump_stack+0x41/0x60
[ 6128.057446]  dump_header+0x4a/0x1df
[ 6128.057451]  oom_kill_process.cold.32+0xb/0x10
[ 6128.057453]  out_of_memory+0x1bd/0x4e0
--
               pgactivate 524
               pgdeactivate 216
               pglazyfree 0
               pglazyfreed 0
               thp_fault_alloc 7831
               thp_collapse_alloc 0
[ 7753.023632] Tasks state (memory values in pages):
[ 7753.023633] [  pid  ]   uid  tgid total_vm      rss pgtables_bytes swapents oom_score_adj name
[ 7753.023634] [ 184245]     0 184245    35965      583   176128        0         -1000 conmon
[ 7753.023638] [ 184260] 1001240000 184260   615530   236719  2535424        0           999 mimir
[ 7753.023640] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=crio-8371caad8cc369f16c55ce039fbbe0e4fd1be88e72f119bce94ccd5d07d152fe.scope,mems_allowed=0-1,oom_memcg=/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod58dc1221_d430_496e_951d_e8cb23caaead.slice,task_memcg=/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod58dc1221_d430_496e_951d_e8cb23caaead.slice/crio-8371caad8cc369f16c55ce039fbbe0e4fd1be88e72f119bce94ccd5d07d152fe.scope,task=mimir,pid=184260,uid=1001240000
[ 7753.023847] Memory cgroup out of memory: Killed process 184260 (mimir) total-vm:2462120kB, anon-rss:900908kB, file-rss:45968kB, shmem-rss:0kB, UID:1001240000 pgtables:2476kB oom_score_adj:999
[ 7753.053098] oom_reaper: reaped process 184260 (mimir), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
[ 7786.981464] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 7786.982492] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 7789.758826] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 7789.759982] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 7853.542000] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 7853.543281] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1
[ 7856.303046] NVRM: GPU 0000:41:00.0: RmInitAdapter failed! (0x62:0x40:2404)
[ 7856.304303] NVRM: GPU 0000:41:00.0: rm_init_adapter failed, device minor number 1

```

- [ ] The need to perform operations on the node in the case of a defect GPU card
	- [ ] This is against our change policy and could potentially harm clientele workload
	- [ ] There is no certainty regarding whether the operation would succeed
		- [ ] Whether:
			- [ ] Whether you would be able to drain the card
			- [ ] Whether you would be able to unbind/remove the card
			- [ ] etc
	- [ ] There is no certainty when and why an operation on a node should be performed in case of a card defect
- [ ] There is a lack of understanding, at least on my part, when and why a GPU node should be drained and\or rebooted during installation\upgrade\uninstall of the operator
	- [ ] My main issue is that the nvidia driver remains on the nodes after uninstalling the operator. In order to remove it, it is required to drain (to ensure the graceful shutdown of clientele workload) and reboot the node, to ensure the driver is gone. Then, after reinstalling, another reboot is needed to ensure clean and operational state, if I understood correctly.
- [ ] **Lack of understanding why installation\\upgrade failed:** Many times, if not all times, installation and\\or upgrade failed, supposably because a defect GPU card or because the GSP (something something) was included in the driver version and A10 cards did not support it. My main issue is not that the installation and\\or upgrade failed, its that it succeeded some of the times. It is the inconsistency.
- [ ] The operator did an unpredictable reboot to a node, which was also without a drain operation.
- [ ] Lack of immediate line of support