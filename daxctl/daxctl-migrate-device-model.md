---
layout: page
---

# NAME

daxctl-migrate-device-model - Opt-in to the /sys/bus/dax device-model,
allow for alternative Device-DAX instance drivers.

# SYNOPSIS

>     daxctl migrate-device-model

Arrange for modprobe to disable the dax_pmem_compat, if present, and
instead deploy the dax_pmem module to convert to the /sys/bus/dax model.
Kernel versions prior to v5.1 may not support /sys/bus/dax in which case
the result of this command is a nop until the kernel is updated. The
motivation for changing from /sys/class/dax to /sys/bus/dax is to allow
for alternative drivers for Device-DAX instances, in particular the
dax_kmem driver.

By default device-dax publishes a /dev/daxX.Y character device for
userspace to directly map performance differentiated memory. This is
fine if the memory is to be exclusively consumed / managed by userspace.
Alternatively an administrator may want the kernel to manage the memory,
make it available via malloc(), allow for over-provisioning, and / or
apply kernel-based resource control schemes to the memory. In that case
the memory fronted by a given Device-DAX instance can be assigned to the
dax_kmem driver which arranges for the core-kernel memory-management
sub-system to assume management of the memory range.

This behavior is opt-in for consideration of existing applications /
scripts that may be hard coded to use /sys/class/dax. Fixes have been
submitted to applications known to have these direct dependencies
[FIO](http://git.kernel.dk/cgit/fio/commit/?id=b08e7d6b18b4)
[PMDK](https://github.com/pmem/pmdk/commit/91bc8620884e), however, there
may be others and a system-owner should be aware of the potential for
regression of Device-DAX consuming scripts, applications, or older
daxctl binaries.

The modprobe policy established by this utility becomes effective after
the next reboot, or after all DAX related modules have been removed and
re-loaded with "udevadm trigger"

# COPYRIGHT

Copyright © 2016 - 2022, Intel Corporation. License GPLv2: GNU GPL
version 2 <http://gnu.org/licenses/gpl.html>. This is free software: you
are free to change and redistribute it. There is NO WARRANTY, to the
extent permitted by law.
