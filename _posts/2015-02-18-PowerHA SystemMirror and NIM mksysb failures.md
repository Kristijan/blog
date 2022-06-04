---
title: "PowerHA SystemMirror and NIM mksysb failures"
date: "2015-02-18"
categories: 
  - "aix"
  - "hacmp"
  - "powerha"
tags: 
  - "aix"
  - "hacmp"
  - "mksysb"
  - "nim"
  - "powerha"
  - "systemmirror"
---

I built a basic two-node PowerHA SystemMirror (HACMP) cluster for my team a little while ago to use as a test environment for patch updates. While it wasn't a true reflection of how the production environment is configured, it was enough to test functionality. As such, I configured a single virtual ethernet adapter in each cluster node, which would house both the boot IP and the service IP of the cluster. After a couple of weeks, I noticed that my weekly NIM mksysb's on one of the two cluster nodes was always failing. Further investigation found that the NIM mksysb's would always fail on the cluster node that had the active resource group with the service IP attached to it. If I failed the resource group over to the other cluster node, the NIM mksysb would complete successfully. The IP addresses in the environment (IP addresses changed/masked from actuals).

| NIM (hostname: nim) | Cluster boot IP (hostname: powerha1) | Service IP (hostname: powerha1-vip) |
| ------------------- | ------------------------------------ | ----------------------------------- |
| 10.10.10.100        | 10.10.10.101                         | 10.10.10.102                        |

The error from the NIM log.

```console
NIM# nim -o define -t mksysb -a server=master -a location=/export/nim/mksysb/powerha1.mksysb -a source=powerha1 -a mk_image=yes -a mksysb_flags=iepmX powerha1_mksysb
0042-001 nim: processing error encountered on "master":
   0042-001 m_mkbosi: processing error encountered on "powerha1":
   warning: 0042-175 c_mkbosi: An unexpected result was returned by the
        "/usr/sbin/mount" command:
 
mount: 1831-011 access denied for nim:/export/nim/mksysb
mount: 1831-008 giving up on:
nim:/export/nim/mksysb
The file access permissions do not allow the specified action.
rc=175
0042-175 c_mkbosi: An unexpected result was returned by the "/usr/sbin/mount" command:
mount: 1831-011 access denied for nim:/export/nim/mksysb
mount: 1831-008 giving up on:
nim:/export/nim/mksysb
The file access permissions do not allow the specified action.
```

Interface configuration on powerha1.

```console
powerha1# ifconfig -a
en0: flags=1e084863,104c0<UP,BROADCAST,NOTRAILERS,RUNNING,SIMPLEX,MULTICAST,GROUPRT,64BIT,CHECKSUM_OFFLOAD(ACTIVE),LARGESEND,CHAIN>
        inet 10.10.10.102 netmask 0xffffff00 broadcast 10.10.10.255
        inet 10.10.10.101 netmask 0xffffff00 broadcast 10.10.10.255
         tcp_sendspace 262144 tcp_recvspace 262144 rfc1323 1
lo0: flags=e08084b,c0<UP,BROADCAST,LOOPBACK,RUNNING,SIMPLEX,MULTICAST,GROUPRT,64BIT,LARGESEND,CHAIN>
        inet 127.0.0.1 netmask 0xff000000 broadcast 127.255.255.255
         tcp_sendspace 131072 tcp_recvspace 131072 rfc1323 1

powerha1# traceroute nim
trying to get source for nim
source should be 10.10.10.102
traceroute to nim (10.10.10.100) from 10.10.10.102 (10.10.10.102), 30 hops max
outgoing MTU = 1500
 1  10.10.10.252 (10.10.10.252)  2 ms  1 ms  2 ms
 2  nim (10.10.10.100)  1 ms  1 ms  1 ms

powerha1# netstat -in
Name  Mtu   Network     Address            Ipkts Ierrs    Opkts Oerrs  Coll
en0   1500  link#2      1a.d8.5f.87.26.15 81979866     0 34796223     0     0
en0   1500  10.10.10    10.10.10.102     81979866     0 34796223     0     0
en0   1500  10.10.10    10.10.10.101     81979866     0 34796223     0     0
lo0   16896 link#1                        2421883     0  2421883     0     0
lo0   16896 127         127.0.0.1         2421883     0  2421883     0     0
```

As you can see from the traceroute and netstat above, because the Service IP (10.10.10.102) of the resource group is configured as the first alias on the en0 interface, the NFS mount request back to the NIM server occurs from this source IP address. However, back on the NIM server, the NIM client definition has the source address as 10.10.10.101. This is why the NIM server fails to mount the NFS mount on the client to take a successful mksysb.

Fortunately, there is an option called **Distribution Preference** that can be configured on the Service IP. For this particular configuration, that has both the Service IP and Boot IP on the same interface, we want to **Disable Firstalias**. For the change to reflect across the cluster nodes, we need to **Verify and Synchronize Cluster Configuration**, and then bring that particular resource group offline, and the back online.

You can find the option using the following fastpath: `smitty cm_service_ip`, or drilling down the following menu options.

```plaintext
Cluster Applications and Resources
 > Resources
  > Configure Service IP Labels/Addresses
   > Configure Service IP Labels/Address Distribution Preference
```

After making the change, the order of the IP aliases within the powerha1 node look like this.

```console
powerha1# ifconfig -a
en0: flags=1e084863,104c0<UP,BROADCAST,NOTRAILERS,RUNNING,SIMPLEX,MULTICAST,GROUPRT,64BIT,CHECKSUM_OFFLOAD(ACTIVE),LARGESEND,CHAIN>
        inet 10.10.10.101 netmask 0xffffff00 broadcast 10.10.10.255
        inet 10.10.10.102 netmask 0xffffff00 broadcast 10.10.10.255
         tcp_sendspace 262144 tcp_recvspace 262144 rfc1323 1
lo0: flags=e08084b,c0<UP,BROADCAST,LOOPBACK,RUNNING,SIMPLEX,MULTICAST,GROUPRT,64BIT,LARGESEND,CHAIN>
        inet 127.0.0.1 netmask 0xff000000 broadcast 127.255.255.255
         tcp_sendspace 131072 tcp_recvspace 131072 rfc1323 1

powerha1# traceroute nim
trying to get source for nim
source should be 10.10.10.101
traceroute to nim (10.10.10.100) from 10.10.10.101 (10.10.10.101), 30 hops max
outgoing MTU = 1500
 1  10.10.10.252 (10.10.10.252)  2 ms  1 ms  2 ms
 2  nim (10.10.10.100)  1 ms  1 ms  1 ms

powerha1# netstat -in
Name  Mtu   Network     Address            Ipkts Ierrs    Opkts Oerrs  Coll
en0   1500  link#2      1a.d8.5f.87.26.15 82006462     0 34814141     0     0
en0   1500  10.10.10    10.10.10.101     82006462     0 34814141     0     0
en0   1500  10.10.10    10.10.10.102     82006462     0 34814141     0     0
lo0   16896 link#1                        2430058     0  2430058     0     0
lo0   16896 127         127.0.0.1         2430058     0  2430058     0     0
```

After this change, both PowerHA SystemMirror cluster nodes successfully take NIM mksysb backups. Thanks goes out to Chris Gibson [@cgibbo](https://twitter.com/cgibbo){:target="_blank"} in helping resolve this issue.
