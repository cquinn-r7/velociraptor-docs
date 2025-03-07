---
title: Windows.Events.Trackaccount
hidden: true
tags: [Client Event Artifact]
---

Artifact to detect account usage by monitoring event id 4624. This is useful for tracking attacker activity. If you want to receive Slack notifications you can enable the server_event artifact named 'Server.Alerts.Trackaccount'


```yaml
name: Windows.Events.Trackaccount
description: |
  Artifact to detect account usage by monitoring event id 4624. This is useful for tracking attacker activity. If you want to receive Slack notifications you can enable the server_event artifact named 'Server.Alerts.Trackaccount'

author: Jos Clephas

type: CLIENT_EVENT

parameters:
  - name: eventLog
    default: C:\Windows\system32\winevt\logs\Security.evtx
  - name: UserRegex
    default: 'admin|user'
    type: regex
  - name: LogonTypeRegex
    type: json_array
    default: '[2,3,4,5,7,8,9,10,11]'

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    query: |
      LET files = SELECT * FROM glob(globs=eventLog)

      SELECT timestamp(epoch=System.TimeCreated.SystemTime) As EventTime,
              System.EventRecordID as EventRecordID,
              System.EventID.Value as EventID,
              System.Computer as SourceComputer,
              EventData.TargetUserName as TargetUserName,
              EventData.LogonType as LogonType,
              EventData.IpAddress as IpAddress,
              EventData.WorkstationName as TargetWorkstationName,
              System,
              EventData,
              Message

        FROM foreach(
          row=files,
          async=TRUE,
          query={
            SELECT *
            FROM watch_evtx(filename=FullPath)
            WHERE System.EventID.Value = 4624
                AND EventData.TargetUserName =~ UserRegex
                AND EventData.LogonType in LogonTypeRegex
        })

```
