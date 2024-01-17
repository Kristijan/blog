---
title: "AIX & VIOS Security Advisories"
date: "2024-01-17"
categories: 
  - "aix"
  - "vios
tags: 
  - "advisories"
  - "aix"
  - "python"
  - "security"
  - "vios"
---

IBM provide a number of resources to keep up to date with security advisory announcements. IBM X-Force Exchange provide an API, IBM Support provide email notifications (along with RSS/Atom feeds), and there is also a JSON feed that comes from [https://esupport.ibm.com/customercare/flrt/doc?page=aparJSON](https://esupport.ibm.com/customercare/flrt/doc?page=aparJSON){:target="_blank"}. For my requirements, I've found the JSON feed to contain the data that I need at a glance. It provides me with an abstract, the URL to the advisory, if the fix requires a reboot, and the CVE number along with its associated CVSS score.

This is enough information for me to quickly triage a security advisory and determine if I need to look into it in any further detail. However, looking at raw JSON data isn't pretty, and I wanted it presented in a format that was a little more involved than what I could do with [`jq`](https://jqlang.github.io/jq){:target="_blank"}. I ended up writing something in python that you can find on my [GitHub page](https://github.com/Kristijan/aix_security_advisories){:target="_blank"} that produces the below formatted table.

![aix_security_advisories.py screenshot](https://raw.githubusercontent.com/Kristijan/aix_security_advisories/main/assets/screenshot.png)
_aix_security_advisories.py screenshot_

I'm not using all the available data in the output. Additional items include a link to download a fix (if available) and a list of impacted filesets and versions. It's not data that I need at a quick glance, and it's all available at the advisory URL. If anyone wants that data included in the table, I welcome contributions, but also happy to add it in. All this is just me trying to have a better understanding of working with python, and making my day-to-day easier.