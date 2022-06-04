---
title: "HMC Elastic CoD detail function"
date: "2015-11-19"
categories: 
  - "hmc"
tags: 
  - "cod"
  - "elastic-capacity-on-demand"
  - "hmc"
  - "scripting"
  - "shell"
  - "shell-scripting"
---

IBM have created a [self-service portal](https://www-304.ibm.com/support/customercare/ss/escod/home){:target="_blank"} for its customers to allow them to request their own Elastic Capacity on Demand codes for their registered systems. In my time using the new website, the codes have been generated and sent to me via email on average between 30 & 45 minutes. This significantly reduces not only the time taken to get new codes posted to the [POD website](http://www-912.ibm.com/pod/pod){:target="_blank"}, but also eliminates the process of having to reach out to your IBM representative to request them from the COD office in the USA. The website does require a number of fields to be filled in for the code to be generated automatically:

- System type
- System serial number
- Anchor card CCIN
- Anchor card serial number
- Anchor card unique identifier
- Resource identifier
- Activate resources
- Sequence number
- Entry check

All this information can be gathered from either the HMC GUI or the CLI. Preferring to work on the CLI, I've written a function (inspiration from Brian Smith) that collects and generates all the data the website requires in a nice to read format (well, nicer to read than what's presented via the `lscod` command).

```console
kristijan@hmc:~> source onoffdetails
 
kristijan@hmc:~> onoffdetails
Usage: onoffdetails [managed system] [mem | proc]
 
kristijan@hmc:~> onoffdetails 9119-FHB-SN1234567 mem
 
Memory details for 9119-FHB-SN1234567
 
                   System type : 9119
          System serial number : 12-34567
              Anchor card CCIN : 52C4
     Anchor card serial number : 00-236D000
 Anchor card unique identifier : 316405567831037C
           Resource identifier : D973
            Activate resources : 0540
               Sequence number : 0046
                   Entry check : A3
 
kristijan@hmc:~> onoffdetails 9119-FHB-SN1234567 proc
 
Processor details for 9119-FHB-SN1234567
 
                   System type : 9119
          System serial number : 12-34567
              Anchor card CCIN : 52C4
     Anchor card serial number : 00-236D000
 Anchor card unique identifier : 316405567831037C
           Resource identifier : D971
            Activate resources : 0450
               Sequence number : 0045
                   Entry check : A1
```

Function code is below, you just need to copy it over to the HMC, and then source it after you've logged in.

```bash
# Script: onoffdetails
#
# Usage: Gathers all the details required to self request On/Off
#        Capacity on Demand from IBM's self-service portal.
#        (https://www-304.ibm.com/support/customercare/ss/escod/home)
#
# Change log:
#    Date       Who                   Comment
#    ----       --                    -------
#    18/11/15   Kristian Milos        Initial write.
#
onoffdetails () {
if [ $# -lt 2 ]; then
       printf "Usage: onoffdetails [managed system] [mem | proc]\n"
else
       COUNT=0
       ERROR=0
       while read SYSTEM; do
              if [ "${SYSTEM}" = "$1" ] ; then
                     COUNT=$((COUNT +1))
              fi
       done < <(lssyscfg -r sys -F "name")
       if [ ${COUNT} -eq 0 ]; then
              printf "\nERROR:  System not found: $1\n\n"
              printf "Known managed systems:\n"
              lssyscfg -r sys -F name
              ERROR=1
       fi
 
       if [[ $2 = "mem" || $2 = "proc" && ${ERROR} != 1 ]]; then
              SYSTEM=$1
              RESOURCE=$2
 
              sys_type=$(lscod -m ${SYSTEM} -t code -r ${RESOURCE} -c onoff -F sys_type)
              sys_serial_num=$(lscod -m ${SYSTEM} -t code -r ${RESOURCE} -c onoff -F sys_serial_num)
              anchor_card_ccin=$(lscod -m ${SYSTEM} -t code -r ${RESOURCE} -c onoff -F anchor_card_ccin)
              anchor_card_serial_num=$(lscod -m ${SYSTEM} -t code -r ${RESOURCE} -c onoff -F anchor_card_serial_num)
              anchor_card_unique_id=$(lscod -m ${SYSTEM} -t code -r ${RESOURCE} -c onoff -F anchor_card_unique_id)
              resource_id=$(lscod -m ${SYSTEM} -t code -r ${RESOURCE} -c onoff -F resource_id)
              activated_resources=$(lscod -m ${SYSTEM} -t code -r ${RESOURCE} -c onoff -F activated_resources)
              sequence_num=$(lscod -m ${SYSTEM} -t code -r ${RESOURCE} -c onoff -F sequence_num)
              entry_check=$(lscod -m ${SYSTEM} -t code -r ${RESOURCE} -c onoff -F entry_check)
 
              if [ ${RESOURCE} == "mem" ]; then
                     printf "\nMemory details for ${SYSTEM}\n\n"
              elif [ ${RESOURCE} == "proc" ]; then
                     printf "\nProcessor details for ${SYSTEM}\n\n"
              fi
              printf "                   System type : ${sys_type}\n"
              printf "          System serial number : ${sys_serial_num}\n"
              printf "              Anchor card CCIN : ${anchor_card_ccin}\n"
              printf "     Anchor card serial number : ${anchor_card_serial_num}\n"
              printf " Anchor card unique identifier : ${anchor_card_unique_id}\n"
              printf "           Resource identifier : ${resource_id}\n"
              printf "            Activate resources : ${activated_resources}\n"
              printf "               Sequence number : ${sequence_num}\n"
              printf "                   Entry check : ${entry_check}\n"
       else
              printf "\nUsage: onoffdetails [managed system] [mem | proc]\n"
       fi
fi
}
```
