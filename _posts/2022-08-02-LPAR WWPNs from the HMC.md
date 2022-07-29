---
title: "LPAR WWPNs from the HMC"
date: "2022-08-13"
categories:
  - "hmc"
tags:
  - "hmc"
  - "scripting"
  - "shell"
  - "shell-scripting"
  - "storage"
---

There are several ways to find the WWPNs for a virtual fibre channel adapter. You can log into the individual LPAR's, get them from the Hardware Management Console (HMC), or from another asset discovery tool that may contain this data.

Using [EZH](http://ezh.sourceforge.net){:target="_blank"} functions originally written by Brian Smith, I've added a function named `lparwwpn` that takes a list of LPAR names as arguments, and prints the virtual fibre channel WWPNs.

```terminal
kristijan@hmc:~> source lparwwpn
kristijan@hmc:~> lparwwpn aix1 aix2

aix1 [POWER-FRAME-1]
c0XXXXXXXXXXXXf2,c0XXXXXXXXXXXXf3
c0XXXXXXXXXXXXf4,c0XXXXXXXXXXXXf5
c0XXXXXXXXXXXXf6,c0XXXXXXXXXXXXf7
c0XXXXXXXXXXXXf8,c0XXXXXXXXXXXXf9

aix2 [POWER-FRAME-2]
c0XXXXXXXXXXXX20,c0XXXXXXXXXXXX21
c0XXXXXXXXXXXX22,c0XXXXXXXXXXXX23
c0XXXXXXXXXXXX24,c0XXXXXXXXXXXX25
c0XXXXXXXXXXXX26,c0XXXXXXXXXXXX27
```

Save the contents of the shell script below to a file named `lparwwpn`, and copy it over to the HMC.

> You can name the file anything you like, just adjust the `source` command above accordingly.
{: .prompt-tip }

```bash
lparframelookup () {
    while read -r system; do
        while read -r lpar; do
            if [ "$lpar" = "$*" ]; then
                echo "${system}";
            fi
        done < <(lssyscfg -m "$system" -r lpar -F name)
    done < <(lssyscfg -r sys -F "name,state" | grep -E "Standby|Operating" | cut -d, -f 1) | tail -n 1
}

checklpar () {
    count=0;
    if [ -n "$1" ]; then
        while read -r system; do
            while read -r l; do
                if [ "$l" = "$1" ] ; then
                    count=$((count +1))
                fi
            done < <(lssyscfg -m "$system" -r lpar -F name)
        done < <(lssyscfg -r sys -F "name,state" | grep -E "Standby|Operating" | cut -d, -f 1)
        l=""
        if [ $count -eq 0 ]; then
            echo "ERROR:  LPAR not found: $1"
            return 1
        fi
        if [ $count -gt 1 ]; then
            echo "ERROR:  Multiple LPAR's with same name $1"
            return 2
        fi
    else
        unset input
        unset lpararray
        echo "Select LPAR: "
        while read -r system; do
            while read -r l; do
                count=$((count +1))
                printf "%5s. %-20s %-20s\n" $count "$l" "$(lssyscfg -r lpar -m \""${system}"\" -F state --filter lpar_names=\""${l}"\")"
                lpararray[$count]="$l"
            done < <(lssyscfg -m "$system" -r lpar -F name)
        done < <(lssyscfg -r sys -F "name,state" | grep -E "Standby|Operating" | cut -d, -f 1)
        echo
        while [ -z "${lpararray[$input]}" ]; do
            printf "%s\n" "Enter LPAR number (1-$count, \"q\" to quit): ";
            read -r input
            if [ "$input" = "q" -o "$input" = "Q" ]; then return 1; fi
        done
        lpar="${lpararray[$input]}"
        checklpar "$lpar" || return 2
    fi
}

lparwwpn () {
    for lpar in "$@"; do
        frame=$(lparframelookup "${lpar}")
        printf "\n%s [%s]\n" "${lpar}" "${frame}"
        checklpar "$lpar" && eval lshwres -m "${frame}" -r virtualio --rsubtype fc --level lpar --filter \"lpar_names="${lpar}"\" | cut -d '=' -f11 | sed 's/\"//' | sort;
    done
}
```

I've recently come across [EEZH](https://github.com/opokam/eezh){:target="_blank"}, which appears to be a fork of Brian Smith's original EZH code. Might be more beneficial to add `lparwwpn` into that project.
