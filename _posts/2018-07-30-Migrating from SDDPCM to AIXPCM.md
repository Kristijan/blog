---
title: "Migrating from SDDPCM to AIXPCM (the easy way)"
date: "2018-07-30"
categories: 
  - "aix"
tags: 
  - "aix"
  - "aixpcm"
  - "sddpcm"
  - "storage"
---

For a while now, IBM have diverted their efforts in storage multipathing from SDDPCM to the [default AIX PCM](https://www-01.ibm.com/support/docview.wss?uid=ssg1S1010218){:target="_blank"}. This brings a few advantages, but specifically for me, it means the driver is now updated as part of the AIX system maintenance, and is no longer something I need to maintain separately. All significant functionality that SDDPCM provides, can now be provided by the default AIX PCM driver. Additionally, SDDPCM is not supported on POWER 9 hardware.

For those currently using SDDPCM, removing the driver can be somewhat complicated, and even more so when the boot LUN (rootvg LUN) is being managed by SDDPCM. Buried deep in the Multipath Subsystem Device Driver Users Guide is a command called `manage_disk_drivers`. The `manage_disk_drivers` command can be used to display a list of storage families, and the driver that manages or supports each family. The command also allows us to easily switch (with a reboot if you boot from an SDDPCM managed device) the driver from SDDPCM to AIXPCM (or vice versa). Below I detail how to switch from SDDPCM to AIXPCM when using LUN's presented to the host via SVC.

## Existing driver manging IBMSVC

From the man page for `manage_disk_drivers`

> If the present driver attribute is set to NO_OVERRIDE, the AIX operating system selects an alternate path control module (PCM), such as Subsystem Device Driver Path Control Module (SDDPCM), if it is installed.

```terminal
# manage_disk_drivers -l
Device              Present Driver        Driver Options
2810XIV             AIX_AAPCM             AIX_AAPCM,AIX_non_MPIO
DS4100              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS4200              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS4300              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS4500              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS4700              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS4800              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS3950              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS5020              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DCS3700             AIX_APPCM             AIX_APPCM
DCS3860             AIX_APPCM             AIX_APPCM
DS5100/DS5300       AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS3500              AIX_APPCM             AIX_APPCM
XIVCTRL             MPIO_XIVCTRL          MPIO_XIVCTRL,nonMPIO_XIVCTRL
2107DS8K            NO_OVERRIDE           NO_OVERRIDE,AIX_AAPCM,AIX_non_MPIO
IBMFlash            NO_OVERRIDE           NO_OVERRIDE,AIX_AAPCM,AIX_non_MPIO
IBMSVC              NO_OVERRIDE           NO_OVERRIDE,AIX_AAPCM,AIX_non_MPIO
```

## Switch to using AIXPCM

```terminal
# manage_disk_drivers -d IBMSVC -o AIX_AAPCM
 ********************** ATTENTION *************************
  For the change to take effect the system must be rebooted
```

After the reboot, you will now see AIX_AAPCM as the present driver being used.

```terminal
# manage_disk_drivers -l
Device              Present Driver        Driver Options
2810XIV             AIX_AAPCM             AIX_AAPCM,AIX_non_MPIO
DS4100              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS4200              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS4300              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS4500              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS4700              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS4800              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS3950              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS5020              AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DCS3700             AIX_APPCM             AIX_APPCM
DCS3860             AIX_APPCM             AIX_APPCM
DS5100/DS5300       AIX_SDDAPPCM          AIX_APPCM,AIX_SDDAPPCM
DS3500              AIX_APPCM             AIX_APPCM
XIVCTRL             MPIO_XIVCTRL          MPIO_XIVCTRL,nonMPIO_XIVCTRL
2107DS8K            NO_OVERRIDE           NO_OVERRIDE,AIX_AAPCM,AIX_non_MPIO
IBMFlash            NO_OVERRIDE           NO_OVERRIDE,AIX_AAPCM,AIX_non_MPIO
IBMSVC              AIX_AAPCM             NO_OVERRIDE,AIX_AAPCM,AIX_non_MPIO
```

From here, you can do one of two things. Leave the SDDPCM driver installed, as this will allow for easy rollback should you experience performance issues, or other driver related problems. Or completely remove the SDDPCM driver from the LPAR.

A few things to keep in mind.

- If you've modified the `queue_depth` attribute on the hdisk, this will be reset to the AIXPCM default of 20. There is already a [good write up](https://www.ibm.com/developerworks/aix/library/au-aix-mpio/index.html){:target="_blank"} on best practises and considerations when working with the default AIXPCM driver.
- When using SDDPCM, you would have used the `pcmpath` command to display and manage devices. As above, someone has already written a [good write up](https://www.ibm.com/developerworks/aix/library/au-aix-multipath-io-mpio/index.html){:target="_blank"} on resiliency and problem determination, and some common `lsmpio` commands you'll want to know.

Taking the IBM recommendations into account, I'll set the hdisk attributes accordingly.

```terminal
# lsdev -Cc disk -F name | while read -r hdisk; do chdev -l ${hdisk} -a queue_depth=32 -a reserve_policy=no_reserve -a algorithm=shortest_queue -P; done
hdisk0 changed
hdisk1 changed
```

Another handy command, which isn't related to the overall driver migration, is using `chdef` to change the default values of the predefined attributes in ODM. Any future LUN's presented to the host will now have the `queue_depth`, `reserve_policy`, and `algorithm` set to the values I want.

```terminal
# chdef -a queue_depth=32 -c disk -s fcp -t mpioosdisk
queue_depth changed
# chdef -a reserve_policy=no_reserve -c disk -s fcp -t mpioosdisk
reserve_policy changed
# chdef -a algorithm=shortest_queue -c disk -s fcp -t mpioosdisk
algorithm changed
```

## Rollback

Should you need to go back to using SDDPCM as the driver, and haven't removed it, you can use `manage_disk_drivers` to flip back and reboot.

```terminal
# manage_disk_drivers -d IBMSVC -o NO_OVERRIDE
 ********************** ATTENTION *************************
 For the change to take effect the system must be rebooted
```
