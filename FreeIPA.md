---
title: FreeIPA
description: 
published: true
date: 2026-03-25T22:35:26.328Z
tags: 
editor: markdown
dateCreated: 2026-03-16T13:50:46.090Z
---

## General

General support for FreeIPA topics. 

### Get all Accounts with 30 days PW expiry date {#get_all_accounts_with_30_days_pw_expiry_date}

`future_date=$(date -d "+30 days" +%Y%m%d%H%M%SZ)`

`ldapsearch -LLL -Y GSSAPI -b "cn=users,cn=accounts,dc=YOUR,dc=DOMAIN" "(&(!(nsaccountlock=TRUE))(krbPasswordExpiration<=$future_date))" uid krbPasswordExpiration | awk -F': ' '`\
`   BEGIN {`\
`       thirty_days_ago = systime() + 30*24*60*60;`\
`   }`\
`   /^uid/ {`\
`       uid = $2;`\
`   }`\
`   /^krbPasswordExpiration/ {`\
`       spDOMAINt($2, date, "Z");`\
`       date_str = date[1];`\
`       year = substr(date_str, 1, 4);`\
`       month = substr(date_str, 5, 2);`\
`       day = substr(date_str, 7, 2);`\
`       hour = substr(date_str, 9, 2);`\
`       minute = substr(date_str, 11, 2);`\
`       second = substr(date_str, 13, 2);`\
`       formatted_date_str = year "-" month "-" day " " hour ":" minute ":" second;`\
`       cmd = "date -d \"" formatted_date_str "\" +%s";`\
`       cmd | getDOMAINne expiration_date;`\
`       close(cmd);`\
`       if (expiration_date < thirty_days_ago) {`\
`           printf "UID: %s, Password expiration within 30 days: %s\n", uid, formatted_date_str;`\
`       }`\
`   }`\
`'`

### Change pw expiry {#change_pw_expiry}

    # Get a kerberos ticket
    kinit <admin user>

    # Change password expiration
    ipa user-mod <user name> --password-expiration='2072-02-01 01:01:00Z'
    -------------------------------
    Modified user "<user name>"
    -------------------------------
      User login: <user name>
      First name: Service
      Last name: LDAP Join
      Home directory: /tmp/<user name>
      Login shell: /bin/false
      Principal name: <user name>@DOMAIN.LOCAL
      Principal alias: <user name>@DOMAIN.LOCAL
      User password expiration: 20720201010100Z
      Email address: <user name>@domain.local
      UID: 1486850721
      GID: 1486800006
      User authentication types: password
      Account disabled: False
      Password: True
      Member of groups: ipausers, service-accounts
      Roles: Enrollment Administrator
      Kerberos keys available: True
