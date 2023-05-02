---
layout: page
---

# NAME

daxctl-reconfigure-device - Reconfigure a dax device into a different
mode

# SYNOPSIS

>     daxctl reconfigure-device <dax0.0> [<dax1.0>…​<daxY.Z>] [<options>]

# DESCRIPTION

Reconfigure the operational mode of a dax device. This can be used to
convert a regular *devdax* mode device to the *system-ram* mode which
arranges for the dax range to be hot-plugged into the system as regular
memory.

<div class="note">

This is a destructive operation. Any data on the dax device **will** be
lost.

</div>

<div class="note">

Device reconfiguration depends on the dax-bus device model. See
[daxctl-migrate-device-model](daxctl-migrate-device-model) for more information. If
dax-class is in use (via the dax_pmem_compat driver), the
reconfiguration will fail with an error such as the following:

</div>

    # daxctl reconfigure-device --mode=system-ram --region=0 all
    libdaxctl: daxctl_dev_disable: dax3.0: error: device model is dax-class
    dax3.0: disable failed: Operation not supported
    error reconfiguring devices: Operation not supported
    reconfigured 0 devices

*daxctl-reconfigure-device* nominally expects that it will online new
memory blocks as *movable*, so that kernel data doesn’t make it into
this memory. However, there are other potential agents that may be
configured to automatically online new hot-plugged memory as it appears.
Most notably, these are the
*/sys/devices/system/memory/auto_online_blocks* configuration, or system
udev rules. If such an agent races to online memory sections, daxctl
checks if the blocks were onlined as *movable* memory. If this was not
the case, and the memory blocks are found to be in a different zone,
then a warning is displayed. If it is desired that a different agent
control the onlining of memory blocks, and the associated memory zone,
then it is recommended to use the --no-online option described below.
This will abridge the device reconfiguration operation to just
hotplugging the memory, and refrain from then onlining it.

In case daxctl detects that there is a kernel policy to auto-online
blocks (via /sys/devices/system/memory/auto_online_blocks), then
reconfiguring to system-ram will result in a failure. This can be
overridden with *--force*.

# THEORY OF OPERATION

The kernel device-dax subsystem surfaces character devices that provide
DAX-access (direct mappings sans page-cache buffering) to a given memory
region. The devices are named /dev/daxX.Y where X is a region-id and Y
is an instance-id within that region. There are 2 mechanisms that
trigger device-dax instances to appear:

1.  Persistent Memory (PMEM) namespace configured in "devdax" mode. See
    "ndctl create-namspace --help" and
    [CONFIG_DEV_DAX_PMEM](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/dax/Kconfig).
    In this case the device-dax instance is statically sized to its host
    memory region which is bounded to the physical address range of the
    host namespace.

2.  Soft Reserved memory enumerated by platform firmware. On EFI systems
    this is communicated via the so called EFI_MEMORY_SP "Special
    Purpose" attribute. See
    [CONFIG_DEV_DAX_HMEM](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/dax/Kconfig).
    In this case the device-dax instance(s) associated with the given
    memory region can be resized and divided into multiple devices.

In the Soft Reservation case the expectation for EFI + ACPI based
platforms is that in addition to the EFI_MEMORY_SP attribute the
firmware also creates distinct ACPI proximity domains for any address
range that has different performance characteristics than default
"System RAM". So, the SRAT will define the proximity domain, the SLIT
communicates relative distance to other proximity domains, and the HMAT
is populated with nominal read/write latency and read/write bandwidth
data. That HMAT data is emitted to the kernel log on bootup, and also
exported to sysfs. See
[NUMAPERF](https://www.kernel.org/doc/html/latest/admin-guide/mm/numaperf.html),
for the runtime representation of CPU to Memory node performance
details.

Outside of the NUMA performance details linked above the other method to
detect the presence of "Soft Reserved" memory is to dump /proc/iomem and
look for "Soft Reserved" ranges. If the kernel was not built with
CONFIG_EFI_SOFT_RESERVE, predates the introduction of
CONFIG_EFI_SOFT_RESERVE (v5.5), or was booted with the efi=nosoftreserve
command line then device-dax will not attach and the expectation is that
the memory shows up as a memory-only NUMA node. Otherwise the memory
shows up as a device-dax instance and DAXCTL(1) can be used to
optionally partition it and assign the memory back to the kernel as
"System RAM", or the device can be mapped directly as the back end of a
userspace memory allocator like
[LIBVMEM](https://pmem.io/vmem/libvmem/).

# EXAMPLES

- Reconfigure dax0.0 to system-ram mode, don’t online the memory

<!-- -->

    # daxctl reconfigure-device --mode=system-ram --no-online dax0.0
    [
      {
        "chardev":"dax0.0",
        "size":16777216000,
        "target_node":2,
        "mode":"system-ram"
      }
    ]

- Reconfigure dax0.0 to devdax mode, attempt to offline the memory

<!-- -->

    # daxctl reconfigure-device --human --mode=devdax --force dax0.0
    {
      "chardev":"dax0.0",
      "size":"15.63 GiB (16.78 GB)",
      "target_node":2,
      "mode":"devdax"
    }

- Reconfigure all dax devices on region0 to system-ram mode

<!-- -->

    # daxctl reconfigure-device --mode=system-ram --region=0 all
    [
      {
        "chardev":"dax0.0",
        "size":16777216000,
        "target_node":2,
        "mode":"system-ram"
      },
      {
        "chardev":"dax0.1",
        "size":16777216000,
        "target_node":3,
        "mode":"system-ram"
      }
    ]

- Run a process called *some-service* using numactl to restrict its cpu
  nodes to *0* and *1*, and memory allocations to node 2 (determined
  using daxctl_dev_get_target_node() or *daxctl list*)

<!-- -->

    # daxctl reconfigure-device --mode=system-ram dax0.0
    [
      {
        "chardev":"dax0.0",
        "size":16777216000,
        "target_node":2,
        "mode":"system-ram"
      }
    ]

    # numactl --cpunodebind=0-1 --membind=2 -- some-service --opt1 --opt2

- Change the size of a dax device

<!-- -->

    # daxctl reconfigure-device dax0.1 -s 16G
    reconfigured 1 device
    # daxctl reconfigure-device dax0.1 -s 0
    reconfigured 1 device

# OPTIONS

`-r; --region=`  
Restrict the operation to devices belonging to the specified region(s).
A device-dax region is a contiguous range of memory that hosts one or
more /dev/daxX.Y devices, where X is the region id and Y is the device
instance id.

`-s; --size=`  
For regions that support dax device creation, change the device size in
bytes. This option supports the suffixes "k" or "K" for KiB, "m" or "M"
for MiB, "g" or "G" for GiB and "t" or "T" for TiB.

    The size must be a multiple of the region alignment.

    This option is mutually exclusive with -m or --mode.

`-a; --align`  
Applications that want to establish dax memory mappings with page table
entries greater than system base page size (4K on x86) need a device
that is sufficiently aligned. This defaults to 2M. Note that "devdax"
mode enforces all mappings to be aligned to this value, i.e. it fails
unaligned mapping attempts.

    This option is mutually exclusive with -m or --mode.

`-m; --mode=`  
Specify the mode to which the dax device(s) should be reconfigured.

- "system-ram": hotplug the device into system memory.

- "devdax": switch to the normal "device dax" mode. This requires the
  kernel to support hot-unplugging *kmem* based memory. If this is not
  available, a reboot is the only way to switch back to *devdax* mode.

`-N; --no-online`  
By default, memory sections provided by system-ram devices will be
brought online automatically and immediately with the *online_movable*
policy. Use this option to disable the automatic onlining behavior.

`-C; --check-config`  
Get reconfiguration parameters from the global daxctl config file. This
is typically used when daxctl-reconfigure-device is called from a
systemd-udevd device unit file. The reconfiguration proceeds only if the
match parameters in a *reconfigure-device* section of the config match
the dax device specified on the command line. See the *PERSISTENT
RECONFIGURATION* section for more details.

<!-- -->

`--no-movable`  
*--movable* is the default. This can be overridden to online new memory
such that it is not *movable*. This allows any allocation to potentially
be served from this memory. This may preclude subsequent removal. With
the *--movable* behavior (which is default), kernel allocations will not
consider this memory, and it will be reserved for application use.

`-f; --force`  
- When converting from "system-ram" mode to "devdax", it is expected
  that all the memory sections are first made offline. By default,
  daxctl won’t touch online memory. However with this option, attempt to
  offline the memory on the NUMA node associated with the dax device
  before converting it back to "devdax" mode.

- Additionally, if a kernel policy to auto-online blocks is detected,
  reconfiguration to system-ram fails. With this option, the failure can
  be overridden to allow reconfiguration regardless of kernel policy.
  Doing this may result in a successful reconfiguration, but it may not
  be possible to subsequently offline the memory without a reboot.

<!-- -->

`-u; --human`  
By default the command will output machine-friendly raw-integer data.
Instead, with this flag, numbers representing storage size will be
formatted as human readable strings with units, other fields are
converted to hexadecimal strings.

<!-- -->

`-v; --verbose`  
Emit more debug messages

# PERSISTENT RECONFIGURATION

The *mode* of a daxctl device is not persistent across reboots by
default. This is because the device itself does not hold any metadata
that hints at what mode it was set to, or is intended to be used. The
default mode for such a device on boot is *devdax*.

The administrator may set policy such that certain dax devices are
always reconfigured into a target configuration every boot. This is
accomplished via a daxctl config file.

The config file may have multiple sections influencing different aspects
of daxctl operation. The section of interest for persistent
reconfiguration is *reconfigure-device*. The format of this is as
follows:

    [reconfigure-device <unique_subsection_name>]
    nvdimm.uuid = <NVDIMM namespace uuid>
    mode = <desired reconfiguration mode> (default: system-ram)
    online = <true|false> (default: true)
    movable = <true|false> (default: true)

Here is an example of a config snippet for managing three devdax
namespaces, one is left in devdax mode, the second is changed to
system-ram mode with default options (online, movable), and the third is
set to system-ram mode, the memory is onlined, but not movable.

Note that the *subsection name* can be arbitrary, and is only used to
identify a specific config section. It does not have to match the
*device name* (e.g. *dax0.0* etc).

    [reconfigure-device dax0]
    nvdimm.uuid = ed93e918-e165-49d8-921d-383d7b9660c5
    mode = devdax

    [reconfigure-device dax1]
    nvdimm.uuid = f36d02ff-1d9f-4fb9-a5b9-8ceb10a00fe3
    mode = system-ram

    [reconfigure-device dax2]
    nvdimm.uuid = f36d02ff-1d9f-4fb9-a5b9-8ceb10a00fe3
    mode = system-ram
    online = true
    movable = false

The following example can be used to create a devdax mode namespace, and
simultaneously add the newly created namespace to the config file for
system-ram conversion.

    ndctl create-namespace --mode=devdax | \
        jq -r "\"[reconfigure-device $(uuidgen)]\", \"nvdimm.uuid = \(.uuid)\", \"mode = system-ram\"" >> $config_path

The default location for daxctl config files is under {daxctl_confdir}/,
and any file with a *.conf* suffix at this location is considered. It is
acceptable to have multiple files containing ini-style config sections,
but the {section, subsection} tuple must be unique across all config
files under {daxctl_confdir}/.

# COPYRIGHT

Copyright © 2016 - 2022, Intel Corporation. License GPLv2: GNU GPL
version 2 <http://gnu.org/licenses/gpl.html>. This is free software: you
are free to change and redistribute it. There is NO WARRANTY, to the
extent permitted by law.

# SEE ALSO

[daxctl-list](daxctl-list),[daxctl-migrate-device-model](daxctl-migrate-device-model)
