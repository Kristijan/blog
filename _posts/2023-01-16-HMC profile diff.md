---
title: "HMC Profile Diff"
date: "2023-01-16"
categories: 
  - "hmc"
tags: 
  - "python"
  - "hmc"
---

More often than not, I find myself having to compare two Hardware Management Console (HMC) logical partition (LPAR) profile configurations. Sometimes this is to ensure that a profile in a disaster recovery site matches that of its production counterpart. Other times, it's to make sure all members of a cluster have identical resource configurations.

In a perfect world, I'd be manging the LPAR configurations using something like Terraform, but this currently isn't an option. I can login to the HMC and manually verify the profiles (which is what I've been doing), but this is tedious and prone to error when you need to compare many profile pairs.

The HMC, a few major version releases ago, introduced a REST API that exposes all the profile data. I've written a script to collect this data, and present it in a more user friendly format for consumption. You can find the [HMC profile diff](https://github.com/Kristijan/hmc_profile_diff){:target="_blank"} script hosted on my GitHub.

Below are some screenshots of the script output.

![Demo](https://raw.githubusercontent.com/Kristijan/hmc_profile_diff/main/img/demo.gif)

Compare and show all LPAR attributes

![Compare All](https://raw.githubusercontent.com/Kristijan/hmc_profile_diff/main/img/compare_all.png){: width="700" height="400" }
_Compare all LPAR profile attributes_

Compare and show only different LPAR attributes

![Compare Diff Only](https://raw.githubusercontent.com/Kristijan/hmc_profile_diff/main/img/compare_diffonly.png){: width="700" height="400" }
_Compare all LPAR profile attributes and only show differences_
