---
title: Linux.Debian.AptSources
hidden: true
tags: [Client Artifact]
---

Parse Debian apt sources.

We first search for \*.list files which contain lines of the form

.. code:: console

   deb http://us.archive.ubuntu.com/ubuntu/ bionic main restricted

For each line we construct the cache file by spliting off the
section (last component) and replacing / and " " with _.

We then try to open the file. If the file exists we parse some
metadata from it. If not we leave those columns empty.


```yaml
name: Linux.Debian.AptSources
description: |
  Parse Debian apt sources.

  We first search for \*.list files which contain lines of the form

  .. code:: console

     deb http://us.archive.ubuntu.com/ubuntu/ bionic main restricted

  For each line we construct the cache file by spliting off the
  section (last component) and replacing / and " " with _.

  We then try to open the file. If the file exists we parse some
  metadata from it. If not we leave those columns empty.

reference:
  - https://osquery.io/schema/3.2.6#apt_sources
parameters:
  - name: linuxAptSourcesGlobs
    description: Globs to find apt source *.list files.
    default: /etc/apt/sources.list,/etc/apt/sources.list.d/*.list
  - name:  aptCacheDirectory
    description: Location of the apt cache directory.
    default: /var/lib/apt/lists/
sources:
  - precondition:
      SELECT OS From info() where OS = 'linux'
    query: |
         /* Search for files which may contain apt sources. The user can
            pass new globs here. */
         LET files = SELECT FullPath from glob(
           globs=split(string=linuxAptSourcesGlobs, sep=","))

         /* Read each line in the sources which is not commented.
            Deb lines look like:
            deb [arch=amd64] http://dl.google.com/linux/chrome-remote-desktop/deb/ stable main
            Contains URL, base_uri and components.
         */
         LET deb_sources = SELECT *
           FROM parse_records_with_regex(
             file=files.FullPath,
             regex="(?m)^ *(?P<Type>deb(-src)?) (?:\\[arch=(?P<Arch>[^\\]]+)\\] )?" +
                  "(?P<URL>https?://(?P<base_uri>[^ ]+))" +
                  " +(?P<components>.+)")

         /* We try to get at the Release file in /var/lib/apt/ by munging
           the components and URL.
           Strip the last component off, convert / and space to _ and
           add _Release to get the filename.
         */
         LET parsed_apt_lines = SELECT Arch, URL,
            base_uri + " " + components as Name, Type,
            FullPath as Source, aptCacheDirectory + regex_replace(
              replace="_",
              re="_+",
              source=regex_replace(
                replace="_", re="[ /]",
                source=base_uri + "_dists_" + regex_replace(
                   source=components,
                   replace="", re=" +[^ ]+$")) + "_Release"
              )  as cache_file
         FROM deb_sources

         /* This runs if the file was found. Read the entire file into
            memory and parse the same record using multiple RegExps.
         */
         LET parsed_cache_files = SELECT Name, Arch, URL, Type,
           Source, parse_string_with_regex(
                string=Record,
                regex=["Codename: (?P<Release>[^\\s]+)",
                       "Version: (?P<Version>[^\\s]+)",
                       "Origin: (?P<Maintainer>[^\\s]+)",
                       "Architectures: (?P<Architectures>[^\\s]+)",
                       "Components: (?P<Components>[^\\s]+)"]) as Record
           FROM parse_records_with_regex(file=cache_file, regex="(?sm)(?P<Record>.+)")

         // Foreach row in the parsed cache file, collect the FileInfo too.
         LET add_stat_to_parsed_cache_file = SELECT * from foreach(
           query={
             SELECT FullPath, Mtime, Ctime, Atime, Record, Type,
               Name, Arch, URL, Source from stat(filename=cache_file)
           }, row=parsed_cache_files)

         /* For each row in the parsed file, run the appropriate query
            depending on if the cache file exists.
            If the cache file is not found, we just copy the lines we
            parsed from the source file and fill in empty values for
            stat.
         */
         LET parse_cache_or_pass = SELECT * from if(
           condition={
              SELECT * from stat(filename=cache_file)
           },
           then=add_stat_to_parsed_cache_file,
           else={
           SELECT Source, Null as Mtime, Null as Ctime,
               Null as Atime, Type,
               Null as Record, Arch, URL, Name from scope()
           })

         -- For each parsed apt .list file line produce some output.
         SELECT * from foreach(
             row=parsed_apt_lines,
             query=parse_cache_or_pass)

```
