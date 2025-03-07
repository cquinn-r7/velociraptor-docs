---
title: Linux.Events.SSHLogin
hidden: true
tags: [Client Event Artifact]
---

This monitoring artifact watches the auth.log file for new
successful SSH login events and relays them back to the server.


```yaml
name: Linux.Events.SSHLogin
description: |
  This monitoring artifact watches the auth.log file for new
  successful SSH login events and relays them back to the server.

reference:
  - https://www.elastic.co/blog/grokking-the-linux-authorization-logs

type: CLIENT_EVENT

parameters:
  - name: syslogAuthLogPath
    default: /var/log/auth.log

  - name: SSHGrok
    description: A Grok expression for parsing SSH auth lines.
    default: >-
      %{SYSLOGTIMESTAMP:timestamp} (?:%{SYSLOGFACILITY} )?%{SYSLOGHOST:logsource} %{SYSLOGPROG}: %{DATA:event} %{DATA:method} for (invalid user )?%{DATA:user} from %{IPORHOST:ip} port %{NUMBER:port} ssh2(: %{GREEDYDATA:system.auth.ssh.signature})?

sources:
  - query: |
      -- Basic syslog parsing via GROK expressions.
      LET success_login = SELECT grok(grok=SSHGrok, data=Line) AS Event, Line
        FROM watch_syslog(filename=syslogAuthLogPath)
        WHERE Event.program = "sshd" AND Event.event = "Accepted"

      SELECT timestamp(string=Event.timestamp) AS Time,
              Event.user AS User,
              Event.method AS Method,
              Event.IP AS SourceIP,
              Event.pid AS Pid
        FROM success_login

```
