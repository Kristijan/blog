---
title: "AIX boot hangs with HMC 2700 LED code"
date: "2013-08-24"
categories: 
  - "aix"
tags: 
  - "2700"
  - "aix"
  - "boot"
  - "hang"
  - "hmc"
---

We recently upgraded the firmare on our Power frame, which required shutting down some of our AIX LPAR's. The firmware upgrade went well, as did starting up all the AIX LPAR's, except for one. This particular LPAR booted to HMC LED code 2700 and hung there. I restarted the partition to the Open Firmware (OF) prompt, and tried booting again using verbose mode to see where the boot process was hanging.

```console
----------------
Attempting to configure device 'fscsi6'
 cfgmgr Time: 0        LEDS: 0x2700
Invoking /usr/lib/methods/cfgefscsi -1 -l fscsi6
Number of running methods: 4
----------------
Completed method for: fcs7, Elapsed time = 0
Return code = 0
***** stdout *****
fscsi7
*** no stderr ****
----------------
Time: 0         LEDS: 0x2700 for fscsi6
Number of running methods: 3
 cfgmgr exec(/bin/sh,-c,/usr/lib/methods/cfgefscsi -1 -l fscsi6){135238,102466}
----------------
Attempting to configure device 'fscsi7'
 cfgmgr Time: 0        LEDS: 0x2700
Invoking /usr/lib/methods/cfgefscsi -1 -l fscsi7
exec(/usr/lib/methods/cfgefscsi,-1,-l,fscsi6){135238,102466}
Number of running methods: 4
exec(/bin/sh,-c,/usr/lib/methods/cfgefscsi -1 -l fscsi7){122944,102466}
exec(/usr/lib/methods/cfgefscsi,-1,-l,fscsi7){122944,102466}
----------------
Completed method for: fscsi7, Elapsed time = 0
Return code = 0
*** no stdout ****
*** no stderr ****
----------------
Time: 0         LEDS: 0x2700 for fscsi6
Number of running methods: 3
 cfgmgr ----------------
Completed method for: fscsi5, Elapsed time = 0
Return code = 0
*** no stdout ****
*** no stderr ****
----------------
Time: 0         LEDS: 0x2700 for fscsi6
Number of running methods: 2
 cfgmgr
```

To verbose boot from OF, you can do the following from the HMC.

```console
$ chsysstate -r lpar -m <frame> -o on -n -b of -f -p <lpar>
$ mkvterm -m <frame> -p <lpar>
0> boot -s verbose
```

From the output, we can see that the virtual fibre channel adapter scans for visible LUN's presented through the adapter and runs `cfgmgr` against them. In this instance, the `cfgmgr` process is hanging, hence causing the HMC LED 2700 code. This particular LPAR has LUN's presented to it from EMC storage through VIOS via NPIV, some of which had a state of NR (Not Ready).

Note that I've deliberately masked the Symmetrix and LUN ID's.

```console
Name: aix_lpar1
 
   Symmetrix ID       : XXXXXXXXXX
   Last updated at    : Mon Mar 18 16:20:54 2013
   Masking Views      : Yes
   FAST Policy        : Yes
 
   Devices (62):
 
    {
    ---------------------------------------------------------
    Sym                             Device               Cap
    Dev    Pdev Name                Config        Sts    (MB)
    ---------------------------------------------------------
    XXXX   N/A                      RDF1+TDEV      NR   16386
    XXXX   N/A                      RDF1+TDEV      RW   20481
    XXXX   N/A                      RDF1+TDEV      RW   51203
    XXXX   N/A                      RDF1+TDEV      RW    8633
    XXXX   N/A                      RDF1+TDEV      NR   65537
    XXXX   N/A                      RDF1+TDEV      RW   79873
    XXXX   N/A                      RDF1+TDEV      RW   79873
    XXXX   N/A                      RDF1+TDEV      NR   71681
    XXXX   N/A                      RDF1+TDEV      NR   61440
    XXXX   N/A                      RDF1+TDEV      NR  131074
    XXXX   N/A                      RDF1+TDEV      NR  153600
    XXXX   N/A                      RDF1+TDEV      NR  212993
    XXXX   N/A                      RDF1+TDEV      NR  311303
    XXXX   N/A                      RDF1+TDEV      NR   16385
    XXXX   N/A                      RDF1+TDEV      NR   16385
    XXXX   N/A                      RDF1+TDEV      NR   16385
    XXXX   N/A                      RDF1+TDEV      NR   16385
    XXXX   N/A                      RDF1+TDEV      NR   16385
    ...
    ...
```

The LUN's masked to this host have a clone configured from another AIX LPAR, which at some point resulted in the target LUN's (our HMC LED 2700 host) being placed in a NR state. Resyncing the target LUN's from the source put them back into the correct state, and allowed the AIX LPAR to boot successfully. During investigation, a PMR was raised with IBM, which eventually came to the same conclusion that EMC LUN's in an NR state will cause an AIX LPAR to hang during boot. The `cfgmgr` operation will eventually fail the LUNs, and the LPAR will boot, but in our instance, with so many LUN's in an NR state, failures (timeouts) across all LUNs wouldn't have happened for many hours.

Preventing this from occuring in the future would be to verify from the storage end that the LUNs are in a ready state prior to boot, or to place all those LUNs in a 'Defined' state in AIX before shutting down. There wasn't much information when I looked for HMC LED 2700 hangs, so hopefully this helps someone with an AIX environment and EMC storage having the same symptoms.
