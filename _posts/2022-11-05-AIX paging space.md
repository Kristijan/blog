---
title: "AIX paging space"
date: "2022-11-06"
categories:
  - "aix"
tags:
  - "aix"
  - "paging"
---

An AIX installation will come with a default configured paging space logical volume (`hd6`). You can increase the size of the logical volume to suit your needs, but at some stage, you may have a requirement to either expand the logical volume greater than the size of the rootvg, or more than 64GB (the current maximum size of a paging logical volume). There are a number of best practises when it comes to paging space on AIX.

1. Don't span paging logical volumes across different disks.
2. Create additional paging logical volumes of equal size. As paging space is allocated in a round-robin type manner, smaller paging spaces will fill up quicker than larger ones.
3. If you can't increase the default paging logical volume `hd6` to be equal size of all other additional paging logical volumes, deactivate it.

As a rule of thumb, I generally try and keep my `rootvg` volume groups relatively lean. I've been bitten way too often with failed [multibos](https://www.ibm.com/docs/en/aix/7.2?topic=m-multibos-command){:target="_blank"} creations due to insufficient free capacity in the `rootvg` volume group. More often than not, this is because someone has increased the size of the `hd6` paging space. Depending on the size of your `rootvg`, you may wish to impose a maximum size limit on the `hd6` paging logical volume. Any requirements for an increase to (or additional) paging space, should be created on a separate volume.

If I have such a requirement for additional paging space (say the largest possible paging size of 64GB), I will create a separate volume group (for example, `pagingvg`) and allocate LUN's of 64GB in size. Each paging logical volume will consume the entire LUN, and I will deactivate `hd6` (see example below).

```terminal
# lsps -a
Page Space      Physical Volume   Volume Group    Size %Used   Active    Auto    Type   Chksum
paging01        hdisk2            pagingvg     61344MB     0     yes     yes      lv       0
paging00        hdisk1            pagingvg     61344MB     0     yes     yes      lv       0
hd6             hdisk0            rootvg        8192MB     0      no     yes      lv       0
# lspv -l hdisk1
hdisk1:
LV NAME               LPs     PPs     DISTRIBUTION          MOUNT POINT
paging00              1917    1917    384..383..383..383..384 N/A
# lspv -l hdisk2
hdisk2:
LV NAME               LPs     PPs     DISTRIBUTION          MOUNT POINT
paging01              1917    1917    384..383..383..383..384 N/A
```

I then have the below script that runs at boot. It will deactivate the default paging device (`hd6`) if an alternate paging space is active, and greater in size.

```bash
#!/bin/ksh
#
# Script      : deactivate_paging.sh
#
# Description : Script runs at boot and will deactivate the default
#               AIX paging device (hd6) if an alternate paging space is
#               active, and greater in size.
#
# Usage       : Script takes no parameters.

if lsps -ac | grep -Evq '^#|^hd6'; then
  # Get size of default paging space
  hd6_size=$(lsps -a | awk '/^hd6/{ print $4 }')

  # Create array of other paging space attributes
  set -A paging_name $(lsps -a | awk '!/^hd6/ && (NR!=1){ print $1 }')
  set -A paging_size $(lsps -a | awk '!/^hd6/ && (NR!=1){ print $4 }')
  set -A paging_active $(lsps -a | awk '!/^hd6/ && (NR!=1){ print $6 }')

  # Default paging space (hd6) will be turned off if any single
  # alternate paging space is active, and greater in size than
  # the default paging space
  count=0
  while [[ "${count}" -lt "${#paging_name[*]}" ]]; do
    if [[ ( "${paging_active[$count]}" = "yes" ) && ( "${paging_size[$count]%??}" -gt "${hd6_size%??}" ) ]]; then
      echo "At least one alternate paging space detected and active [${paging_name[$count]}]" > /dev/console
      echo 'Deactivating default paging space hd6...' > /dev/console
      swapoff /dev/hd6 > /dev/console
      exit
    else
      (( count+=1 ))
    fi
  done
fi
```
{: file="/etc/rc.d/init.d/deactivate_paging.sh" }

Console output during system boot.

```terminal
# alog -t console -o
...
...
...
         0 Mon Nov  7 10:23:14 AEDT 2022 At least one alternate paging space detected and active [paging00]
         0 Mon Nov  7 10:23:14 AEDT 2022 Deactivating default paging space hd6...
```

You can find some good information regarding paging space placement best practices on IBM's website - [https://www.ibm.com/support/pages/paging-space-placement-best-practices](https://www.ibm.com/support/pages/paging-space-placement-best-practices){:target="_blank"}
