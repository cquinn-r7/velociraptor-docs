---
title: Windows.Search.FileFinder
hidden: true
tags: [Client Artifact]
---

Find files on the filesystem using the filename or content.


## Performance Note

This artifact can be quite expensive, especially if we search file
content. It will require opening each file and reading its entire
content. To minimize the impact on the endpoint we recommend this
artifact is collected with a rate limited way (about 20-50 ops per
second).

This artifact is useful in the following scenarios:

  * We need to locate all the places on our network where customer
    data has been copied.

  * We’ve identified malware in a data breach, named using short
    random strings in specific folders and need to search for other
    instances across the network.

  * We believe our user account credentials have been dumped and
    need to locate them.

  * We need to search for exposed credit card data to satisfy PCI
    requirements.

  * We have a sample of data that has been disclosed and need to
    locate other similar files


```yaml
name: Windows.Search.FileFinder
description: |
  Find files on the filesystem using the filename or content.


  ## Performance Note

  This artifact can be quite expensive, especially if we search file
  content. It will require opening each file and reading its entire
  content. To minimize the impact on the endpoint we recommend this
  artifact is collected with a rate limited way (about 20-50 ops per
  second).

  This artifact is useful in the following scenarios:

    * We need to locate all the places on our network where customer
      data has been copied.

    * We’ve identified malware in a data breach, named using short
      random strings in specific folders and need to search for other
      instances across the network.

    * We believe our user account credentials have been dumped and
      need to locate them.

    * We need to search for exposed credit card data to satisfy PCI
      requirements.

    * We have a sample of data that has been disclosed and need to
      locate other similar files


precondition:
  SELECT * FROM info() where OS = 'windows'

parameters:
  - name: SearchFilesGlob
    default: C:\Users\*
    description: Use a glob to define the files that will be searched.

  - name: SearchFilesGlobTable
    type: csv
    default: |
      Glob
      C:/Users/SomeUser/*
    description: Alternative specify multiple globs in a table

  - name: Accessor
    default: auto
    description: The accessor to use
    type: choices
    choices:
      - auto
      - registry
      - file
      - ntfs

  - name: YaraRule
    type: yara
    default:
    description: A yara rule to search for matching files.

  - name: Upload_File
    default: N
    type: bool

  - name: Calculate_Hash
    default: N
    type: bool

  - name: MoreRecentThan
    default: ""
    type: timestamp

  - name: ModifiedBefore
    default: ""
    type: timestamp


sources:
  - query: |
      LET file_search = SELECT FullPath,
               get(item=Data, field="mft") as Inode,
               Mode.String AS Mode, Size,
               Mtime AS MTime,
               Atime AS ATime,
               Btime AS BTime,
               Ctime AS CTime, "" AS Keywords,
               IsDir, Data
        FROM glob(globs=SearchFilesGlobTable.Glob + SearchFilesGlob,
                  accessor=Accessor)

      LET more_recent = SELECT * FROM if(
        condition=MoreRecentThan,
        then={
          SELECT * FROM file_search
          WHERE MTime > MoreRecentThan
        }, else=file_search)

      LET modified_before = SELECT * FROM if(
        condition=ModifiedBefore,
        then={
          SELECT * FROM more_recent
          WHERE MTime < ModifiedBefore
           AND  MTime > MoreRecentThan
        }, else=more_recent)

      LET keyword_search = SELECT * FROM if(
        condition=YaraRule,
        then={
          SELECT * FROM foreach(
            row={
               SELECT * FROM modified_before
               WHERE NOT IsDir
            },
            query={
               SELECT FullPath, Inode, Mode,
                      Size, MTime, ATime, CTime, BTime,
                      str(str=String.Data) As Keywords, IsDir, Data

               FROM yara(files=FullPath,
                         key="A",
                         rules=YaraRule,
                         accessor=Accessor)
            })
        }, else=modified_before)

      SELECT FullPath, Inode, Mode, Size, MTime, ATime,
             CTime, BTime, Keywords, IsDir,
               if(condition=Upload_File and NOT IsDir,
                  then=upload(file=FullPath, accessor=Accessor)) AS Upload,
               if(condition=Calculate_Hash and NOT IsDir,
                  then=hash(path=FullPath, accessor=Accessor)) AS Hash,
            Data
      FROM keyword_search

column_types:
  - name: Modified
    type: timestamp
  - name: ATime
    type: timestamp
  - name: MTime
    type: timestamp
  - name: CTime
    type: timestamp

```
