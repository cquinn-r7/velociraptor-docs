---
title: Windows.KapeFiles.Extract
hidden: true
tags: [Client Artifact]
---

The Windows.KapeFiles.Targets artifact collects files into a Zip
file. Zip files can not generally preserve timestamps since they
only have a single timestamp concept. Velociraptor will only record
the modified time in the zip file header itself but all the times
are present in the metadata file:

"Windows.KapeFiles.Targets/All File Metadata.json"

Sometimes, users wish to extract the contents of a collection to a
directory, and run an external tool over the data. Some such
external tools assume the file timestamps (e.g. prefetch files) are
meaningful. In this case we need to preserve the timestamps.

You can use this artifact to extract the content of a collection
while preserving the timestamps. The artifact will read the metadata
file, unpack the contents of the container and set the timestamps on
the resulting file.

NOTE: Windows allows 3 timestamps to be set (MAC time except for
Btime), while Linux only allows 2 timestamps (Modified and
Accessed).

## Example - command line invocation

```
velociraptor-v0.6.1-rc1-linux-amd64 artifacts collect Windows.KapeFiles.Extract --args ContainerPath=Collection-DESKTOP-2OR51GL-2021-07-16_06_56_50_-0700_PDT.zip --args OutputDirectory=/tmp/MyOutput/
```


```yaml
name: Windows.KapeFiles.Extract
description: |
  The Windows.KapeFiles.Targets artifact collects files into a Zip
  file. Zip files can not generally preserve timestamps since they
  only have a single timestamp concept. Velociraptor will only record
  the modified time in the zip file header itself but all the times
  are present in the metadata file:

  "Windows.KapeFiles.Targets/All File Metadata.json"

  Sometimes, users wish to extract the contents of a collection to a
  directory, and run an external tool over the data. Some such
  external tools assume the file timestamps (e.g. prefetch files) are
  meaningful. In this case we need to preserve the timestamps.

  You can use this artifact to extract the content of a collection
  while preserving the timestamps. The artifact will read the metadata
  file, unpack the contents of the container and set the timestamps on
  the resulting file.

  NOTE: Windows allows 3 timestamps to be set (MAC time except for
  Btime), while Linux only allows 2 timestamps (Modified and
  Accessed).

  ## Example - command line invocation

  ```
  velociraptor-v0.6.1-rc1-linux-amd64 artifacts collect Windows.KapeFiles.Extract --args ContainerPath=Collection-DESKTOP-2OR51GL-2021-07-16_06_56_50_-0700_PDT.zip --args OutputDirectory=/tmp/MyOutput/
  ```

parameters:
  - name: MetadataFile
    default: "Windows.KapeFiles.Targets/All File Metadata.json"
    description: Name of the KapeFile.Targets metadata file.
  - name: UploadsFile
    default: "Windows.KapeFiles.Targets/Uploads.json"
    description: Name of the KapeFile.Targets uploads file.
  - name: OutputDirectory
    description: Directory to write on (must be set).
  - name: ContainerPath
    description: Path to container (zip file) to unpack.

sources:
  - query: |
      // Read the uploads source into memory so we can translate
      // between the original path to the zip path quickly.
      LET Uploads <= memoize(key="SourceFile", query={
        SELECT SourceFile, DestinationFile
        FROM parse_jsonl(accessor="zip",
                         filename=pathspec(DelegatePath=ContainerPath, Path=UploadsFile))
      })

      // Walk over the Metadata source and resolve each file's container name.
      LET FileStats = SELECT *, get(item=Uploads, field=SourceFile).DestinationFile AS ZipPath
         FROM parse_jsonl(
           accessor="zip",
           filename=pathspec(DelegatePath=ContainerPath, Path=MetadataFile))

      // Copy the file from the container to the output directory
      // preserving timestamps.
      LET doit = SELECT ZipPath, Created, LastAccessed, Modified,
         upload_directory(
           output=OutputDirectory, name=ZipPath,
           mtime=Modified, atime=LastAccessed, ctime=Created,
           accessor='zip',
           file=pathspec(
             DelegatePath=ContainerPath,
             Path=ZipPath)) AS CreatedFile
      FROM FileStats

      // Check that the user gave us all the require args.
      SELECT * FROM if(condition= OutputDirectory AND ContainerPath, then=doit,
         else={
           SELECT * FROM scope() WHERE
           log(message="<red>ERROR</>: Both OutputDirectory and ContainerPath must be specified.") AND FALSE
      })

```
