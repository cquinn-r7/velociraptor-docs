---
title: Windows.ETW.DNS
hidden: true
tags: [Client Event Artifact]
---

This artifact monitors DNS queries using ETW.

There are several filteres availible to the user to filter out and target with 
regex, by default duplicate DNSCache requests are filtered out.


```yaml
name: Windows.ETW.DNS
author: Matt Green - @mgreen27
description: |
  This artifact monitors DNS queries using ETW.
  
  There are several filteres availible to the user to filter out and target with 
  regex, by default duplicate DNSCache requests are filtered out.

type: CLIENT_EVENT

parameters:
  - name: ImageRegex
    description: "ImagePath regex filter for"
    default: .
    type: regex
  - name: CommandLineRegex
    description: "Commandline to filter for."
    default: .
    type: regex
  - name: QueryRegex
    description: "DNS query request (domain) to filter for."
    default: .
    type: regex
  - name: AnswerRegex
    description: "DNS answer to filter for."
    default: .
    type: regex
  - name: CommandLineExclusion
    description: "Commandline to filter out. Typically we do not want Dnscache events."
    default: 'svchost.exe -k NetworkService -p -s Dnscache$'
    type: regex
    
    
sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
      
    query: |
      LET RecentProcesses = SELECT * FROM fifo(query={
                SELECT System.TimeStamp AS CreateTime, 
                    EventData.ImageName AS ImageName,
                    int(int=EventData.ProcessID) AS Pid,
                    EventData.MandatoryLabel AS MandatoryLabel,
                    EventData.ProcessTokenElevationType AS ProcessTokenElevationType,
                    EventData.ProcessTokenIsElevated AS TokenIsElevated
                FROM watch_etw(guid="{22fb2cd6-0e7b-422b-a0c7-2fad1fd0e716}", any=0x10)
                WHERE System.ID = 1   
            }, max_rows=1000, max_age=60)
        
      -- Query it once to materialize the FIFO
      LET _ <= SELECT * FROM RecentProcesses
        
      LET GetProcessInfo(TargetPid) = SELECT *, ThreadId as ProcessThreadId
        FROM switch(
            -- First try to get the pid directly
            a={
                SELECT 
                    Name, Pid, CreateTime,
                    Exe as ImageName,
                    CommandLine,
                    Username,
                    TokenIsElevated
                FROM pslist(pid=TargetPid)
            },
            -- Failing this look in the FIFO for a recently started process.
            b={
                SELECT
                    basename(path=ImageName) as Name,
                    Pid,
                    CreateTime,
                    ImageName,
                    Null as CommandLine,
                    Null as Username,
                    if(condition= TokenIsElevated="0", 
                        then= false, 
                        else= true ) as TokenIsElevated
                FROM RecentProcesses
                WHERE Pid = TargetPid
                LIMIT 1
            })
        
        SELECT System.TimeStamp AS EventTime,
            EventData.QueryName AS Query,
            get(item=dict(
                        `1` = 'A',
                        `2` = 'NS',
                        `5` = 'CNAME',
                        `6` = 'SOA',
                        `12` = 'PTR',
                        `13` = 'HINFO',
                        `15` = 'MX',
                        `16` = 'TXT',
                        `17` = 'RP',
                        `18` = 'AFSDB',
                        `24` = 'SIG',
                        `25` = 'KEY',
                        `28` = 'AAAA',
                        `29` = 'LOC',
                        `33` = 'SRV',
                        `35` = 'NAPTR',
                        `36` = 'KX',
                        `37` = 'CERT',
                        `39` = 'DNAME',
                        `42` = 'APL',
                        `43` = 'DS',
                        `44` = 'SSHFP',
                        `45` = 'IPSECKEY',
                        `46` = 'RRSIG',
                        `47` = 'NSEC',
                        `48` = 'DNSKEY',
                        `49` = 'DHCID',
                        `50` = 'NSEC3',
                        `51` = 'NSEC3PARAM',
                        `52` = 'TLSA',
                        `53` = 'SMIMEA',
                        `55` = 'HIP',
                        `59` = 'CDS',
                        `60` = 'CDNSKEY',
                        `61` = 'OPENPGPKEY',
                        `62` = 'CSYNC',
                        `63` = 'ZONEMD',
                        `64` = 'SVCB',
                        `65` = 'HTTPS',
                        `108` = 'EUI48',
                        `109` = 'EUI64',
                        `249` = 'TKEY',
                        `250` = 'TSIG',
                        `256` = 'URI',
                        `257` = 'CAA',
                        `32768` = 'TA',
                        `32769` = 'DLV'),
                    member=str(str=EventData.QueryType)) AS Type,
               EventData.QueryResults AS Answer,
               GetProcessInfo(TargetPid=System.ProcessID,ThreadId=System.ThreadID)[0] as Process
        FROM watch_etw(guid="{1C95126E-7EEA-49A9-A3FE-A378B03DDB4D}")
        WHERE System.ID = 3008
            AND Query AND Process AND Answer 
            AND NOT Process.CommandLine =~ CommandLineExclusion
            AND Process.ImageName =~ ImageRegex
            AND Query =~ QueryRegex
            AND Answer =~ AnswerRegex
```
