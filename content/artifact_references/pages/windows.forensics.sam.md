---
title: Windows.Forensics.SAM
hidden: true
tags: [Client Artifact]
---

Parses user account information from the SAM hive.

Based on Omer Yampel's parser

reference: https://github.com/yampelo/samparser/blob/master/samparser.py


```yaml
name: Windows.Forensics.SAM
description: |
   Parses user account information from the SAM hive.

   Based on Omer Yampel's parser

   reference: https://github.com/yampelo/samparser/blob/master/samparser.py

parameters:
   - name: SAMPath
     description: Path to the SAM file to parse.
     default: C:/Windows/System32/Config/SAM

export: |
     // Reference: https://github.com/yampelo/samparser/blob/master/samparser.py
     LET Profile = '''
     [
       ["F", 0, [
         ["LastLoginDate", 8, "WinFileTime"],
         ["PasswordResetDate", 24, "WinFileTime"],
         ["PasswordFailDate", 40, "WinFileTime"],
         ["RID", 48, "uint32"],
         ["Flags", 56, "Flags", {
             "type": "uint16",
             "bitmap": {
              "Account Disabled": 0,
              "Home directory required": 1,
              "Password not required": 2,
              "Temporary duplicate account": 3,
              "Normal user account": 4,
              "MNS logon user account": 5,
              "Interdomain trust account": 6,
              "Workstation trust account": 7,
              "Server trust account": 8,
              "Password does not expire": 9,
              "Account auto locked": 10
             }
         }],
         ["FailedLoginCount", 64, "uint16"],
         ["LoginCount", 66, "uint16"],
       ]],
       ["V", 0, [
        ["AccountType", 4, "Enumeration", {
            "type": "uint32",
            "choices": {
               "188" : "Default Admin User",
               "212" : "Custom Limited Acct",
               "176" : "Default Guest Acct"
            }
        }],
        ["__username_offset", 12, "uint32"],
        ["__username_length", 16, "uint32"],
        ["username", "x=>x.__username_offset + 0xcc", "String", {
            "length": "x=>x.__username_length",
            "encoding": "utf16",
        }],
        ["__fullname_offset", 24, "uint32"],
        ["__fullname_length", 28, "uint32"],
        ["fullname", "x=>x.__fullname_offset + 0xcc", "String", {
            "length": "x=>x.__fullname_length",
            "encoding": "utf16",
        }],
        ["__comment_offset", 36, "uint32"],
        ["__comment_length", 40, "uint32"],
        ["comment", "x=>x.__comment_offset + 0xcc", "String", {
            encoding: "utf16",
            length: "x=>x.__comment_length",
        }],

        ["__driveletter_offset", 84, "uint32"],
        ["__driveletter_length", 88, "uint32"],
        ["driveletter", "x=>x.__driveletter_offset + 0xcc", "String", {
            encoding: "utf16",
            length: "x=>x.__driveletter_length",
        }],

        ["__logon_script_offset", 96, "uint32"],
        ["__logon_script_length", 100, "uint32"],
        ["logon_script", "x=>x.__logon_script_offset + 0xcc", "String", {
            encoding: "utf16",
            length: "x=>x.__logon_script_length",
        }],

        ["__profile_path_offset", 108, "uint32"],
        ["__profile_path_length", 112, "uint32"],
        ["profile_path", "x=>x.__profile_path_offset + 0xcc", "String", {
            encoding: "utf16",
            length: "x=>x.__profile_path_length",
        }],

        ["__workstation_offset", 120, "uint32"],
        ["__workstation_length", 124, "uint32"],
        ["workstation", "x=>x.__workstation_offset + 0xcc", "String", {
            encoding: "utf16",
            length: "x=>x.__workstation_length",
        }],

        ["__lmpwd_hash_offset", 156, "uint32"],
        ["__lmpwd_hash_length", 160, "uint32"],
        ["lmpwd_hash", "x=>x.__lmpwd_hash_offset + 0xcc", "String", {
            encoding: "utf16",
            length: "x=>x.__lmpwd_hash_length",
        }],

        ["__ntpwd_hash_offset", 168, "uint32"],
        ["__ntpwd_hash_length", 172, "uint32"],
        ["ntpwd_hash", "x=>x.__ntpwd_hash_offset + 0xcc", "String", {
            encoding: "utf16",
            length: "x=>x.__ntpwd_hash_length",
        }]
       ]]
     ]
     '''

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'

    query: |
        SELECT Key.OSPath.Path AS Key,
           Key.OSPath.DelegatePath AS Hive,
           get(field="F") AS _F,
           get(field="V") AS _V,
           get(field="SupplementalCredentials") AS _SupplementalCredentials,
           parse_binary(accessor="data", filename=F,
                        profile=Profile, struct="F") AS ParsedF,
           parse_binary(accessor="data", filename=V,
                        profile=Profile, struct="V") AS ParsedV
        FROM read_reg_key(
           globs='SAM\\Domains\\Account\\Users\\0*',
           root=pathspec(DelegatePath=SAMPath),
           accessor="raw_reg")
        WHERE _F AND _V

column_types:
  - name: F
    type: hex
  - name: V
    type: hex

```
