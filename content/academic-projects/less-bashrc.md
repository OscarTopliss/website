---
title: "Linux Virtual Filesystem Report"
weight: 1
---
# Operating Systems Coursework
I completed this report as part of my first year **Operating Systems** module:

{{<button href="/academic-reports/Oscar-Topliss-less-bashrc-coursework.pdf/">}}
Download Report
{{</button>}}

It was an analysis of what actually happens when a user enters the following
into a terminal:
```bash
less .bashrc
```

My research/report focused on the **Linux Virtual Filesystem**, and included the
following:

- Components of the VFS, including **superblocks**, **inodes**, **dentry
objects** and **file objects**.
- How **paths**, e.g. `./.bashrc`, are resolved using `link_path_walk()`.
- How **files** are represented and handled on a **per-process** basis,
including opening, reading from, and closing files.
- The basics of **Memory isolation** on Linux-based systems.
- How **paging** works on Linux-based systems, including retrieving pages from
secondary storage.
- How processes are spawned from the command line using variations of `fork()`.
- The basics of **user mode** and **kernal mode** execution, including the use
of **syscalls** and **traps** to execute kernel-level operations from a user
mode process.

I achieved a grade of **81%** for this coursework.
