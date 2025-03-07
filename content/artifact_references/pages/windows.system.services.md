---
title: Windows.System.Services
hidden: true
tags: [Client Artifact]
---

List Service details.


```yaml
name: Windows.System.Services
description: |
  List Service details.

parameters:
  - name: servicesKeyGlob
    default: HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\
  - name: Calculate_hashes
    default: N
    type: bool
  - name: CertificateInfo
    default: N
    type: bool
  - name: NameRegex
    default: .
    type: regex
  - name: DisplayNameRegex
    default: .
    type: regex
  - name: PathNameRegex
    default: .
    type: regex
  - name: ServiceDllRegex
    default: .
    type: regex
  - name: FailureCommandRegex
    default: .
    type: regex

export: |
    LET Profile = '''
        [
        ["ServiceFailureActions", 0, [
          ["ResetPeriod", 0, "uint32"],
          ["__ActionsCount", 12, "uint32"],
          ["__lpsaActionsHeader", 16, "uint32"],
          ["FailureAction", "x=>x.__lpsaActionsHeader", "Array", {
              "type": "ServiceAction",
              "count": "x=>x.__ActionsCount"
          }]
        ]],
        ["ServiceAction", 8, [
            ["Type", 0, "Enumeration", {
                "type": "uint32",
                "map": {
                    "SC_ACTION_NONE": 0,
                    "SC_ACTION_RESTART": 1,
                    "SC_ACTION_REBOOT": 2,
                    "SC_ACTION_RUN_COMMAND": 3,
                }}],
            ["__DelayMsec", 4, "uint32"],
            ["Delay", 4,"Value",{ "value": "x=>x.__DelayMsec/1000" }],
        ]],
      ]
      '''

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'

    query: |
      LET service <= SELECT State, Name, DisplayName, Status,
            ProcessId as Pid, ExitCode, StartMode,
            PathName, ServiceType, StartName as UserAccount,
            {
                SELECT Mtime as Created
                FROM stat(filename=servicesKeyGlob + Name, accessor='registry')
            } AS Created,
            {
                SELECT expand(path=ServiceDll) AS ServiceDll
                FROM read_reg_key(globs=servicesKeyGlob + Name + "\\Parameters")
                LIMIT 1
            } AS ServiceDll,
            {
                SELECT FailureCommand FROM read_reg_key(globs=servicesKeyGlob + Name)
                LIMIT 1
            } AS FailureCommand,
            {
                SELECT if(condition=FailureActions,
                   then=parse_binary(accessor='data',
                                     filename= FailureActions || " ",
                                     profile=Profile,
                                     struct='ServiceFailureActions')) as FailureActions
                FROM read_reg_key(globs=servicesKeyGlob + Name)
            } AS FailureActions,
            expand(path=parse_string_with_regex(regex=
                ['^"(?P<AbsoluteExePath>[^"]+)','(?P<AbsoluteExePath>^[^ "]+)'],
                string=PathName).AbsoluteExePath) as AbsoluteExePath
        FROM wmi(query="SELECT * From Win32_service", namespace="root/CIMV2")
        WHERE Name =~ NameRegex
            AND DisplayName =~ DisplayNameRegex
            AND PathName =~ PathNameRegex
            AND if(condition=ServiceDll, then=ServiceDll =~ ServiceDllRegex, else=TRUE)
            AND if(condition=FailureCommand, then=FailureCommand =~ FailureCommandRegex, else=TRUE)

      SELECT *,
        if(condition=Calculate_hashes,
            then=hash(path=AbsoluteExePath, accessor="auto")) AS HashServiceExe,
                 if(condition=CertificateInfo,
                    then=authenticode(filename=AbsoluteExePath)) AS CertinfoServiceExe,
                 if(condition=Calculate_hashes,
                    then=hash(path=ServiceDll,accessor="auto")) AS HashServiceDll,
                 if(condition=CertificateInfo,
                    then=authenticode(filename=ServiceDll)) AS CertinfoServiceDll
      FROM service

```
