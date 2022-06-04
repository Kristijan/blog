---
title: "Automatically reduce image.data to a single PV"
date: "2012-04-04"
categories: 
  - "aix"
  - "powervm"
  - "shell-scripting"
tags: 
  - "aix"
  - "image-data"
  - "ksh"
  - "scripting"
  - "shell"
  - "shell-scripting"
  - "volume-group"
---

I've been working with a client who is going through the process of migrating from physical Power 5 servers to a virtualized Power 7 environment with PowerVM. Due to the I/O limitations in the P5 servers, our only method of migration was to take mksysb/savevg's of the current servers, create NIM resources out of them, and then restore onto the P7 LPAR's.

The Power 5 rootvg consisted of two internal disks in a LVM mirror, with the other volume groups backed by either internal disk, or locally attached storage. The Power 7 which we were migrating to had it's storage provided by a shiny DS8800. Given the boot from SAN solution we had, we no longer required two disks to form the rootvg, as all the mirroring and redundancy was being handled by the SVC's. To successfuly restore the Power 5 mksysb onto the new Power 7 LPAR, we needed to provide an alternate image.data file during installation which has been modified to break the mirror and reduce the volume group down to a single physical disk. You can do this manually, and I would have if there was a small number of hosts to migrate, but I was dealing with a rather large enviornment.

The below script does the following :

1. Modifies the `source_disk_data` stanza to a single entry.
2. Modifies the `VG_SOURCE_DISK_LIST` stanza to the disk set by the -d paramater.
3. Modifies the `LV_SOURCE_DISK_LIST` stanza to the disk set by the -d paramater.
4. Modifies the `COPIES` stanza to 1.
5. Modifies the `PP` stanza to match the `LPs` value.

The script takes two values, the resulting hdisk on the other node (In my case, hdisk0 for rootvg, and hdisk1 for datavg), and the location of the image.data or vg.data file.

```console
root@AIX #> ./modify_data.sh
Usage: ./modify_data.sh -d <hdisk> -f <file>
        -d      Destination hdisk on P7 LPAR
        -f      image.data or vg.data file to modify
        -h      Script usage
```

```bash
#!/bin/ksh
#
# Name:
#	modify_data.sh
# Description:
#       This script is for use during migrations where mksysb/savevg 
#       consists of multiple physical volumes and is being restored
#       to a single volume. 
#
#       This script modifies the image.data or vg.data file to allow
#       the restore to a single volume.
#
# Usage:
#       # ./modify_data.sh -d <hdisk> -f <file>
#               -d      Destination hdisk on P7 LPAR
#               -f      image.data or vg.data file to modify
#               -h      Script usage
#
 
#*********************************************************#
# Functions, Variables, Checks  and Command line options *#
#*********************************************************#
 
# Variables
DATE=`date +%Y%m%d`
PWD=`pwd`
 
# Usage function
_usage() {
  echo "Usage: $0 -d <hdisk> -f <file>"
  echo "        -d      Destination hdisk on P7 LPAR"
  echo "        -f      image.data or vg.data file to modify"
  echo "        -h      Script usage"
  echo ""
  echo "Example:"
  echo "       $0 -d hdisk0 -f image.data"
}
 
 
# Display usage if no command line arguments
if [ $# -eq 0 ]; then
  _usage
  exit 1
fi
 
 
# Read command line options
while getopts "d:f:h" OPTION
  do
        case ${OPTION} in
          d) HDISK="${OPTARG}";;
          f) FILE="${OPTARG}";;
          h|*) _usage;;
        esac
  done
 
 
# Check if file exists
if [ ! -f $FILE ]; then
  echo "File \"${FILE}\" does not exist"
  exit 2
fi
 
 
#*********#
# * Main *#
#*********#
 
# Make a copy of the current file
cp $FILE ${PWD}/${FILE}.${DATE}
 
# Modify source_disk_data stanza
awk '
/source_disk_data:/{c=8}!(c&&c--)
' ${FILE} > /tmp/${FILE}.tmp1
 
sed '
/source_disk_data;/ a\
\
source_disk_data:\
        PVID= 000107b0e68c43ec\
        PHYSICAL_LOCATION= U787F.001.DQM0XW6-P1-C4-T1-L3-L0\
        CONNECTION= scsi0//3,0\
        LOCATION= 00-08-00-3,0\
        SIZE_MB= 70006\
        HDISKNAME= '"$HDISK"'
' /tmp/${FILE}.tmp1 > /tmp/${FILE}.tmp2
 
# Modify VG_SOURCE_DISK_LIST, LV_SOURCE_DISK_LIST and
# COPIES stanza
sed -e '
/VG_SOURCE_DISK_LIST/ c\
        VG_SOURCE_DISK_LIST= '"$HDISK"'
' -e '
/LV_SOURCE_DISK_LIST/ c\
        LV_SOURCE_DISK_LIST= '"$HDISK"'
' -e '
/COPIES/ c\
        COPIES= 1
' /tmp/${FILE}.tmp2 > /tmp/${FILE}.tmp3
 
# Modify PP stanza
awk '
BEGIN { FS="="; modified=0}
 
/COPIES/,/BB_POLICY/ {
 
   if ($1 ~ /LPs$/) { saved = $2; }
 
   if($1 ~ /PP$/) { var = saved; }
   else { var=$2; }
 
   print $1"="var;
   modified = 1
}
{
  if (modified == 0) { print; }
 
  else { modified = 0; }
}
' /tmp/${FILE}.tmp3 > /tmp/${FILE}.tmp4
 
# Copy modified file to $PWD
cp /tmp/${FILE}.tmp4 ${PWD}/${FILE}.1PV
 
# Cleanup temp files
rm /tmp/${FILE}.tmp1
rm /tmp/${FILE}.tmp2
rm /tmp/${FILE}.tmp3
rm /tmp/${FILE}.tmp4
 
# Output
echo ""
echo "Backup of ${FILE} is located at ${PWD}/${FILE}.${DATE}"
echo ""
echo "Modified file is located at ${PWD}/${FILE}.1PV"
echo ""
 
exit 0
```
{: file="modify_data.sh" }

As you can see, the script is very ugly and messy, but it does the job. I've successfully used this to do a number of migrations. Feel free to use, abuse or modify it. If you do modify it though, please let me know, and I can incorporate the changes into mine :)
