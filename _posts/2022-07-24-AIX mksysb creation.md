---
title: "AIX mksysb creation date"
date: "2022-07-24"
categories: 
  - "aix"
tags: 
  - "python"
  - "scripting"
  - "shell"
  - "shell-scripting"
---

Something that I do quite frequently is verify that I have a valid mksysb image for the AIX LPAR that I'm about to make changes on (for example, before applying a service pack). You can do this from the AIX LPAR itself using the `nimclient` command.

```terminal
kristijan@aix13 # nimclient -l -L -t mksysb aix13 | grep aix13
aix13_220720_img     mksysb
aix13_220713_img     mksysb
kristijan@aix13 # nimclient -l -l aix13_220720_img
aix13_220720_img:
   class         = resources
   type          = mksysb
   creation_date = Wed Jul 20 00:17:25 2022
   source_image  = aix13
   arch          = power
   Rstate        = ready for use
   prev_state    = unavailable for use
   location      = /mksysb/aix13/aix13_220720_img
   version       = 7
   release       = 2
   mod           = 5
   oslevel_r     = 7200-05
   oslevel_s     = 7200-05-04-2220
   alloc_count   = 0
   server        = master
```

In my AIX environment, I define a "valid" mksysb as one that was created within the last 2 weeks. Anything created after that date I'd be hesitant in restoring from. Parsing the `creation_date` with a shell script isn't all that elegant. The python [datetime](https://docs.python.org/3/library/datetime.html){:target="_blank"} module allows us to parse this easily.

The script does assume (on line 18) that your NIM mksysb resource names contain the hostname you're running the script from.

```python
#!/usr/bin/env python3
#
# Check that there is at least one mksysb for the client, and
# that the creation date of the mksysb is not older than 15 days

import sys
import socket
import subprocess
from datetime import datetime

# Get hostname
hostname = socket.gethostname()

# Get current time
current_time = datetime.now()

# Create list of mksysb resources from the NIM server
nim_mksysb_list = subprocess.check_output(f"/usr/sbin/nimclient -l -L -t mksysb {hostname} | /usr/bin/awk '/{hostname}/{{ print $1 }}'", shell=True, encoding='utf-8').split()
# If the subprocess above returns no values, the result is a
# single item list with an empty string. Let's strip that out.
nim_mksysb_list = filter(None, nim_mksysb_list)

# Parse list of NIM mksysb backups and compare creation date
if nim_mksysb_list:
    for mksysb in nim_mksysb_list:
        mksysb_creation_time = subprocess.check_output(f"/usr/sbin/nimclient -l -l {mksysb} | /usr/bin/awk -F = '/creation_date/{{ print $2 }}'", shell=True, encoding='utf-8').strip()
        mksysb_creation_time = datetime.strptime(''.join(mksysb_creation_time), '%c')
        elapsed = current_time - mksysb_creation_time
        if elapsed.days < 15:
            # mksysb found on NIM and is not older than 15 days
            sys.exit(0)
else:
    # List is empty, no mksysb backups on NIM.
    sys.exit(1)

# If we've made it this far, all mksysb's are older than 15 days
sys.exit(2)
```

The above script will return the following exit codes:

| Exit code | Description                                |
| --------- | ------------------------------------------ |
| `0`       | mksysb found and is not older than 15 days |
| `1`       | no mksysb found                            |
| `2`       | all mksysbs found are older than 15 days   |

The use of exit codes to determine the validity of a mksysb is because its intended use is part of a larger pipeline of checks that occur before an AIX LPAR is patched. It forms one of many pre-checks that are run on an AIX LPAR before it's deemed healthy to be patched.
