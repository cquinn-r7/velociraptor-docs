---
title: Windows.Sys.Drivers
hidden: true
tags: [Client Artifact]
---

Details for in-use Windows device drivers. This does not display
installed but unused drivers.


```yaml
name: Windows.Sys.Drivers
description: |
  Details for in-use Windows device drivers. This does not display
  installed but unused drivers.

precondition:
      SELECT OS From info() where OS = 'windows'

parameters:
  - name: AlsoCheckAuthenticode
    type: bool
    description: If selected we also check the authenticode information.
    default: "Y"

sources:
  - name: SignedDrivers
    query: |
       SELECT *
       FROM wmi(
          query="select * from Win32_PnPSignedDriver",
          namespace="ROOT\\CIMV2")

  - name: RunningDrivers
    query: |
       SELECT *, if(
         condition=AlsoCheckAuthenticode,
         then=authenticode(filename=PathName)) AS Authenticode,
         hash(path=PathName) AS Hashes
       FROM wmi(
         query="select * from Win32_SystemDriver",
         namespace="ROOT\\CIMV2")
    notebook:
      - type: vql_suggestion
        name: Unique issuers
        template: |
          /*
          # Unique Issuers of drivers

          These are the unique signers of drivers on a system
          (excluding microsoft drivers).

          */
          SELECT count() AS Count,
                 enumerate(items=Name) AS Names,
                 Authenticode.IssuerName AS Issuer, Hashes
          FROM source(artifact="Windows.Sys.Drivers/RunningDrivers")
          WHERE NOT Issuer =~ "Microsoft"
          GROUP BY Issuer

```
