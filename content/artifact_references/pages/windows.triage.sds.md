---
title: Windows.Triage.SDS
hidden: true
tags: [Client Artifact]
---

Collects the $Secure:$SDS stream from the NTFS volume. The $Secure
stream is both a directory (it has I30 stream) and a file (it has a
$DATA stream) and therefore confuses the Windows.KapeFiles.Target
artifact which relies on globbing. Use this artifact to collect the
$SDS stream.


```yaml
name: Windows.Triage.SDS
description: |
  Collects the $Secure:$SDS stream from the NTFS volume. The $Secure
  stream is both a directory (it has I30 stream) and a file (it has a
  $DATA stream) and therefore confuses the Windows.KapeFiles.Target
  artifact which relies on globbing. Use this artifact to collect the
  $SDS stream.

parameters:
  - name: Drive
    description: The Drive letter to analyze
    default: "C:"

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'

    query: |
      LET Device <= pathspec(parse=Drive)

      SELECT *, upload(accessor="mft", file=Device + Inode, name=Name) AS Upload
      FROM foreach(row=parse_ntfs(device=Device, mft=9).Attributes, column="_value")
      WHERE Name =~ "\\$S" AND TypeId IN (128, 160)

```
