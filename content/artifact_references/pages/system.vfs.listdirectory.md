---
title: System.VFS.ListDirectory
hidden: true
tags: [Client Artifact]
---

This is an internal artifact used by the GUI to populate the
VFS. You may run it manually if you like, but typically it is
launched by the GUI when a user clicks the "Refresh this directory"
button.


```yaml
name: System.VFS.ListDirectory
description: |
  This is an internal artifact used by the GUI to populate the
  VFS. You may run it manually if you like, but typically it is
  launched by the GUI when a user clicks the "Refresh this directory"
  button.

parameters:
  - name: Path
    description: The path of the file to download.
    default: "/"

  - name: Components
    type: json_array
    description: Alternatively, this is an explicit list of components.

  - name: Accessor
    default: file

  - name: Depth
    type: int
    default: 0

sources:
  - query: |
      // Glob > v2 accepts a component list for the root parameter.
      LET Path <= if(condition=version(plugin="glob") > 2 AND Components,
        then=Components, else=Path)

      // Old versions do not have the root parameter to glob()
      // Fixes https://github.com/Velocidex/velociraptor/issues/322
      LET LegacyQuery = SELECT FullPath as _FullPath,
           Accessor as _Accessor,
           Data as _Data,
           Name, Size, Mode.String AS Mode,
           Mtime as mtime,
           Atime as atime,
           Ctime as ctime
        FROM glob(globs=Path + if(condition=Depth,
             then=format(format='/**%v', args=Depth), else='/*'),
             accessor=Accessor)

      LET NewQuery = SELECT FullPath as _FullPath,
           Accessor as _Accessor,
           Data as _Data,
           Name, Size, Mode.String AS Mode,
           Mtime as mtime,
           Atime as atime,
           Ctime as ctime,
           Btime AS btime
        FROM glob(
             globs=if(condition=Depth,
                then=format(format='/**%v', args=Depth),
                else='/*'),
             root=Path,
             accessor=Accessor)

      SELECT * FROM if(
       condition=version(plugin="glob") >= 1,
       then=NewQuery,
       else=LegacyQuery)

```
