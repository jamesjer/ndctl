---
title: ndctl
layout: pmdk
---

# NAME

ndctl-activate-firmware - activate staged firmware on memory devices

# SYNOPSIS

>     ndctl activate-firmware [<bus-id> <bus-id2> …​ <bus-idN>] [<options>]

Some persistent memory devices run a firmware locally on the device /
"DIMM" to perform tasks like media management, capacity provisioning,
and health monitoring. The process of updating that firmware typically
involves a reboot because it has implications for in-flight memory
transactions. However, reboots can be costly for systems that can not
tolerate extended downtime.

The kernel detects platforms that expose support for
runtime-firmware-activation (FWA). The *ndctl update-firmware* stages
new firmware binaries, but if the platform supports FWA it will
additionally arm the devices for activation. Then *ndctl
activate-firmware* may attempt to activate the firmware live. However,
if the platform indicates that the memory controller will be taken
off-line for the duration of the update "activate_method == suspend"
then the default policy for firmware activation is to inject a truncated
hibernate cycle to freeze devices and applications before the hard
quiesce is injected by the platform, and then resume the system.

**DANGER** the activate-firmware command includes a --force option to
tell the driver bypass the hibernation cycle and perform the update
"live". I.e. it arranges for applications and devices to race the
platform injected quiesce period. This option should only be used
explicit knowledge that the platform quiesce time will not trigger
completion timeout violations for any devices in the system.

# EXAMPLES

Check for any buses that support activation without triggering an
activation:

    # ndctl activate-firmware all --dry-run
    ACPI.NFIT: ndbus1: has no devices that support firmware update.
    nfit_test.1: ndbus3: has no devices that support firmware update.
    e820: ndbus0: has no devices that support firmware update.
    [
      {
        "provider":"nfit_test.0",
        "dev":"ndbus1",
        "scrub_state":"idle",
        "firmware":{
          "activate_method":"suspend",
          "activate_state":"idle"
        },
        "dimms":[
          {
    ...

Check that a specific bus supports activation without performing an
activation:

    # ndctl activate-firmware nfit_test.0 --dry-run --force
    [
      {
        "provider":"nfit_test.0",
        "dev":"ndbus2",
        "scrub_state":"idle",
        "firmware":{
          "activate_method":"suspend",
          "activate_state":"idle"
        },
        "dimms":[
    ...
    ]

The result is equivalent to *ndctl list -BFDu* upon successful
activation.

The *ndctl list* command can also enumerate the default activation
method:

    # ndctl list -b nfit_test.0 -BF
    [
      {
        "provider":"nfit_test.0",
        "dev":"ndbus2",
        "scrub_state":"idle",
        "firmware":{
          "activate_method":"suspend",
          "activate_state":"idle"
        }
      }
    ]

# OPTIONS

`-n; --dry-run`  
Perform all actions related to activation including honoring --idle and
--force, but skip the final execution of the activation. The overrides
are undone before the command completes. Any failed overrides will be
reported as error messages.

`-I; --idle`  
Implied by default, this option controls whether the platform will
attempt to increase the completion timeout of all devices in the system
and validate that the max completion timeout satisfies the time needed
to perform the activation. This validation step can be overridden by
specifying --no-idle.

`-f; --force`  
The activation method defaults to the reported
"bus.firmware.activate_method" property. When the method is "live" then
this --force option is ignored. When the method is "reset" no runtime
activation is attempted. When the method is "suspend" this option
indicates to the driver to bypass the hibernate cycle to activate
firmware. in the bus When the reported "activate_method" is "suspend"
the kernel driver may support overriding the suspend requirement and
instead issue the firmware-activation live. **CAUTION** this may lead to
undefined system behavior if device completion timeouts are violated for
in-flight memory operations.

`-v; --verbose`  
Emit debug messages for the firmware activation procedure

# COPYRIGHT

Copyright © 2016 - 2022, Intel Corporation. License GPLv2: GNU GPL
version 2 <http://gnu.org/licenses/gpl.html>. This is free software: you
are free to change and redistribute it. There is NO WARRANTY, to the
extent permitted by law.

# SEE ALSO

[ndctl-update-firmware](ndctl-update-firmware.md) , [Intel Optane PMem DSM
Interface](https://pmem.io/documents/IntelOptanePMem_DSM_Interface-V2.0.pdf)
