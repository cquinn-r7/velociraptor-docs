---
title: Linux.Sys.Pslist
hidden: true
tags: [Client Artifact]
---

List processes and their running binaries.


```yaml
name: Linux.Sys.Pslist
description: |
  List processes and their running binaries.

parameters:
  - name: processRegex
    default: .
    type: regex

precondition: SELECT OS From info() where OS = 'linux'

sources:
  - query: |
        SELECT Pid, Ppid, Name, CommandLine, Exe,
               hash(path=Exe) as Hash,
               Username, timestamp(epoch=CreateTime/1000) AS CreatedTime,
               MemoryInfo.RSS AS RSS,
               Exe =~ "\\(deleted\\)$" AS Deleted
        FROM process_tracker_pslist()
        WHERE Name =~ processRegex

```
