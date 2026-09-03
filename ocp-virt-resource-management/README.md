# CPU overcommit

Default overcommit ratio for CPU is 10/1. In the spec of the hyperconverged CR you will see:

```
resourceRequirements:
  vmiCPUAllocationRatio: 10
```

In the launcher pod you will see the CPU requests. With this default value you will see 100M CPU request.

```
 resources:
   requests:
     cpu: 100m
```


# Memory Management

Kernel samepage sharing (KSM) - VMware equivalent is Transparent page sharing (TPS).

KSM deduplicates identical data found in memory. VMware has TPS disabled by default because there is a risk (even if low) of using TPS to compromise another VM. We also have it disabled by default

KSM should only be enabled if you trust the workloads; therefore, it is a balancing act for the customer and the security team. 

With this in mind, KSM is largely ineffective when working with modern systems where hugepages are enabled, as the likelihood of finding identical 2MB chunks is much lower. Even VMware says the same thing about TPS.

# Swap and Memory overcommit

Swap should be enabled prior to overcommiting memory. Without swap you will get OOM killed VMs. Unlike VMware, SWAP is in the host where VMware by default creates the SWAP file on the same datastore as the VMX.

Docs for SWAP configuration - https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/virtualization/postinstallation-configuration#virt-configuring-higher-vm-workload-density

Memory overcommit is disabled by default. When launching a VM, the launcher pod will have a memory request equal to the total memory of the VM. Resource scheduler will prevent VM launch where total memory requests exceed the total memory of the host.

Looking at the launcher pod you can see the memory requests which will include some overheads.

```
resources:
  requests:
    memory: 1292Mi
```

After configuring SWAP, you can tweak the memory overcommit via the hyperconverged CR. The maximum is 400.

```
spec:
  higherWorkloadDensity:
    memoryOvercommitPercentage: 200
```

This will now halve the memory request for the launcher pod effectively allowing you to schedule more VMs than the physical RAM of the host. Capacity management is required and you should use the descheduler to help balance workloads.

Existing VMs will need to be live migrated or restarted for the memory resource request to be updated.

This is a good blog post on memory management - https://developers.redhat.com/blog/2025/01/31/memory-management-openshift-virtualization#
