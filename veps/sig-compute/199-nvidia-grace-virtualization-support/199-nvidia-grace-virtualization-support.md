# VEP 199: NVIDIA Grace-Blackwell Support in KubeVirt

## VEP Status Metadata

### Target releases

- This VEP targets alpha for version: v1.9
- This VEP targets beta for version:
- This VEP targets GA for version:

### Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [x] (R) Enhancement issue created, which links to VEP dir in [kubevirt/enhancements] (not the initial VEP PR)
- [x] (R) Alpha target version is explicitly mentioned and approved
- [ ] (R) Beta target version is explicitly mentioned and approved
- [ ] (R) GA target version is explicitly mentioned and approved

## Overview

This VEP enables production-ready GPU passthrough on NVIDIA Grace Hopper and Grace Blackwell (GB200/GB300) systems in KubeVirt. It introduces a new `GraceIOVirtualization` feature gate and an annotation-driven configuration model that controls the following Grace-specific subsystems:

- **SMMUv3 with nested translation** and iommufd device binding
- **Extended GPU Memory (EGM)** as the guest memory backend
- **ACPI Generic Initiator (GI)** with 8 dedicated NUMA nodes per GPU for MIG support
- **Per-GPU isolated PXB hierarchies** with per-bus SMMUv3 scoping
- **Configurable 64-bit MMIO aperture** (pcihole64) for large-BAR GPUs
- **PCIe link speed/width modeling** derived from host sysfs topology
- **Mixed GPU+NIC PCI topology** with independent isolation policies
- **VFIO overhead and memlock sizing** for large-BAR devices
- **Admission validation** for Grace-specific configuration constraints

This VEP depends on [VEP 115: PCIe NUMA Topology Awareness](https://github.com/kubevirt/enhancements/issues/115), which provides the generic NUMA-aware PXB planner architecture that this VEP extends for Grace-specific requirements. VEP 115 supplies the stateful PCI planner with resource reservation, collision prevention, and per-device-group PXB isolation. This VEP layers Grace-specific behaviors on top of that planner: per-GPU PXB isolation for SMMUv3 scoping, ACPI Generic Initiator NUMA cells, Grace-specific NUMA distances, and platform-specific subsystem configuration.

## Motivation

NVIDIA Grace Hopper and Grace Blackwell (GB200/GB300) platforms are becoming the foundation for GPU-accelerated AI training, inference, and HPC workloads in data centers and cloud environments. Running these workloads inside KubeVirt virtual machines enables multi-tenancy, security isolation, and Kubernetes-native lifecycle management — but Grace introduces Cache-Coherent Interconnect between GPU and CPU, and ARM64-specific I/O virtualization requirements, which differ fundamentally from x86 and a PCIe-only system. Without native KubeVirt support, operators must hand-craft QEMU command lines with dozens of platform-specific flags covering SMMUv3, EGM memory backends, ACPI Generic Initiator structures, PXB isolation, and PCIe link overrides. This makes GPU VM provisioning on Grace error-prone, operationally unscalable, and incompatible with declarative Kubernetes workflows.

By integrating Grace-specific virtualization support directly into KubeVirt, this VEP enables:

- **Annotation-driven automation**: A single VM annotation replaces dozens of manual QEMU/libvirt configuration parameters, reducing operator toil and misconfiguration risk.
- **Near-native GPU performance in VMs**: EGM and vCMDQ eliminate PCIe DMA overhead and TLB invalidation bottlenecks, enabling GPU workloads inside VMs to achieve performance parity with bare-metal deployments.
- **Multi-tenant GPU infrastructure**: Kubernetes-managed VMs with proper IOMMU isolation (SMMUv3 + iommufd) allow multiple tenants to share Grace hardware securely, improving GPU utilization and reducing total cost of ownership (TCO).
- **CUDA driver support in guest**: ACPI Generic Initiator support provides the guest NUMA topology required by the NVIDIA driver, enabling a consistent topology on the host.
- **Validated configurations**: Admission-time validation catches invalid Grace configurations before VM creation, preventing runtime failures from incorrect device combinations, missing feature gates, or incompatible memory backends.

### ARM IOMMU (SMMUv3)

Grace uses ARM SMMUv3 with nested translation (stage-1 + stage-2) for hardware-accelerated address translation. Unlike x86 where IOMMU is transparent to the guest, Grace requires explicit SMMUv3 device instances per PCI bus in the QEMU domain. Without these instances, GPU passthrough on Grace fails entirely. Each GPU bus needs `arm-smmuv3` with `accel=on,ats=on,ril=off,pasid=on,oas=48` and optionally `cmdqv=on` for hardware-accelerated command queue virtualization (vCMDQ). KubeVirt must create and configure these SMMUv3 instances automatically based on the VM's device topology.

### Extended GPU Memory (EGM)

Grace Hopper and Blackwell systems can use Extended GPU Memory (EGM), where guest system memory is backed by character devices (`/dev/egmN`) instead of hugepages. EGM provides physically contiguous memory that is directly accessible by the GPU via NVLink-C2C, eliminating PCIe DMA overhead and enabling near-native memory bandwidth for GPU workloads. EGM memory natively supports vCMDQ without requiring hugepage backing, simplifying host memory management. Without EGM support in KubeVirt, operators cannot leverage this performance-critical feature through standard Kubernetes APIs.

### ACPI Generic Initiator

Each Grace GPU requires 8 dedicated NUMA nodes for Multi-Instance GPU (MIG) support. The NVIDIA GPU driver uses these nodes to online memory to the kernel for MIG partitions, regardless of whether MIG is currently enabled. These NUMA nodes are linked to the GPU via ACPI Generic Initiator (GI) structures in the guest firmware tables. Failing to provision these GI nodes causes the guest NVIDIA driver to fail initialization, rendering passed-through GPUs unusable.

### Large PCI BAR Support

Grace GPUs have Base Address Registers (BARs) exceeding 256 GB. The default 64-bit PCI memory window in QEMU (32 GiB) is far too small for these devices, and the VM will fail to map GPU BARs at boot if the window is not enlarged. A configurable `pcihole64` size (4 TiB for GPU-only, 16 TiB for mixed GPU+NIC topologies) is required for the guest to address all device BARs. KubeVirt must expose this as a user-configurable annotation with validated defaults.

### PCIe Link Modeling

Network devices (especially mlx5 ConnectX NICs) negotiate link speed with their upstream root port. If the guest root port reports incorrect or default link characteristics, NIC drivers configure suboptimal link parameters, degrading network throughput significantly. Host sysfs provides the actual negotiated link speed and width for each PCI device, which must be automatically discovered and propagated to the corresponding guest root port to achieve full link performance.

## Goals

- Enable production-ready GPU passthrough on NVIDIA Grace Hopper and Grace Blackwell systems with a single annotation-driven configuration.
- Automatically configure SMMUv3, iommufd, EGM, ACPI GI, PXB isolation, MMIO aperture, and PCIe link modeling without manual QEMU command-line tuning.
- Validate Grace-specific configuration constraints at admission time to prevent invalid VM configurations from being created.
- Support mixed GPU+NIC topologies on Grace systems with independent isolation policies per device type.

## Non Goals

- Upstreaming NVIDIA QEMU/libvirt patches -- those are a platform prerequisite, not a KubeVirt code change.
- vGPU or MIG partitioning orchestration -- this VEP covers passthrough only.
- x86 platforms -- this VEP is Grace/ARM64-specific. Generic NUMA topology improvements are covered by VEP 115.
- Live migration of Grace VMs.
- CXL or other non-Grace heterogeneous memory architectures.

## Assumptions, Constraints, and Dependencies

- **VEP 115**: This VEP requires the stateful NUMA-aware PXB planner architecture from VEP 115, including per-device-group PXB isolation and collision-free resource allocation.
- **Grace Platform Software**: Requires NVIDIA-patched QEMU (`10.1.0+nvidia5`) and libvirt (`11.9.0+nvidia4-1`) for SMMUv3 nested
  translation, vCMDQ, vEGM, and ACPI GI support.
- **Grace Firmware**: NVIDIA GB200 firmware version 1.3 or later for ACPI IORT CANWBS memory access flag support.
- **Machine Type**: ARM64 `virt` machine type.
- **EGM Constraint**: EGM requires all GPUs on a compute tray to be passed through. Partial GPU configurations are not supported with EGM.
- **Host Kernel**: Linux kernel with SMMUv3 nested translation support and iommufd enabled, e.g. 6.17-HWE (linux-nvidia Ubuntu Kernel).

## Definition of Users

- Infrastructure administrators deploying NVIDIA Grace Hopper or Grace Blackwell systems for AI/ML workloads.
- Cloud providers offering high-performance GPU instances on Grace platforms.
- Platform engineers building Kubernetes-native AI infrastructure on ARM64 Grace hardware.

## User Stories

- As a cloud provider deploying Grace systems, I want KubeVirt to automatically configure SMMUv3, EGM, and ACPI GI so that GPU passthrough VMs work without manual QEMU command-line tuning.
- As an AI platform engineer, I want to pass through 4 Grace Blackwell GPUs with EGM-backed memory and have MIG work out of the box inside the VM.
- As an infrastructure administrator, I want KubeVirt to reject invalid Grace configurations (e.g., EGM without all GPUs, vCMDQ without SMMUv3) at admission time rather than failing at VM boot.

## Repos

- [kubevirt](https://github.com/kubevirt/kubevirt) upstream
- [kubevirt-aie nvidia branch](https://github.com/kubevirt/kubevirt-aie/tree/release-1.7-aie-nv) dedicated fork

---

## Design

### Feature Gate

A new `GraceIOVirtualization` feature gate controls all Grace-specific behavior. When enabled, it implicitly activates the `PCINUMAAwareTopology` behavior for Grace VMs regardless of whether the generic gate is also set.

### Platform Detection and Configuration

Grace systems are configured via a VMI annotation:

```
alpha.kubevirt.io/graceVirtualization: '{"hostDevices":true,"smmuv3":true,"vcmdq":true,"egm":true}'
```

The annotation is a JSON object with boolean fields that individually
control Grace subsystems:

```go
type GraceVirtualizationConfig struct {
    HostDevices *bool `json:"hostDevices,omitempty"`
    SMMUv3      *bool `json:"smmuv3,omitempty"`
    VCMDQ       *bool `json:"vcmdq,omitempty"`
    EGM         *bool `json:"egm,omitempty"`
}
```

| Field | Description |
|---|---|
| `hostDevices` | Enable Grace-specific host device topology (per-GPU PXB, GI nodes) |
| `smmuv3` | Enable ARM SMMUv3 with nested translation per GPU bus |
| `vcmdq` | Enable hardware command queue virtualization (requires `smmuv3`) |
| `egm` | Use EGM character devices as guest memory backend |

This annotation is a transitional alpha API. Once the design stabilizes, the intent is to promote these fields to a first-class VMI spec type.

---

## Per-GPU Isolated PXB Hierarchy

Unlike the original VEP 115 which uses one pxb-pcie per NUMA node with multiple root-ports, Grace requires **one pxb-pcie per GPU**. Each GPU gets its own isolated PCI bus so that SMMUv3 can be scoped per bus.

NICs may share a PXB bus or receive isolated PXBs depending on topology and whether SMMUv3 is enabled for that bus.

The NUMA PCI planner groups devices by their host NUMA node and device type, then allocates PXB controllers accordingly:

```
Per-GPU PXB topology (4 GPUs + 1 NIC):
  pcie.0
  ├── pxb-pcie.1 (GPU 0, NUMA 1, SMMUv3.1)
  │   └── pcie-root-port.1
  │       └── vfio-pci-nohotplug (GPU 0)
  ├── pxb-pcie.2 (GPU 1, NUMA 1, SMMUv3.2)
  │   └── pcie-root-port.2
  │       └── vfio-pci-nohotplug (GPU 1)
  ├── pxb-pcie.3 (GPU 2, NUMA 3, SMMUv3.3)
  │   └── pcie-root-port.3
  │       └── vfio-pci-nohotplug (GPU 2)
  ├── pxb-pcie.4 (GPU 3, NUMA 3, SMMUv3.4)
  │   └── pcie-root-port.4
  │       └── vfio-pci-nohotplug (GPU 3)
  └── pxb-pcie.5 (NIC, NUMA 0)
      └── pcie-root-port.5
          └── vfio-pci (NIC 0)
```

### PXB Bus Number Assignment

PXB bus numbers are assigned deterministically using a base+spacing scheme: `busNr = 0x20 + (hostNUMA * 0x20)`. Socket 0 PXBs start at bus 0x21 (33), socket 1 at bus 0x41 (65), with dedicated per-GPU PXBs incrementing by 2 within each range (e.g., 33, 35, 37, 39 for 4 GPUs on socket 0).

The planner tracks used bus numbers and PCI slots to avoid conflicts:

```go
type numaPCIPlanner struct {
	domain              *api.Domain
	nextControllerIndex int
	nextChassis         int
	nextPXBSlot         int
	nextPCIBus          int
	usedBusSlots        map[busSlotKey]map[int]struct{}
	usedNUMAChassis     map[int]struct{}
	usedPCIBuses        map[int]struct{}
	pxbs                map[pxbKey]*pxbInfo
	rootPorts           map[deviceGroupKey]*rootPortInfo
	pcieSwitches        map[deviceGroupKey]*pcieSwitchInfo // Track switches per device group
	existingPXBs        map[string]*pxbInfo
	existingRootPorts   map[string]*rootPortInfo
	nextRootHotplugSlot int
	nextRootHotplugPort int
	// When enabled, NUMA PXB controllers are emitted without user aliases so libvirt
	// keeps the default PCI controller IDs (pci.<index>), which are required by
	// libvirt-native smmuv3 driver pciBus wiring.
	useDefaultPXBIDs bool
}
```

### PCIe Switch Topology Mirroring

On Grace systems, GPUs connect to the CPU via NVLink-C2C, not PCIe.
Each GPU appears on its own PCI domain with a direct two-element path (root port -> device, no intermediary switch):

```
Grace GPU topology (from lspci -vt):
  -[0008:00]---00.0-[01]----00.0  NVIDIA Corporation Device 2941  (GPU 0)
  -[0009:00]---00.0-[01]----00.0  NVIDIA Corporation Device 2941  (GPU 1)
  -[0018:00]---00.0-[01]----00.0  NVIDIA Corporation Device 2941  (GPU 2)
  -[0019:00]---00.0-[01]----00.0  NVIDIA Corporation Device 2941  (GPU 3)
```

However, other PCIe devices on Grace systems -- notably ConnectX-7 NICs, BlueField-3 DPUs, and NVMe storage -- do traverse PCIe switches:

```
Grace NIC/storage topology (from lspci -vt):
  -[0000:00]---00.0-[01-07]----00.0-[02-07]--+-00.0-[03]----00.0  ConnectX-7
                                             \-01.0-[04-07]----00.0-[05-07]--...
  -[0006:00]---00.0-[01-09]----00.0-[02-09]--+-00.0-[03]----00.0  BlueField-3
                                             \-02.0-[04-09]----00.0-[05-09]--+-00.0-[06]  NVMe
                                                                             +-04.0-[07]  NVMe
                                                                             +-08.0-[08]  NVMe
                                                                             \-0c.0-[09]  NVMe
```

AI frameworks (NCCL, UCX) use PCIe hierarchy to detect device locality -- devices behind the same switch are considered "closer" than devices on different switches or root ports. The planner preserves this hierarchy in the guest so that framework topology detection works correctly.

The planner detects PCIe switch topologies by analyzing the PCI path hierarchy from sysfs for each passthrough device. Devices whose path shares the same upstream bridge (path depth >= 3 with a common `path[1]`) are grouped together:

```go
func ComputePCISwitchGroupKey(path []string, bdf string) string {
    // path = ["root_port", "switch_upstream", "switch_downstream", "device"]
    // Devices with the same path[0..1] share the same physical PCIe switch.
    if len(path) >= 3 {
        return strings.Join(path[:2], "/")
    }
    // Shorter paths (GPUs on Grace, direct-attached NICs): no switch
    ...
}
```

When multiple devices share a PCIe switch (e.g., NVMe SSDs behind a BlueField-3 DPU on Grace, or GPUs behind an HGX switch on x86 systems), the planner mirrors it in the guest using QEMU's `x3130-upstream` and `xio3130-downstream` controller models:

```
Guest topology (mirrored NVMe behind BlueField-3):
  pcie.0
  └── pxb-pcie (NUMA node)
        └── pcie-root-port
              └── pcie-switch-upstream-port (x3130-upstream)
                    ├── xio3130-downstream → NVMe 0
                    ├── xio3130-downstream → NVMe 1
                    ├── xio3130-downstream → NVMe 2
                    └── xio3130-downstream → NVMe 3
```

The number of downstream ports matches the physical topology (default 4, scaling to 8 or 16 when more devices share a switch). Devices with a short path (path depth < 3), such as Grace GPUs, skip switch creation and attach directly to a guest root port.

The corresponding domain XML for a PCIe switch hierarchy:

```xml
<!-- Root port under PXB -->
<controller type='pci' index='3' model='pcie-root-port'>
  <target chassis='1' port='0x0'/>
  <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
</controller>

<!-- PCIe switch upstream port attached to root port -->
<controller type='pci' index='4' model='pcie-switch-upstream-port'>
  <model name='x3130-upstream'/>
  <address type='pci' domain='0x0000' bus='3' slot='0x00' function='0x0'/>
</controller>

<!-- PCIe switch downstream ports -->
<controller type='pci' index='5' model='pcie-switch-downstream-port'>
  <model name='xio3130-downstream'/>
  <target chassis='2' port='0x0'/>
  <address type='pci' domain='0x0000' bus='4' slot='0x00' function='0x0'/>
</controller>
<controller type='pci' index='6' model='pcie-switch-downstream-port'>
  <model name='xio3130-downstream'/>
  <target chassis='3' port='0x1'/>
  <address type='pci' domain='0x0000' bus='4' slot='0x01' function='0x0'/>
</controller>
```

This feature is orthogonal to per-GPU PXB isolation: PXBs provide NUMA-level grouping while PCIe switches provide intra-NUMA device
locality. On a Grace Blackwell system the combined topology looks like:

```
pcie.0
├── pxb-pcie.1 (GPU 0, NUMA 1, SMMUv3)      ← direct root-port, no switch
│   └── pcie-root-port → GPU 0
├── pxb-pcie.2 (GPU 1, NUMA 1, SMMUv3)
│   └── pcie-root-port → GPU 1
├── pxb-pcie.3 (NIC + NVMe, NUMA 0)         ← switch-mirrored hierarchy
│   └── pcie-root-port
│         └── x3130-upstream
│               ├── xio3130-downstream → ConnectX-7
│               └── xio3130-downstream → NVMe 0
└── pxb-pcie.4 (NIC, NUMA 2)
    └── pcie-root-port → ConnectX-7
```

### Large MMIO PXB Isolation

Grace GPUs expose enormous MMIO BARs (256 GB+ for device memory accessed via the C2C interconnect). When multiple large-BAR devices share a PXB hierarchy, they can exhaust the 64-bit PCI address space of that bus segment and cause guest firmware to fail BAR assignment.

The planner automatically detects large-MMIO devices and isolates each one onto its own dedicated PXB hierarchy. The threshold is 128 GiB of total MMIO aperture:

```go
const (
    ioResourceMemFlag                 = 0x00000200
    largeMMIOPXBIsolationThresholdGiB = uint64(128)
    largeMMIOPXBIsolationThreshold    = largeMMIOPXBIsolationThresholdGiB << 30
)
```

#### MMIO Size Detection

The planner reads the sysfs `resource` file for each passthrough device to compute its total MMIO footprint:

```go
func getDevicePCITotalMMIOSize(bdf string) (uint64, error) {
    // Parses /sys/bus/pci/devices/<bdf>/resource
    // Each line: start_addr  end_addr  flags
    // Sums (end - start + 1) for all memory BARs (flags & 0x200 != 0)
}
```

Devices at or above the threshold are marked `dedicatedPXB = true`,
which has three effects:

1. **PXB isolation**: The device receives its own `pxb-pcie` controller rather than sharing one with other devices on the same NUMA node. This is implemented by setting a unique `pxbGroup` key derived from the device's PCI path, which forces the planner to create a separate PXB even when the NUMA node already has one.

2. **GI node assignment**: Only `dedicatedPXB` devices (i.e., GPU-class devices) receive ACPI Generic Initiator NUMA node ranges. This correctly limits the 8 GI nodes per device to actual GPUs rather than NICs or other small-BAR peripherals.

3. **iommufd binding**: When iommufd is available, `dedicatedPXB` devices use iommufd hostdev binding regardless of whether global SMMUv3 is enabled, since their large BARs benefit from the modern IOMMU path.

#### Mixed Topology Forcing

When SMMUv3 is enabled and the topology contains both large-MMIO devices (GPUs) and small-MMIO devices (NICs), the planner forces
isolated PXB hierarchies for **all** passthrough devices:

```go
func shouldForceIsolatedPXBsForGraceMixedTopology(
    graceSMMUv3Enabled bool, devices []deviceNUMAInfo) bool {
    // Returns true when both dedicatedPXB (large) and
    // non-dedicatedPXB (small) devices are present.
}
```

This prevents a scenario where a NIC sharing a PXB with a GPU inherits the GPU's SMMUv3 scoping constraints, which would cause incorrect SMMU
configuration for the NIC's bus.

---

## SMMUv3 and iommufd Wiring

### SMMUv3 Device Instances

When the annotation enables `smmuv3`, the planner emits libvirt-native IOMMU elements scoped to each GPU's PXB bus:

```xml
<devices>
  <iommu model='smmuv3'>
    <driver pciBus='0x01' accel='on' ats='on' ril='off'
            pasid='on' oas='48' cmdqv='on'/>
  </iommu>
  <iommu model='smmuv3'>
    <driver pciBus='0x02' accel='on' ats='on' ril='off'
            pasid='on' oas='48' cmdqv='on'/>
  </iommu>
</devices>
```

Each attribute:

| Attribute | Value | Description |
|---|---|---|
| `pciBus` | PXB bus number | Scopes this SMMUv3 to one PCI bus |
| `accel` | `on` | Enable nested translation (host stage-2 + guest stage-1) |
| `ats` | `on` | Address Translation Services for PCIe ATS capability |
| `ril` | `off` | Range Invalidation Length (disabled for Grace) |
| `pasid` | `on` | Process Address Space ID support |
| `oas` | `48` | Output Address Size (48-bit physical address) |
| `cmdqv` | `on` | vCMDQ hardware command queue virtualization |

The schema additions:

```go
type IOMMUDriver struct {
    PCIBus string `xml:"pciBus,attr,omitempty"`
    Accel  string `xml:"accel,attr,omitempty"`
    ATS    string `xml:"ats,attr,omitempty"`
    RIL    string `xml:"ril,attr,omitempty"`
    PASID  string `xml:"pasid,attr,omitempty"`
    OAS    string `xml:"oas,attr,omitempty"`
    CMDQV  string `xml:"cmdqv,attr,omitempty"`
}
```

### iommufd Fallback

When `/dev/iommu` is not available on the host (e.g., kernel without iommufd support), the planner falls back to non-accelerated mode
(`accel=off`) to maintain basic functionality without nested translation.

### Hostdev iommufd Binding

Host devices on Grace use iommufd binding instead of legacy VFIO groups:

```go
type HostDeviceDriver struct {
    IOMMUFD string `xml:"iommufd,attr,omitempty"`
}
```

```xml
<hostdev mode='subsystem' type='pci' managed='no'>
  <driver iommufd='yes'/>
  <source>
    <address domain='0x0000' bus='0x09' slot='0x01' function='0x0'/>
  </source>
</hostdev>
```

---

## ACPI Generic Initiator

Each Grace GPU requires 8 dedicated NUMA nodes for MIG support. The NVIDIA GPU driver uses these nodes to online memory to the kernel for MIG partitions, regardless of whether MIG is currently enabled.

### GI Node Allocation

The planner allocates GI node ranges per GPU starting from a base offset after the last CPU/device NUMA cell:

```go
const giNodesPerGPU = 8

func allocateGINodeRanges(gpuDevices []deviceInfo, baseNodeID int) map[string]giRange {
    ranges := make(map[string]giRange)
    nextNode := baseNodeID
    for _, gpu := range gpuDevices {
        ranges[gpu.pciAddress] = giRange{
            start: nextNode,
            end:   nextNode + giNodesPerGPU - 1,
        }
        nextNode += giNodesPerGPU
    }
    return ranges
}
```

### Guest NUMA Cell Layout

Zero-memory guest NUMA cells are created for each GI node. The hostdev ACPI `nodeset` attribute references these cells:

```xml
<cpu>
  <numa>
    <!-- CPU cell(s) from guestMappingPassthrough -->
    <cell id='0' cpus='0-59' memory='468713472' unit='KiB'/>
    <!-- GI nodes for GPU 0 (cells 1-8) -->
    <cell id='1' cpus='' memory='0' unit='KiB'/>
    <cell id='2' cpus='' memory='0' unit='KiB'/>
    <cell id='3' cpus='' memory='0' unit='KiB'/>
    <cell id='4' cpus='' memory='0' unit='KiB'/>
    <cell id='5' cpus='' memory='0' unit='KiB'/>
    <cell id='6' cpus='' memory='0' unit='KiB'/>
    <cell id='7' cpus='' memory='0' unit='KiB'/>
    <cell id='8' cpus='' memory='0' unit='KiB'/>
    <!-- GI nodes for GPU 1 (cells 9-16) -->
    <cell id='9' cpus='' memory='0' unit='KiB'/>
    <!-- ... -->
  </numa>
</cpu>

<devices>
  <hostdev mode='subsystem' type='pci' managed='no'>
    <driver iommufd='yes'/>
    <source>
      <address domain='0x0000' bus='0x09' slot='0x01' function='0x0'/>
    </source>
    <acpi nodeset='1-8'/>
  </hostdev>
</devices>
```

The schema additions:

```go
type HostDeviceACPI struct {
    NodeSet string `xml:"nodeset,attr"`
}
```

### Grace Cross-NUMA Device Placement

On Grace systems, GPUs are physically on a separate NUMA node from all CPUs (connected via NVLink-C2C, not PCIe). When `guestMappingPassthrough` is configured, the guest NUMA topology is built from vCPU pinning, which only creates cells for the CPU NUMA node(s). The GPU's NUMA node has no corresponding guest cell.

VEP 115 identifies generic cross-NUMA CPU-less cell creation as future work. Grace does not need this generic mechanism because the ACPI Generic Initiator (GI) node allocation provides a superset of the cross-NUMA cell concept:

- Each GPU receives 8 dedicated CPU-less guest NUMA cells (GI nodes).
- The first GI node serves as the GPU's "device NUMA node" in the guest, providing the same NUMA affinity visibility that a generic cross-NUMA cell would provide.
- The remaining 7 GI nodes support MIG partitioning.
- The `pcie-expander-bus` controller for each GPU targets the GI node range, and the hostdev `acpi nodeset` binds the GPU to its GI cells.

This approach is superior to a single generic cross-NUMA cell per device because it simultaneously solves device NUMA affinity, MIG support, and per-GPU bus isolation. The planner's `ensureGuestNUMACellRange` function creates these zero-memory CPU-less cells:

```go
func ensureGuestNUMACellRange(domain *api.Domain, start, end int) {
    // Creates NUMACell{ID, Memory: 0, Unit: "KiB"} for each missing ID
    // in the range [start, end]. No CPUs are assigned to these cells.
}
```

---

## Extended GPU Memory (EGM)

### Discovery

When the annotation enables `egm`:

1. Enumerate `/sys/class/egm/egm*` devices on the host.
2. Read `gpu_devices` to correlate each EGM device with its GPU PCI address.
3. Read `egm_size` to determine the available EGM memory per device.
4. Validate that all GPUs on the compute tray are passed through (partial EGM is not supported).

### Domain Configuration

EGM replaces hugepages as the guest memory backend. The domain-level `<memoryBacking>` switches to file-backed shared memory:

```xml
<domain type='kvm'>
  <memory unit='b'>959925190656</memory>
  <memoryBacking>
    <source type='file'/>
    <access mode='shared'/>
    <allocation mode='immediate'/>
  </memoryBacking>
</domain>
```

Each CPU NUMA cell uses `memory-backend-file` pointing to `/dev/egmN`. On a 2-socket Grace Blackwell GB200 system with 2 GPUs per socket, each EGM device provides the aggregate memory for its socket. One `memory-backend-file` is created per host socket (per `/dev/egmN` character device), and each CPU NUMA cell references the EGM device for its socket:

```
QEMU objects (generated by the converter, 2-socket GB200 example):
  -object memory-backend-file,id=m0,mem-path=/dev/egm4,size=457728M,share=on,prealloc=on
  -object memory-backend-file,id=m1,mem-path=/dev/egm5,size=457728M,share=on,prealloc=on
  -numa node,memdev=m0,cpus=0-59,nodeid=0
  -numa node,memdev=m1,cpus=60-119,nodeid=1
```

### Per-GPU EGM Memory Devices

In addition to the CPU NUMA cell memory backends, each GPU requires a `<memory model='egm'>` device element that associates the GPU with its EGM region. The planner generates these via `applyEGMMemoryDevices`, which correlates each passthrough GPU with its host EGM character device and computes the per-GPU memory size:

```go
domain.Spec.Devices.MemoryDevices = append(domain.Spec.Devices.MemoryDevices,
    api.MemoryDevice{
        Model:  "egm",
        Access: "shared",
        Source: &api.MemoryDeviceSource{Path: egm.DevPath},
        Target: &api.MemoryTarget{
            Size:   api.Memory{Value: perGPUSizeMiB, Unit: "MiB"},
            Node:   strconv.Itoa(gpuInfo.guestNUMANode),
            PCIDev: "ua-" + hostdevAlias,
        },
    })
```

The resulting domain XML (2-socket GB200, 2 GPUs per socket, 228864 MiB per GPU):

```xml
<devices>
  <!-- GPU 0 EGM: /dev/egm4 (socket 0) -->
  <memory model='egm' access='shared'>
    <source><path>/dev/egm4</path></source>
    <target>
      <size unit='MiB'>228864</size>
      <node>0</node>
      <pciDev>ua-hostdev0</pciDev>
    </target>
  </memory>
  <!-- GPU 1 EGM: /dev/egm4 (socket 0, shared with GPU 0) -->
  <memory model='egm' access='shared'>
    <source><path>/dev/egm4</path></source>
    <target>
      <size unit='MiB'>228864</size>
      <node>0</node>
      <pciDev>ua-hostdev1</pciDev>
    </target>
  </memory>
  <!-- GPU 2 EGM: /dev/egm5 (socket 1) -->
  <memory model='egm' access='shared'>
    <source><path>/dev/egm5</path></source>
    <target>
      <size unit='MiB'>228864</size>
      <node>1</node>
      <pciDev>ua-hostdev2</pciDev>
    </target>
  </memory>
  <!-- GPU 3 EGM: /dev/egm5 (socket 1, shared with GPU 2) -->
  <memory model='egm' access='shared'>
    <source><path>/dev/egm5</path></source>
    <target>
      <size unit='MiB'>228864</size>
      <node>1</node>
      <pciDev>ua-hostdev3</pciDev>
    </target>
  </memory>
</devices>
```

The `pciDev` attribute references the GPU hostdev alias, which QEMU uses to create `acpi-egm-memory` objects that link each GPU to its EGM backing. This is required for the NVIDIA GPU driver to correctly identify and use EGM memory inside the guest.

#### EGM Size Partitioning

When multiple GPUs share a single EGM character device (e.g., 2 GPUs per socket on GB200), the total EGM size is divided evenly:

```
Total EGM for /dev/egm4: 457728 MiB (2 GPUs on socket 0)
Per-GPU EGM size: 228864 MiB (457728 / 2)
```

The planner validates that the total EGM size divides evenly by the number of GPUs sharing the device, and rejects configurations where
only a subset of GPUs on a socket are passed through.

### Schema Additions for EGM

```go
type MemoryDevice struct {
    Model  string              `xml:"model,attr"`
    Access string              `xml:"access,attr,omitempty"`
    Source *MemoryDeviceSource `xml:"source,omitempty"`
    Target *MemoryTarget       `xml:"target"`
}

type MemoryDeviceSource struct {
    Path string `xml:"path"`
}

type MemoryTarget struct {
    Size   Memory `xml:"size"`
    Node   string `xml:"node"`
    PCIDev string `xml:"pciDev,omitempty"`
}
```

### Constraints

- EGM rejects hugepages: `spec.domain.memory.hugepages` must not be set.
- EGM requires all GPUs sharing the same EGM character device (`/dev/egmN`) to be passed through to the VM — partial EGM configurations are unsupported. On Grace Blackwell GB200 systems each compute tray exposes one EGM device per socket, so enabling EGM effectively mandates a compute-tray-sized VM that consumes every GPU on that tray. Without EGM, a subset of GPUs can be assigned to the VM, allowing multiple smaller VMs to coexist on the same compute tray.
- EGM memory is natively contiguous and does not require hugepage backing for vCMDQ.
- Each `/dev/egmN` character device must be exposed to the virt-launcher pod.

---

## Configurable 64-bit MMIO Aperture

Grace GPUs have BARs exceeding 256 GB. The default PCI 64-bit hole is insufficient. KubeVirt supports a configurable `pcihole64` size via annotation. The annotation value is in **KiB**:

```
kubevirt.io/pciHole64Size: "4294967296"  # 4 TiB in KiB
```

The converter emits the `<pcihole64>` element on the `pcie-root` controller:

```xml
<controller type='pci' index='0' model='pcie-root'>
  <pcihole64 unit='KiB'>4294967296</pcihole64>
</controller>
```

A safety cap of 16 TiB (`17179869184` KiB) is enforced.

| Platform | Recommended size | Annotation value (KiB) |
|---|---|---|
| Grace Hopper (1 GPU) | 4 TiB | `"4294967296"` |
| Grace Blackwell GB200 (4 GPUs only) | 4 TiB | `"4294967296"` |
| Grace Blackwell GB200 (4 GPUs + 4 NICs) | 4 TiB | `"4294967296"` |
| GB300 (4 GPUs) | 8 TiB | `"8589934592"` |

Mixed GPU+NIC topologies require a larger aperture because each device (including SR-IOV NICs) gets its own isolated PXB, and the total MMIO space must accommodate all device BARs.

Schema:

```go
type PCIHole64 struct {
    Value uint `xml:",chardata"`
    Unit  string `xml:"unit,attr,omitempty"`
}
```

---

## PCIe Link Speed and Width Modeling

Guest root ports need accurate PCIe link characteristics for correct bandwidth reporting to drivers (especially mlx5 NICs that negotiate link speed with the root port).

### Discovery

The planner reads host sysfs attributes for each BDF along the PCI device path, preferring `max_link_speed` over `current_link_speed` and `max_link_width` over `current_link_width`:

```go
func readPCIeLinkSpeedForBDF(bdf string) string {
    for _, attr := range []string{"max_link_speed", "current_link_speed"} {
        data, err := readPCIDeviceFile(
            filepath.Join("/sys/bus/pci/devices", bdf, attr))
        if err == nil && data != "" {
            return normalizePCIeLinkSpeed(data)
        }
    }
    return ""
}
```

Raw sysfs values are normalized to QEMU root-port property values:

| Sysfs speed | QEMU `x-speed` | PCIe generation |
|---|---|---|
| 2.5 GT/s | `2_5` | Gen1 |
| 5.0 GT/s | `5` | Gen2 |
| 8.0 GT/s | `8` | Gen3 |
| 16.0 GT/s | `16` | Gen4 |
| 32.0 GT/s | `32` | Gen5 |
| 64.0 GT/s | `64` | Gen6 |

### Bottleneck-Aware Derivation

When a device's PCI path traverses multiple bridge hops (root port -> switch upstream -> switch downstream -> device), each hop may have a different link capability. The planner computes the **bottleneck** -- the minimum speed and minimum width across all hops:

```go
func derivePCIeLinkCharacteristicsFromPath(path []string) *pcieLinkCharacteristics {
    var result *pcieLinkCharacteristics
    for _, bdf := range path {
        speed := readPCIeLinkSpeedForBDF(bdf)
        width := readPCIeLinkWidthForBDF(bdf)
        if result == nil {
            result = &pcieLinkCharacteristics{speed: speed, width: width}
            continue
        }
        if speed != "" && comparePCIeSpeed(speed, result.speed) < 0 {
            result.speed = speed  // take the slower speed
        }
        if width != "" && comparePCIeWidth(width, result.width) < 0 {
            result.width = width  // take the narrower width
        }
    }
    return result
}
```

This derivation is applied per device group during PXB allocation. The resulting link characteristics are injected as `qemu:override` properties on the guest root port:

```go
func applyGuestRootPortLinkCharacteristics(domain *api.Domain,
    rootPort *rootPortInfo, link *pcieLinkCharacteristics) {
    if link.speed != "" {
        upsertQEMUOverrideProperty(domain, rootPort.qemuID,
            "x-speed", "string", link.speed)
    }
    if link.width != "" {
        upsertQEMUOverrideProperty(domain, rootPort.qemuID,
            "x-width", "string", link.width)
    }
}
```

This preserves per-device accuracy -- each device type on the same host gets its root-port link characteristics derived from the actual host topology:

| Device type | Typical x-speed | Typical x-width | Reason |
|---|---|---|---|
| GPU (Grace C2C) | `16` (Gen4) | `1` | Synthetic PCIe over NVLink-C2C; minimal link width |
| NIC (ConnectX-7 IB) | `32` (Gen5) | `16` | Physical PCIe Gen5 x16 link |

---

## Grace NUMA Distance Matrix

When Grace host devices are enabled, the planner computes inter-node distances based on the relationship between NUMA cells. Unlike a generic sysfs-based approach (which VEP 115 identifies as future work), Grace requires a platform-specific distance model because the ACPI Generic Initiator nodes create a guest NUMA topology that does not exist on the host (the host has 2-4 NUMA nodes, but a 4-GPU guest may have 34+). Reading host sysfs distances would be meaningless for these synthetic GI cells.

The Grace distance model captures the actual interconnect topology:

```go
const (
    graceNUMADistanceLocal        = 10   // same node
    graceNUMADistanceSameGPUGroup = 11   // GI nodes within same GPU
    graceNUMADistanceRemoteNode   = 40   // cross-socket CPU-to-CPU
    graceNUMADistanceLocalToGPU   = 80   // CPU to local GPU
    graceNUMADistanceRemoteToGPU  = 120  // CPU to remote GPU
)
```

The planner classifies each guest NUMA cell by kind (CPU cell, GI cell) and computes per-pair distances:

| Source | Destination | Distance | Path |
|---|---|---|---|
| CPU cell | Same CPU cell | 10 | Local |
| CPU cell | Other CPU cell | 40 | Cross-socket |
| CPU cell | GI cell (local GPU) | 80 | NVLink-C2C |
| CPU cell | GI cell (remote GPU) | 120 | Cross-socket + NVLink-C2C |
| GI cell | Same GI cell (self) | 10 | Self |
| GI cell | Same GPU GI cell | 11 | Same GPU group |
| GI cell | Other GPU GI cell | 40 | Cross-GPU (uniform) |

For a 4-GPU, 2-socket system with 34 NUMA cells (2 CPU + 32 GI), each cell
carries a full 34-entry distance vector. Example for CPU cell 0 (socket 0):

```xml
<cell id='0' cpus='0-59' memory='468713472' unit='KiB'>
  <distances>
    <sibling id='0' value='10'/>     <!-- self -->
    <sibling id='1' value='40'/>     <!-- cross-socket CPU -->
    <sibling id='2' value='80'/>     <!-- GPU 0 GI (local) -->
    <!-- ... sibling 3-9: 80 -->
    <sibling id='10' value='80'/>    <!-- GPU 1 GI (local) -->
    <!-- ... sibling 11-17: 80 -->
    <sibling id='18' value='120'/>   <!-- GPU 2 GI (remote) -->
    <!-- ... sibling 19-33: 120 -->
  </distances>
</cell>
```

### Schema Additions for NUMA Distances

The existing `NUMACell` struct gains an optional `Distances` field and the `CPUs` field changes from required to optional (to support CPU-less GI cells):

```go
type NUMACell struct {
    ID           string             `xml:"id,attr"`
    CPUs         string             `xml:"cpus,attr,omitempty"`
    Memory       uint64             `xml:"memory,attr"`
    Unit         string             `xml:"unit,attr,omitempty"`
    MemoryAccess string             `xml:"memAccess,attr,omitempty"`
    Distances    *NUMACellDistances `xml:"distances,omitempty"`
}

type NUMACellDistances struct {
    Siblings []NUMACellSibling `xml:"sibling"`
}

type NUMACellSibling struct {
    ID    string `xml:"id,attr"`
    Value uint64 `xml:"value,attr"`
}
```

These schema changes are shared with the NUMACell definition from VEP 115 and support both Grace-specific distances and future generic distance passthrough.

---

## Admission Validation

When `GraceIOVirtualization` is enabled, the admission webhook validates the annotation with strict JSON decoding (unknown fields rejected):

```go
func validateGraceVirtualizationAnnotation(vmi *v1.VirtualMachineInstance,
    config *KubeVirtClusterConfig) []metav1.StatusCause {
    // Feature gate check
    // JSON parse with DisallowUnknownFields
    // Dependency validation
}
```

### Validation Rules

| Rule | Reason |
|---|---|
| `vcmdq` requires `smmuv3` | vCMDQ is a SMMUv3 hardware acceleration feature |
| `egm` requires `smmuv3` and `hostDevices` | EGM memory is tied to passed-through GPUs |
| `egm` rejects `hugepages` | EGM memory is natively contiguous |
| `egm` requires `dedicatedCPUPlacement` | NUMA-aware topology requires pinned vCPUs |

### Dependency Graph

```
dedicatedCPUPlacement
  └── egm (rejects hugepages, requires smmuv3, requires hostDevices)

hostDevices
  └── smmuv3
        └── vcmdq (requires hugepages when EGM is not enabled)  
```

---

## VFIO Overhead and Memlock Sizing

Grace GPUs have enormous BARs (256 GB+) that must be mapped into the QEMU process address space. The VFIO overhead and memlock limit calculations must account for:

1. **Device BAR sizes**: Read from sysfs `resource` file for each
   passthrough device.
2. **MMIO mapping overhead**: Each BAR mapping consumes memlock quota.
3. **Safety margin**: Additional headroom for QEMU internal allocations.

The converter computes the required memlock limit and passes it to the virt-launcher pod's security context.

---

## Schema Changes Summary

### New Types

```go
type IOMMUDriver struct {
    PCIBus string `xml:"pciBus,attr,omitempty"`
    Accel  string `xml:"accel,attr,omitempty"`
    ATS    string `xml:"ats,attr,omitempty"`
    RIL    string `xml:"ril,attr,omitempty"`
    PASID  string `xml:"pasid,attr,omitempty"`
    OAS    string `xml:"oas,attr,omitempty"`
    CMDQV  string `xml:"cmdqv,attr,omitempty"`
}

type HostDeviceDriver struct {
    IOMMUFD string `xml:"iommufd,attr,omitempty"`
}

type HostDeviceACPI struct {
    NodeSet string `xml:"nodeset,attr"`
}

type PCIHole64 struct {
    Value uint   `xml:",chardata"`
    Unit  string `xml:"unit,attr,omitempty"`
}

type NUMACellDistances struct {
    Siblings []NUMACellSibling `xml:"sibling"`
}

type NUMACellSibling struct {
    ID    string `xml:"id,attr"`
    Value uint64 `xml:"value,attr"`
}
```

### Modified Types

```go
type NUMACell struct {
    // CPUs field changes from required to omitempty (for CPU-less GI cells)
    CPUs      string             `xml:"cpus,attr,omitempty"`
    // New field for Grace NUMA distance matrix
    Distances *NUMACellDistances `xml:"distances,omitempty"`
}
```

---

## API Examples

### KubeVirt Configuration

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
spec:
  configuration:
    developerConfiguration:
      featureGates:
      - GraceIOVirtualization
      - PCINUMAAwareTopology
```

### Grace Blackwell VMI (4 GPUs + 4 IB NICs, EGM, SMMUv3, vCMDQ)

Full-featured configuration with EGM-backed memory, 4 GPU passthrough, 4 InfiniBand SR-IOV NICs, and all Grace subsystems enabled. When EGM is enabled, `guestMappingPassthrough` is not required -- the EGM converter builds guest NUMA cells from EGM device discovery.

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  annotations:
    kubevirt.io/pciHole64Size: "4294967296"  # 4 TiB in KiB
    alpha.kubevirt.io/graceVirtualization: '{"hostDevices":true,"smmuv3":true,"vcmdq":true,"egm":true}'
  name: grace-4gpu-4ib-egm
spec:
  architecture: arm64
  domain:
    cpu:
      cores: 60
      model: host-passthrough
      maxSockets: 2
      sockets: 2
      threads: 1
      dedicatedCpuPlacement: true
    devices:
      autoattachGraphicsDevice: false
      autoattachMemBalloon: false
      disks:
      - disk:
          bus: virtio
        name: rootdisk
      interfaces:
      - name: default
        bridge: {}
      - name: ib1
        sriov: {}
      - name: ib2
        sriov: {}
      - name: ib3
        sriov: {}
      - name: ib4
        sriov: {}
      hostDevices:
      - deviceName: nvidia.com/GB100_HGX_GB200
        name: gpu1
      - deviceName: nvidia.com/GB100_HGX_GB200
        name: gpu2
      - deviceName: nvidia.com/GB100_HGX_GB200
        name: gpu3
      - deviceName: nvidia.com/GB100_HGX_GB200
        name: gpu4
    firmware:
      bootloader:
        efi:
          secureBoot: false
    machine:
      type: virt
    memory:
      guest: 915456Mi # guest memory is backed by EGM devices, the amount must match EGM reserved memory on the host
    resources:
      requests:
        memory: 10Gi
  networks:
  - multus:
      default: true
      networkName: default/ovn-primary
    name: default
  - multus:
      networkName: tenant/ib-pf-network-1
    name: ib1
  - multus:
      networkName: tenant/ib-pf-network-1
    name: ib2
  - multus:
      networkName: tenant/ib-pf-network-1
    name: ib3
  - multus:
      networkName: tenant/ib-pf-network-1
    name: ib4
  volumes:
  - name: rootdisk
    persistentVolumeClaim:
      claimName: pvc-rootdisk
```

### Grace Blackwell VMI (4 GPUs, EGM, minimal)

Minimal EGM configuration with GPUs only, no SR-IOV NICs.

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  annotations:
    kubevirt.io/pciHole64Size: "4294967296"  # 4 TiB in KiB
    alpha.kubevirt.io/graceVirtualization: '{"hostDevices":true,"smmuv3":true,"vcmdq":true,"egm":true}'
  name: grace-4gpu-egm
spec:
  architecture: arm64
  domain:
    cpu:
      cores: 60
      model: host-passthrough
      maxSockets: 2
      sockets: 2
      threads: 1
      dedicatedCpuPlacement: true
    devices:
      autoattachGraphicsDevice: false
      autoattachMemBalloon: false
      disks:
      - disk:
          bus: virtio
        name: rootdisk
      interfaces:
      - name: default
        bridge: {}
      hostDevices:
      - deviceName: nvidia.com/GB100_HGX_GB200
        name: gpu1
      - deviceName: nvidia.com/GB100_HGX_GB200
        name: gpu2
      - deviceName: nvidia.com/GB100_HGX_GB200
        name: gpu3
      - deviceName: nvidia.com/GB100_HGX_GB200
        name: gpu4
    firmware:
      bootloader:
        efi:
          secureBoot: false
    machine:
      type: virt
    memory:
      guest: 915456Mi
    resources:
      requests:
        memory: 10Gi
  networks:
  - multus:
      default: true
      networkName: default/ovn-primary
    name: default
  volumes:
  - name: rootdisk
    persistentVolumeClaim:
      claimName: pvc-rootdisk
```

### Grace Blackwell VMI (4 GPUs + 4 IB NICs, hugepages, no EGM)

Configuration without EGM, using hugepages for vCMDQ contiguous memory. When EGM is not used, `guestMappingPassthrough` is required so the planner can build guest NUMA cells from vCPU pinning. Grace uses a 64K page kernel where 1G hugepages are not supported; use 512Mi instead.

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  annotations:
    kubevirt.io/pciHole64Size: "4294967296"  # 4 TiB in KiB
    alpha.kubevirt.io/graceVirtualization: '{"hostDevices":true,"smmuv3":true,"vcmdq":true,"egm":false}'
  name: grace-4gpu-4ib-hugepages
spec:
  architecture: arm64
  domain:
    cpu:
      cores: 60
      model: host-passthrough
      maxSockets: 2
      sockets: 2
      threads: 1
      dedicatedCpuPlacement: true
      numa:
        guestMappingPassthrough: {}
    devices:
      autoattachGraphicsDevice: false
      autoattachMemBalloon: false
      disks:
      - disk:
          bus: virtio
        name: rootdisk
      interfaces:
      - name: default
        bridge: {}
      - name: ib1
        sriov: {}
      - name: ib2
        sriov: {}
      - name: ib3
        sriov: {}
      - name: ib4
        sriov: {}
      hostDevices:
      - deviceName: nvidia.com/GB100_HGX_GB200
        name: gpu1
      - deviceName: nvidia.com/GB100_HGX_GB200
        name: gpu2
      - deviceName: nvidia.com/GB100_HGX_GB200
        name: gpu3
      - deviceName: nvidia.com/GB100_HGX_GB200
        name: gpu4
    firmware:
      bootloader:
        efi:
          secureBoot: false
    machine:
      type: virt
    memory:
      guest: 16Gi
      hugepages:
        pageSize: 512Mi # Grace-system uses 512Mi page-size
    resources:
      requests:
        hugepages-512Mi: 16Gi
        memory: 10Gi
      limits:
        hugepages-512Mi: 16Gi
        memory: 10Gi
  networks:
  - multus:
      default: true
      networkName: default/ovn-primary
    name: default
  - multus:
      networkName: tenant/ib-pf-network-1
    name: ib1
  - multus:
      networkName: tenant/ib-pf-network-1
    name: ib2
  - multus:
      networkName: tenant/ib-pf-network-1
    name: ib3
  - multus:
      networkName: tenant/ib-pf-network-1
    name: ib4
  volumes:
  - name: rootdisk
    persistentVolumeClaim:
      claimName: pvc-rootdisk
```

---

## Domain XML Example (Grace Blackwell, 120 vCPUs, 4 GPUs with EGM)

The following is a condensed version of the actual domain XML generated by KubeVirt on a Grace Blackwell system (GB200) with 120 vCPUs (2 sockets x 60 cores), 4 GPU passthrough, EGM, SMMUv3, and vCMDQ. GPUs 0-1 are on socket 0, GPUs 2-3 are on socket 1. Distance entries and GI cells 4-33 are truncated for brevity but follow the same pattern.

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <memory unit='b'>959925190656</memory>
  <vcpu placement='static'>120</vcpu>
  <memoryBacking>
    <source type='file'/>
    <access mode='shared'/>
    <allocation mode='immediate'/>
  </memoryBacking>

  <cpu mode='host-passthrough'>
    <topology sockets='2' cores='60' threads='1'/>
    <numa>
      <!-- CPU cell 0: socket 0, 60 vCPUs, EGM-backed -->
      <cell id='0' cpus='0-59' memory='468713472' unit='KiB'>
        <distances>
          <sibling id='0' value='10'/>     <!-- self -->
          <sibling id='1' value='40'/>     <!-- cross-socket CPU -->
          <sibling id='2' value='80'/>     <!-- GPU 0 GI (local) -->
          <!-- ... sibling 3-9: 80 (GPU 0 GI, local) -->
          <sibling id='10' value='80'/>    <!-- GPU 1 GI (local) -->
          <!-- ... sibling 11-17: 80 (GPU 1 GI, local) -->
          <sibling id='18' value='120'/>   <!-- GPU 2 GI (remote) -->
          <!-- ... sibling 19-33: 120 (GPU 2-3 GI, remote) -->
        </distances>
      </cell>
      <!-- CPU cell 1: socket 1, 60 vCPUs, EGM-backed -->
      <cell id='1' cpus='60-119' memory='468713472' unit='KiB'>
        <distances>
          <sibling id='0' value='40'/>     <!-- cross-socket CPU -->
          <sibling id='1' value='10'/>     <!-- self -->
          <sibling id='2' value='120'/>    <!-- GPU 0 GI (remote) -->
          <!-- ... sibling 3-17: 120 (GPU 0-1 GI, remote) -->
          <sibling id='18' value='80'/>    <!-- GPU 2 GI (local) -->
          <!-- ... sibling 19-33: 80 (GPU 2-3 GI, local) -->
        </distances>
      </cell>

      <!-- GI nodes for GPU 0 (cells 2-9, socket 0) -->
      <cell id='2' memory='0' unit='KiB'>
        <distances>
          <sibling id='0' value='80'/>     <!-- local CPU -->
          <sibling id='1' value='120'/>    <!-- remote CPU -->
          <sibling id='2' value='10'/>     <!-- self -->
          <sibling id='3' value='11'/>     <!-- same GPU group -->
          <!-- ... sibling 4-9: 11 (same GPU group) -->
          <sibling id='10' value='40'/>    <!-- GPU 1 GI -->
          <!-- ... sibling 11-33: 40 (other GPU GI nodes) -->
        </distances>
      </cell>
      <cell id='3' memory='0' unit='KiB'><!-- ... distances ... --></cell>
      <!-- ... cells 4-9: GPU 0 GI nodes -->

      <!-- GI nodes for GPU 1 (cells 10-17, socket 0) -->
      <cell id='10' memory='0' unit='KiB'>
        <distances>
          <sibling id='0' value='80'/>     <!-- local CPU (socket 0) -->
          <sibling id='1' value='120'/>    <!-- remote CPU -->
          <sibling id='2' value='40'/>     <!-- GPU 0 GI -->
          <!-- ... sibling 3-9: 40 (GPU 0 GI) -->
          <sibling id='10' value='10'/>    <!-- self -->
          <sibling id='11' value='11'/>    <!-- same GPU group -->
          <!-- ... sibling 12-17: 11 -->
          <sibling id='18' value='40'/>    <!-- GPU 2-3 GI -->
          <!-- ... sibling 19-33: 40 -->
        </distances>
      </cell>
      <!-- ... cells 11-17: GPU 1 GI nodes -->

      <!-- GI nodes for GPU 2 (cells 18-25, socket 1) -->
      <cell id='18' memory='0' unit='KiB'>
        <distances>
          <sibling id='0' value='120'/>    <!-- remote CPU -->
          <sibling id='1' value='80'/>     <!-- local CPU (socket 1) -->
          <sibling id='2' value='40'/>     <!-- GPU 0-1 GI -->
          <!-- ... sibling 3-17: 40 -->
          <sibling id='18' value='10'/>    <!-- self -->
          <sibling id='19' value='11'/>    <!-- same GPU group -->
          <!-- ... sibling 20-25: 11 -->
          <sibling id='26' value='40'/>    <!-- GPU 3 GI -->
          <!-- ... sibling 27-33: 40 -->
        </distances>
      </cell>
      <!-- ... cells 19-25: GPU 2 GI nodes -->
      <!-- ... cells 26-33: GPU 3 GI nodes (socket 1) -->
    </numa>
  </cpu>

  <devices>
    <!-- pcie-root with 4 TiB MMIO aperture -->
    <controller type='pci' index='0' model='pcie-root'>
      <pcihole64 unit='KiB'>4294967296</pcihole64>
    </controller>

    <!-- Default root port for non-GPU devices (multifunction for hotplug slots) -->
    <controller type='pci' index='1' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='1' port='0x0'/>
      <alias name='ua-default-root-port'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01'
               function='0x0' multifunction='on'/>
    </controller>

    <!-- GPU 0 (0008:01:00.0): PXB on socket 0 -->
    <controller type='pci' index='2' model='pcie-expander-bus'>
      <model name='pxb-pcie'/>
      <target busNr='33'>
        <node>0</node>
      </target>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x0a' function='0x0'/>
    </controller>
    <controller type='pci' index='3' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='2' port='0x0'/>
      <alias name='ua-numa-rp-626a37d52307'/>
      <address type='pci' domain='0x0000' bus='0x02' slot='0x00'
               function='0x0' multifunction='on'/>
    </controller>

    <!-- GPU 1 (0009:01:00.0): PXB on socket 0 -->
    <controller type='pci' index='4' model='pcie-expander-bus'>
      <model name='pxb-pcie'/>
      <target busNr='35'>
        <node>0</node>
      </target>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x0b' function='0x0'/>
    </controller>
    <controller type='pci' index='5' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='3' port='0x0'/>
      <alias name='ua-numa-rp-56e111efc912'/>
      <address type='pci' domain='0x0000' bus='0x04' slot='0x00'
               function='0x0' multifunction='on'/>
    </controller>

    <!-- GPU 2 (0018:01:00.0): PXB on socket 1 -->
    <controller type='pci' index='6' model='pcie-expander-bus'>
      <model name='pxb-pcie'/>
      <target busNr='65'>
        <node>1</node>
      </target>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x0c' function='0x0'/>
    </controller>
    <controller type='pci' index='7' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='4' port='0x0'/>
      <alias name='ua-numa-rp-3ad011c4b50a'/>
      <address type='pci' domain='0x0000' bus='0x06' slot='0x00'
               function='0x0' multifunction='on'/>
    </controller>

    <!-- GPU 3 (0019:01:00.0): PXB on socket 1 -->
    <controller type='pci' index='8' model='pcie-expander-bus'>
      <model name='pxb-pcie'/>
      <target busNr='67'>
        <node>1</node>
      </target>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x0d' function='0x0'/>
    </controller>
    <controller type='pci' index='9' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='5' port='0x0'/>
      <alias name='ua-numa-rp-69d3daf8bead'/>
      <address type='pci' domain='0x0000' bus='0x08' slot='0x00'
               function='0x0' multifunction='on'/>
    </controller>

    <!-- Root hotplug ports (index 10-16) share slot 0x01 functions 0x1-0x7
         with the default root port for virtio devices, disks, etc. -->
    <!-- ... controllers index 10-16 omitted for brevity ... -->

    <!-- SMMUv3 per GPU PXB bus (pciBus references PXB controller index) -->
    <iommu model='smmuv3'>
      <driver pciBus='2' accel='on' ats='on' ril='off'
              pasid='on' oas='48' cmdqv='on'/>
    </iommu>
    <iommu model='smmuv3'>
      <driver pciBus='4' accel='on' ats='on' ril='off'
              pasid='on' oas='48' cmdqv='on'/>
    </iommu>
    <iommu model='smmuv3'>
      <driver pciBus='6' accel='on' ats='on' ril='off'
              pasid='on' oas='48' cmdqv='on'/>
    </iommu>
    <iommu model='smmuv3'>
      <driver pciBus='8' accel='on' ats='on' ril='off'
              pasid='on' oas='48' cmdqv='on'/>
    </iommu>

    <!-- GPU 0 hostdev -->
    <hostdev type='pci' managed='no' mode='subsystem'>
      <source>
        <address domain='0x0008' bus='0x01' slot='0x00' function='0x0'/>
      </source>
      <driver iommufd='yes'/>
      <acpi nodeset='2-9'/>
      <address type='pci' domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
      <alias name='ua-hostdev0'/>
    </hostdev>

    <!-- GPU 1 hostdev -->
    <hostdev type='pci' managed='no' mode='subsystem'>
      <source>
        <address domain='0x0009' bus='0x01' slot='0x00' function='0x0'/>
      </source>
      <driver iommufd='yes'/>
      <acpi nodeset='10-17'/>
      <address type='pci' domain='0x0000' bus='0x05' slot='0x00' function='0x0'/>
      <alias name='ua-hostdev1'/>
    </hostdev>

    <!-- GPU 2 hostdev -->
    <hostdev type='pci' managed='no' mode='subsystem'>
      <source>
        <address domain='0x0018' bus='0x01' slot='0x00' function='0x0'/>
      </source>
      <driver iommufd='yes'/>
      <acpi nodeset='18-25'/>
      <address type='pci' domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
      <alias name='ua-hostdev2'/>
    </hostdev>

    <!-- GPU 3 hostdev -->
    <hostdev type='pci' managed='no' mode='subsystem'>
      <source>
        <address domain='0x0019' bus='0x01' slot='0x00' function='0x0'/>
      </source>
      <driver iommufd='yes'/>
      <acpi nodeset='26-33'/>
      <address type='pci' domain='0x0000' bus='0x09' slot='0x00' function='0x0'/>
      <alias name='ua-hostdev3'/>
    </hostdev>

    <!-- EGM memory devices (2 GPUs share /dev/egm4 on socket 0,
         2 GPUs share /dev/egm5 on socket 1; 228864 MiB per GPU) -->
    <memory model='egm' access='shared'>
      <source><path>/dev/egm4</path></source>
      <target>
        <size unit='MiB'>228864</size>
        <node>0</node>
        <pciDev>ua-hostdev0</pciDev>
      </target>
    </memory>
    <memory model='egm' access='shared'>
      <source><path>/dev/egm4</path></source>
      <target>
        <size unit='MiB'>228864</size>
        <node>0</node>
        <pciDev>ua-hostdev1</pciDev>
      </target>
    </memory>
    <memory model='egm' access='shared'>
      <source><path>/dev/egm5</path></source>
      <target>
        <size unit='MiB'>228864</size>
        <node>1</node>
        <pciDev>ua-hostdev2</pciDev>
      </target>
    </memory>
    <memory model='egm' access='shared'>
      <source><path>/dev/egm5</path></source>
      <target>
        <size unit='MiB'>228864</size>
        <node>1</node>
        <pciDev>ua-hostdev3</pciDev>
      </target>
    </memory>
  </devices>

  <!-- PCIe link speed/width on root ports (derived from host sysfs) -->
  <qemu:override>
    <qemu:device alias='ua-numa-rp-626a37d52307'>
      <qemu:frontend>
        <qemu:property name='x-speed' type='string' value='16'/>
        <qemu:property name='x-width' type='string' value='1'/>
      </qemu:frontend>
    </qemu:device>
    <qemu:device alias='ua-numa-rp-56e111efc912'>
      <qemu:frontend>
        <qemu:property name='x-speed' type='string' value='16'/>
        <qemu:property name='x-width' type='string' value='1'/>
      </qemu:frontend>
    </qemu:device>
    <qemu:device alias='ua-numa-rp-3ad011c4b50a'>
      <qemu:frontend>
        <qemu:property name='x-speed' type='string' value='16'/>
        <qemu:property name='x-width' type='string' value='1'/>
      </qemu:frontend>
    </qemu:device>
    <qemu:device alias='ua-numa-rp-69d3daf8bead'>
      <qemu:frontend>
        <qemu:property name='x-speed' type='string' value='16'/>
        <qemu:property name='x-width' type='string' value='1'/>
      </qemu:frontend>
    </qemu:device>
  </qemu:override>
</domain>
```

### Mixed GPU+NIC Topology Variant

When SR-IOV NICs (e.g., ConnectX-7 InfiniBand) are added alongside GPUs, the domain XML changes in several ways:

| Aspect | GPU-only (above) | Mixed 4 GPU + 4 NIC |
|---|---|---|
| `pcihole64` | 4 TiB (`4294967296` KiB) | 4 TiB (`4294967296` KiB) |
| PXB controllers | 4 (one per GPU) | 8 (one per GPU + one per NIC) |
| SMMUv3 instances | 4 (one per GPU bus) | 8 (one per device bus) |
| `qemu:override` entries | 4 (all Gen4 x1) | 8 (GPUs: Gen4 x1, NICs: Gen5 x16) |

NIC hostdevs differ from GPU hostdevs in the `acpi nodeset` attribute: NICs use the **CPU NUMA node** (0 or 1), not GI node ranges, because NICs do not require MIG/GI support:

```xml
<!-- NIC hostdev: ACPI nodeset references CPU NUMA node -->
<hostdev type='pci' managed='no'>
  <source>
    <address domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
  </source>
  <driver iommufd='yes'/>
  <acpi nodeset='0'/>
  <address type='pci' domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
  <alias name='ua-sriov-ib1'/>
</hostdev>

<!-- GPU hostdev: ACPI nodeset references GI node range -->
<hostdev type='pci' managed='no'>
  <source>
    <address domain='0x0008' bus='0x01' slot='0x00' function='0x0'/>
  </source>
  <driver iommufd='yes'/>
  <acpi nodeset='2-9'/>
  <address type='pci' domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
  <alias name='ua-hostdev0'/>
</hostdev>
```

The planner's bottleneck-aware link derivation (see [PCIe Link Speed and Width Modeling](#pcie-link-speed-and-width-modeling)) automatically discovers different link characteristics for each device type. NIC root-port overrides reflect the physical PCIe link (Gen5 x16), while GPU root ports use the synthetic C2C link (Gen4 x1):

```xml
<qemu:override>
  <!-- NIC root port: physical PCIe Gen5 x16 -->
  <qemu:device alias='ua-numa-rp-2aeadfd714c9'>
    <qemu:frontend>
      <qemu:property name='x-speed' type='string' value='32'/>
      <qemu:property name='x-width' type='string' value='16'/>
    </qemu:frontend>
  </qemu:device>
  <!-- GPU root port: synthetic C2C Gen4 x1 -->
  <qemu:device alias='ua-numa-rp-626a37d52307'>
    <qemu:frontend>
      <qemu:property name='x-speed' type='string' value='16'/>
      <qemu:property name='x-width' type='string' value='1'/>
    </qemu:frontend>
  </qemu:device>
</qemu:override>
```

---

## Scalability

The solution scales to:

- 4+ GPUs per tray with 8 GI NUMA nodes each (32+ guest NUMA cells for GI alone on a 4-GPU system).
- Mixed GPU+NIC topologies with independent PXB isolation per device type.
- EGM configurations with 200+ GB of GPU-accessible system memory per socket.
- Distance matrices up to 34x34 (2 CPU cells + 32 GI cells for 4 GPUs).

## Update/Rollback Compatibility

- Existing VMs without Grace annotations continue to work unchanged.
- The `GraceIOVirtualization` feature gate is independent -- disabling it removes all Grace-specific behavior without affecting generic NUMA topology from VEP 115.
- The annotation-based API is versioned with `alpha.kubevirt.io/` prefix and can be deprecated without breaking stable APIs.

## Functional Testing Approach

### Unit Tests

- NUMA PCI planner: per-GPU PXB isolation, SMMUv3 scoping, bus number allocation, chassis/port assignment.
- GI node allocation: correct ranges per GPU, NUMA cell creation, hostdev nodeset injection.
- EGM: device discovery, memory layout, hugepage rejection.
- PCIe link: sysfs reading, bottleneck derivation, QEMU property injection.
- Distance matrix: Grace-specific per-pair values, GI node grouping.
- Admission: all validation rules, dependency graph enforcement.

### End-to-End Tests

- On Grace hardware: domain XML verification, GPU driver initialization, `nvidia-smi` validation.
- MIG partition creation inside the VM.
- NCCL bandwidth benchmarks to verify correct NUMA distance reporting.
- Mixed GPU+NIC topology: verify independent link characteristics.

## Platform Prerequisites

### NVIDIA QEMU

Grace requires NVIDIA-patched QEMU (10.1.0+nvidia1egmfix) with:

- SMMUv3 nested translation support (`arm-smmuv3` device)
- vCMDQ (hardware command queue virtualization)
- vEGM (Extended GPU Memory backend)
- ACPI Generic Initiator (`acpi-generic-initiator` objects)

### NVIDIA libvirt

Grace requires NVIDIA-patched libvirt (11.9.0+nvidia4) with:

- `<iommu model='smmuv3'>` support with `pciBus` scoping
- `<driver iommufd='yes'/>` on hostdev elements
- `<acpi nodeset='...'>` on hostdev elements
- EGM memory-backend-file handling

These patches are being upstreamed to QEMU and libvirt mainline. Until upstream acceptance, they are a platform prerequisite for Grace deployments.

## Open Questions

- **NVIDIA QEMU/libvirt upstream timeline**: Track acceptance of SMMUv3 nested translation, vCMDQ, vEGM, and ACPI GI patches into QEMU and libvirt mainline.
- **API graduation**: The `alpha.kubevirt.io/graceVirtualization` annotation should be promoted to a first-class VMI spec type once the design stabilizes. The exact API shape (annotation vs. spec field) should be decided based on community feedback.
- **Multi-node NVLink systems**: GB200 NVL multi-node configurations may require additional NUMA topology modeling beyond single-tray passthrough.
- **vGPU on Grace**: Future work may extend this VEP to support Grace vGPU configurations, which have different IOMMU and memory requirements.

## Graduation Requirements

### Alpha (v1.9)

- All Grace subsystems implemented behind `GraceIOVirtualization`.
- Unit tests for all components.
- End-to-end tests on Grace Hopper and/or Grace Blackwell hardware.

### Beta

- NVIDIA QEMU/libvirt patches accepted upstream.
- Community feedback incorporated.
- Extended testing across Grace Hopper, GB200, and GB300 platforms.

### GA

- Full documentation and operational guides.
- Promote `alpha.kubevirt.io/graceVirtualization` to stable VMI spec fields.
- `GraceIOVirtualization` enabled by default on detected Grace platforms.
