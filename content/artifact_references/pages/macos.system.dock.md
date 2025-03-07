---
title: MacOS.System.Dock
hidden: true
tags: [Client Artifact]
---

This artifact examines the contents of the user's dock.  The
property list entry for each application represented within the dock
can be modified to point to a malcious application.

 By comparing the application name, CFURLString, and book, we can
 gather greater context to assist in determining if an adversary may
 have tampered with an entry, or if an entry has been added to
 emulate a legitimate application.

**ATT&CK**: [T1547.009/T1547.011 - Shortcut modification/Plist modification](https://attack.mitre.org/techniques/T1547/)


```yaml
name: MacOS.System.Dock
description: |
  This artifact examines the contents of the user's dock.  The
  property list entry for each application represented within the dock
  can be modified to point to a malcious application.

   By comparing the application name, CFURLString, and book, we can
   gather greater context to assist in determining if an adversary may
   have tampered with an entry, or if an entry has been added to
   emulate a legitimate application.

  **ATT&CK**: [T1547.009/T1547.011 - Shortcut modification/Plist modification](https://attack.mitre.org/techniques/T1547/)

reference:
  - https://specterops.io/so-con2020/event-758922

author: Wes Lambert - @therealwlambert

type: CLIENT

parameters:
   - name: DockGlob
     default: Users/*/Library/Preferences/com.apple.dock.plist

precondition:
      SELECT OS From info() where OS = 'darwin'

sources:
  - query: |
       LET DockList = SELECT FullPath from glob(globs=DockGlob)

       LET PersistentApps = SELECT get(member="persistent-apps") as PA
         FROM plist(file=DockList.FullPath)

       LET TD = SELECT GUID, TileData from foreach(row=PersistentApps,
       query={
          SELECT get(item=PA, member="GUID") AS GUID,
                 get(item=PA, member="tile-data") AS TileData
          FROM scope()
       })

       LET TDEach = SELECT GUID, TileData FROM foreach(row=TD,
       query={
           SELECT GUID, TileData from scope()
       })

       SELECT
          get(member="file-label") AS FileLabel,
          get(member="file-data._CFURLString") AS AppLocation,
          timestamp(mactime=get(member="file-mod-date")) AS FileModDate,
          timestamp(mactime=get(member="parent-mod-date")) AS ParentModDate,
          get(member="bundle-identifier") AS BundleIdentifier,
          get(member="dock-extra") AS DockExtra,
          book AS Book
        FROM foreach(row=TDEach,query={SELECT * from foreach(row=TileData)})

```
