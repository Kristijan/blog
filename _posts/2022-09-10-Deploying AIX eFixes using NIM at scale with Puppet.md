---
title: "Deploying AIX eFixes using NIM at scale with Puppet"
date: "2022-09-10"
categories:
  - "aix"
tags:
  - "aix"
  - "nim"
---

Most will be familiar with using the `emgr` command to install emergency fixes (eFixes) on AIX. You can additionally use a Network Installation Management [(NIM)](https://www.ibm.com/docs/en/aix/7.2?topic=installing-network-installation-management){:target="_blank"} server in either a push or pull operation to install fixes, which is handy when deploying at scale (more on this later in the post with deploying fixes at scale with Puppet).

## Prepare NIM resources

There is a little setup work that you first need to perform on the NIM server to make this possible. At a minimum, on the NIM server, you will require a lpp_source resource to place the fixes in. Creating a lpp_source resource is outside the scope of this post, but you can find additional information [here](https://www.ibm.com/docs/en/aix/7.2?topic=resource-defining-lpp-source){:target="_blank"}.

```terminal
(nim01) # lsnim -l 7200TL5SP4_lpp
7200TL5SP4_lpp:
   class       = resources
   type        = lpp_source
   comments    = AIX 7.2 TL5 SP4 lppsource
   arch        = power
   Rstate      = ready for use
   prev_state  = unavailable for use
   location    = /nim/lpp/7200TL5SP4_lpp
   simages     = yes
   alloc_count = 0
   server      = master
(nim01) # ls -l /nim/lpp/7200TL5SP4_lpp/emgr/ppc
total 64320
-rw-r--r--    1 root     system     28191901 Aug 16 00:49 1121201a.220804.epkg.Z
-rw-r--r--    1 root     system        49935 Jun 11 03:37 IJ39876s3a.220506.epkg.Z
-rw-r--r--    1 root     system      4683895 Aug 15 16:08 IJ41139m4a.220809.epkg.Z
```

Optionally, if you're installing multiple fixes at the same time, you can create a NIM installp_bundle resource. Creating an installp_bundle resource is outside the scope of this post, but you can find additional information [here](https://www.ibm.com/docs/en/aix/7.2?topic=resource-defining-installp-bundle){:target="_blank"}.

```terminal
(nim01) # lsnim -l 7200TL5SP4_emgr
7200TL5SP4_emgr:
   class       = resources
   type        = installp_bundle
   comments    = AIX 7.2 TL5 SP4 emgr_bundle
   Rstate      = ready for use
   prev_state  = unavailable for use
   location    = /nim/bundles/7200TL5SP4_emgr
   alloc_count = 0
   server      = master
```

Contents of `/nim/bundles/7200TL5SP4_emgr`{: .filepath}

```plaintext
E:IJ39876s3a.220506.epkg.Z
E:IJ41139m4a.220809.epkg.Z
E:1121201a.220804.epkg.Z
```
{: file="/nim/bundles/7200TL5SP4_emgr" }

## Installing fixes on AIX hosts

You have two options when it comes to installing the fixes onto AIX hosts using the NIM server. You can either push the fixes from the NIM server to the AIX host, or you can pull the fixes from the AIX host.

### Push

Installing a single fix from a NIM lpp_source resource.

```terminal
(nim01) # nim -o cust -a lpp_source=7200TL5SP4_lpp -a filesets=E:1121201a.220804.epkg.Z aix01

Initializing log /var/adm/ras/emgr.log ...
EPKG NUMBER       LABEL               OPERATION              RESULT
===========       ==============      =================      ==============
1                 1121201a            INSTALL                SUCCESS
Return Status = SUCCESS
```

Installing multiple fixes from a NIM installp_bundle resource.

```terminal
(nim01) # nim -o cust -a lpp_source=7200TL5SP4_lpp -a installp_bundle=7200TL5SP4_emgr aix01

Initializing log /var/adm/ras/emgr.log ...
EPKG NUMBER       LABEL               OPERATION              RESULT
===========       ==============      =================      ==============
1                 IJ39876s3a          INSTALL                SUCCESS
2                 IJ41139m4a          INSTALL                SUCCESS
3                 1121201a            INSTALL                SUCCESS
Return Status = SUCCESS
```

### Pull

Installing a single fix from a NIM lpp_source resource.

```terminal
(aix01) # nimclient -o cust -a lpp_source=7200TL5SP4_lpp -a filesets=E:1121201a.220804.epkg.Z

Initializing log /var/adm/ras/emgr.log ...
EPKG NUMBER       LABEL               OPERATION              RESULT
===========       ==============      =================      ==============
1                 1121201a            INSTALL                SUCCESS
Return Status = SUCCESS
```

Installing multiple fixes from a NIM installp_bundle resource.

```terminal
(aix01) # nimclient -o cust -a lpp_source=7200TL5SP4_lpp -a installp_bundle=7200TL5SP4_emgr

Initializing log /var/adm/ras/emgr.log ...
EPKG NUMBER       LABEL               OPERATION              RESULT
===========       ==============      =================      ==============
1                 IJ39876s3a          INSTALL                SUCCESS
2                 IJ41139m4a          INSTALL                SUCCESS
3                 1121201a            INSTALL                SUCCESS
Return Status = SUCCESS
```

## Deploying fixes at scale with Puppet Tasks

If you're using Puppet Enterprise, and can use Puppet Tasks, here is one that allows you to install both a single fix, or multiple fixes using the above mentioned methods.

### Task metadata

```json
{
    "description": "A task for installing/uninstalling AIX fixes.",
    "parameters": {
        "install": {
            "description": "Set to true to install efix, or set to false to uninstall efix.",
            "type": "Boolean"
        },
        "lpp_source": {
            "description": "NIM lpp_source containing the efix(s).",
            "type": "String[1]"
        },
        "fix_name": {
            "description": "Name of a single efix to be installed/uninstalled.\n\tWhen installing, the 'fix_name' is the name of the package (e.g. IJ31191m1a.210322.epkg.Z).\n\tWhen uninstalling, the 'fix_name' is the name of the label (e.g. IJ31191m1a).",
            "type": "Optional[String[1]]"
        },
        "installp_bundle": {
            "description": "Name of installp_bundle to use when installing multiple efixes.",
            "type": "Optional[String[1]]"
        },
        "reboot": {
            "description": "If true, the system is rebooted after installing/uninstalling the efix(s).",
            "type": "Boolean"
        }
    },
    "puppet_task_version": 1,
    "supports_noop": false
}
```

### Task

```ruby
#!/opt/puppetlabs/puppet/bin/ruby
#
# A task for installing/uninstalling AIX fixes
require 'puppet'
require 'json'

# Set FACTERLIB so custom facts resolve
Puppet.initialize_settings
ENV['FACTERLIB'] = Puppet.settings['factpath']
require 'facter'

# Sanity check that we're an AIX host
if Facter.value(:operatingsystem) == "AIX" then
    params          = JSON.parse(STDIN.read)
    install         = params['install']
    lpp_source      = params['lpp_source']
    fix_name        = params['fix_name']
    installp_bundle = params['installp_bundle']
    reboot          = params['reboot']

    if install then
        # Check if we have either a fix_name or installp_bundle set
        if params.key?("installp_bundle") then
            fix = "installp_bundle=#{installp_bundle}"
        elsif params.key?("fix_name") then
            fix = "filesets=E:#{fix_name}"
        else
            print "You need to specify either a fix_name or installp_bundle."
            exit(2)
        end

        # Install efix(s)
        system("/usr/sbin/nimclient", "-o", "cust", "-a", "lpp_source=#{lpp_source}", "-a", "#{fix}")
        if $?.exitstatus != 0
            print "efix failed to install."
            exit(2)
        end
    else
        # Uninstall efix
        if params.key?("fix_name") then
            system("/usr/sbin/nimclient", "-o", "cust", "-a", "lpp_source=#{lpp_source}", "-a", "filesets=E:#{fix_name}", "-a", "installp_flags=u")
            if $?.exitstatus != 0
                print "efix failed to uninstall."
                exit(2)
            end
        else
            print "You need to specify a fix_name to uninstall."
        end
    end

    # Reboot
    if reboot then
        system("/usr/bin/echo '/usr/sbin/shutdown -r +3' | /usr/bin/at now")
    end
else
  print "Task only supported on AIX."
  exit(1)
end
```

### Running the Task

```terminal
$ puppet task show patch::aix_efix

patch::aix_efix - A task for installing/uninstalling AIX fixes.

USAGE:
$ puppet task run patch::aix_efix install=<value> lpp_source=<value> reboot=<value> [fix_name=<value>] [installp_bundle=<value>] <[--nodes, -n <node-names>] | [--query, -q <'query'>]>

PARAMETERS:
- install : Boolean
    Set to true to install efix, or set to false to uninstall efix.
- lpp_source : String[1]
    NIM lpp_source containing the efix(s).
- reboot : Boolean
    If true, the system is rebooted after installing/uninstalling the efix(s).
- fix_name : Optional[String[1]]
    Name of a single efix to be installed/uninstalled.
        When installing, the 'fix_name' is the name of the package (e.g. IJ31191m1a.210322.epkg.Z).
        When uninstalling, the 'fix_name' is the name of the label (e.g. IJ31191m1a).
- installp_bundle : Optional[String[1]]
    Name of installp_bundle to use when installing multiple efixes.

$ puppet task run patch::aix_efix install=true lpp_source=7200TL5SP4_lpp reboot=false installp_bundle=7200TL5SP4_emgr -n aix01
Starting job ...
Note: The task will run only on permitted nodes.
New job ID: 83786
Nodes: 1

Started on aix01 ...
Finished on node aix01
  STDOUT:

    Initializing log /var/adm/ras/emgr.log ...
    EPKG NUMBER       LABEL               OPERATION              RESULT
    ===========       ==============      =================      ==============
    1                 IJ39876s3a          INSTALL                SUCCESS
    2                 IJ41139m4a          INSTALL                SUCCESS
    3                 1121201a            INSTALL                SUCCESS
    Return Status = SUCCESS

Job completed. 1/1 nodes succeeded.
Duration: 1 min, 3 sec
```
