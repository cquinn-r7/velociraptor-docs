---
title: Windows.Applications.IISLogs
hidden: true
tags: [Client Artifact]
---

This artifact enables grep of IISLogs.
Parameters include SearchRegex and WhitelistRegex as regex terms.


```yaml
name: Windows.Applications.IISLogs
description: |
  This artifact enables grep of IISLogs.
  Parameters include SearchRegex and WhitelistRegex as regex terms.

author: "Matt Green - @mgreen27"

parameters:
  - name: IISLogFiles
    default: '*:/inetpub/logs/**3/*.log'
  - name: SearchRegex
    description: "Regex of strings to search in line."
    default: ' POST '
    type: regex
  - name: WhitelistRegex
    description: "Regex of strings to leave out of output."
    default:
    type: regex

sources:
  - precondition: SELECT OS From info() where OS = 'windows'

    query: |
      LET files = SELECT FullPath FROM glob(globs=IISLogFiles)

      SELECT * FROM foreach(row=files,
          query={
              SELECT Line, FullPath FROM parse_lines(filename=FullPath)
              WHERE
                Line =~ SearchRegex
                AND NOT if(condition= WhitelistRegex,
                    then= Line =~ WhitelistRegex,
                    else= FALSE)
          })

```
