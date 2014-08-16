---
title: "Automated gmail backup via IMAP"
layout: "post"
permalink: "/2007/11/automated-gmail-backup-via-imap.html"
uuid: "9036663649306703661"
guid: "tag:blogger.com,1999:blog-32090046.post-9036663649306703661"
date: "2007-11-23 15:45:00"
updated: "2007-11-23 19:15:14"
description: 
blogger:
    siteid: "32090046"
    postid: "9036663649306703661"
    comments: "9"
categories: [backup, gmail, imap]
author: 
    name: "rectalogic"
    url: "undefined?rel=author"
    image: "http://img2.blogblog.com/img/b16-rounded.gif"
---

This is how I setup automated gmail backup using IMAP via mbsync. Parts are MacOS X specific.

* Enable IMAP in your gmail account.
* Install [mbsync](http://isync.sourceforge.net/), if using [MacPorts](http://www.macports.org/) do: `sudo port install isync`
* Create a new directory `~/Backup/gmail`
* Save this certificate as `~/Backup/gmail/gmail.pem`. This is the gmail IMAP SSL certificate, retrieved via `openssl s_client -connect imap.gmail.com:993 -showcerts`

    ```
    -----BEGIN CERTIFICATE-----
    MIIDYzCCAsygAwIBAgIQcdBJTwL0s64EVDDexAG1jTANBgkqhkiG9w0BAQUFADCB
    zjELMAkGA1UEBhMCWkExFTATBgNVBAgTDFdlc3Rlcm4gQ2FwZTESMBAGA1UEBxMJ
    Q2FwZSBUb3duMR0wGwYDVQQKExRUaGF3dGUgQ29uc3VsdGluZyBjYzEoMCYGA1UE
    CxMfQ2VydGlmaWNhdGlvbiBTZXJ2aWNlcyBEaXZpc2lvbjEhMB8GA1UEAxMYVGhh
    d3RlIFByZW1pdW0gU2VydmVyIENBMSgwJgYJKoZIhvcNAQkBFhlwcmVtaXVtLXNl
    cnZlckB0aGF3dGUuY29tMB4XDTA3MDUxMTE1NTUzMFoXDTA4MDUxMDE1NTUzMFow
    aDELMAkGA1UEBhMCVVMxEzARBgNVBAgTCkNhbGlmb3JuaWExFjAUBgNVBAcTDU1v
    dW50YWluIFZpZXcxEzARBgNVBAoTCkdvb2dsZSBJbmMxFzAVBgNVBAMTDmltYXAu
    Z21haWwuY29tMIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDp8NJaYAUMfsA4
    uW1y5wXh6sE31Uh/s0dLgeTu4NbzA36Ru3bmpB4zxCNUgnNT73OfhhtTitx8BPoB
    zdY4Tgwc1asVXSw0h2iNoj6/KIiCv4r5FFqWMQxdHZh3n6/VJnNzCPnD62fJy9D8
    j9jIfU6NGM91+zgsEexU7JuHB+y2jwIDAQABo4GmMIGjMB0GA1UdJQQWMBQGCCsG
    AQUFBwMBBggrBgEFBQcDAjBABgNVHR8EOTA3MDWgM6Axhi9odHRwOi8vY3JsLnRo
    YXd0ZS5jb20vVGhhd3RlUHJlbWl1bVNlcnZlckNBLmNybDAyBggrBgEFBQcBAQQm
    MCQwIgYIKwYBBQUHMAGGFmh0dHA6Ly9vY3NwLnRoYXd0ZS5jb20wDAYDVR0TAQH/
    BAIwADANBgkqhkiG9w0BAQUFAAOBgQBIuR0Dr4wURb1CjxMVjWA9/lPZl2f2Kr++
    naPcrIw+gJMLwU88OCfs7XqOHQ/n9dRnQ+mXcrmJKHVQAh0d038JKOaglVBn6LdX
    nJovtY8DyeYPXMGHdIwxPj7H583HQRGqkDF1usr29X3JZxcpPi3ICk+lRYoSHBvH
    /MXIPo3WJA==
    -----END CERTIFICATE-----
    ```

* Save this certificate as `~/Backup/gmail/thawte.pem`. This is the CA for the gmail certificate, downloaded from http://www.thawte.com/roots/.

    ```
    -----BEGIN CERTIFICATE-----
    MIIDJzCCApCgAwIBAgIBATANBgkqhkiG9w0BAQQFADCBzjELMAkGA1UEBhMCWkEx
    FTATBgNVBAgTDFdlc3Rlcm4gQ2FwZTESMBAGA1UEBxMJQ2FwZSBUb3duMR0wGwYD
    VQQKExRUaGF3dGUgQ29uc3VsdGluZyBjYzEoMCYGA1UECxMfQ2VydGlmaWNhdGlv
    biBTZXJ2aWNlcyBEaXZpc2lvbjEhMB8GA1UEAxMYVGhhd3RlIFByZW1pdW0gU2Vy
    dmVyIENBMSgwJgYJKoZIhvcNAQkBFhlwcmVtaXVtLXNlcnZlckB0aGF3dGUuY29t
    MB4XDTk2MDgwMTAwMDAwMFoXDTIwMTIzMTIzNTk1OVowgc4xCzAJBgNVBAYTAlpB
    MRUwEwYDVQQIEwxXZXN0ZXJuIENhcGUxEjAQBgNVBAcTCUNhcGUgVG93bjEdMBsG
    A1UEChMUVGhhd3RlIENvbnN1bHRpbmcgY2MxKDAmBgNVBAsTH0NlcnRpZmljYXRp
    b24gU2VydmljZXMgRGl2aXNpb24xITAfBgNVBAMTGFRoYXd0ZSBQcmVtaXVtIFNl
    cnZlciBDQTEoMCYGCSqGSIb3DQEJARYZcHJlbWl1bS1zZXJ2ZXJAdGhhd3RlLmNv
    bTCBnzANBgkqhkiG9w0BAQEFAAOBjQAwgYkCgYEA0jY2aovXwlue2oFBYo847kkE
    VdbQ7xwblRZH7xhINTpS9CtqBo87L+pW46+GjZ4X9560ZXUCTe/LCaIhUdib0GfQ
    ug2SBhRz1JPLlyoAnFxODLz6FVL88kRu2hFKbgifLy3j+ao6hnO2RlNYyIkFvYMR
    uHM/qgeN9EJN50CdHDcCAwEAAaMTMBEwDwYDVR0TAQH/BAUwAwEB/zANBgkqhkiG
    9w0BAQQFAAOBgQAmSCwWwlj66BZ0DKqqX1Q/8tfJeGBeXm43YyJ3Nn6yF8Q0ufUI
    hfzJATj/Tb7yFkJD57taRvvBxhEf8UqwKEbJw8RCfbz6q1lu1bdRiBHjpIUZa4JM
    pAwSremkrj/xw0llmozFyD4lt5SZu5IycQfwhl7tUCemDaYj+bvLpgcUQg==
    -----END CERTIFICATE-----
    ```

* Save this config file in `~/Backup/gmail/mbsync-config`. Replace LOGIN with your gmail username. This sets up a one-way channel from your gmail *All Mail* store to a local maildir. So the local maildir will be kept in sync with all gmail mail.

    ```
    MaildirStore local
    Path ~/Backup/gmail/maildir/

    IMAPStore gmail
    Host imap.gmail.com
    User LOGIN@gmail.com
    UseIMAPS yes
    CertificateFile ~/Backup/gmail/gmail.pem
    CertificateFile ~/Backup/gmail/thawte.pem

    Channel backup
    Master ":gmail:[Gmail]/All Mail"
    Slave :local:gmail
    Create Slave
    Expunge Slave
    Sync Pull
    ```

* Save your gmail password in [Keychain](http://en.wikipedia.org/wiki/Apple_Keychain). The simplest way to do this is to login to your gmail account and have Safari remember the password.
* Give security permission to access this password. Run this command (replace LOGIN with your gmail username) and when prompted click *Always Allow* to allow security access. `security find-internet-password -g  -a `**LOGIN**` -s www.google.com`
* Save this script as `~/Backup/gmail/backup-gmail` and make it executable `chmod +x backup-gmail`. Replace LOGIN with your gmail username. This script uses [security](http://developer.apple.com/documentation/Darwin/Reference/ManPages/man1/security.1.html) to retrieve your gmail password from Keychain - to avoid storing it in plain text in the config file. mbsync uses [getpass](http://www.gnu.org/software/libc/manual/html_node/getpass.html) to read the password directly from the TTY, so this won't work when you run backup-gmail directly from Terminal. It will work when run via launchd (see below).

    ```bash
    #!/bin/bash
    security find-internet-password -g  -a LOGIN -s www.google.com 2>&1 |\
        sed -n -e '1s/password: "\(.*\)"/\1/;1p' |\
        /opt/local/bin/mbsync --config ~/Backup/gmail/mbsync-config backup 2>&1 > ~/Backup/gmail/mbsync.log
    ```

* Save this launchd plist as `~/Backup/gmail/com.rectalogic.gmail.backup.plist` and then symlink it into your LaunchAgents directory `ln -sf ~/Backup/gmail/com.rectalogic.gmail.backup.plist ~/Library/LaunchAgents`. This places the backup script under launchd control and will run it every 24 hours. You can run it manually to test via `launchctl start com.rectalogic.gmail.backup`. Monitor `~/Backup/gmail/mbsync.log` for errors.

    ```XML
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
     <key>Label</key>
      <string>com.rectalogic.gmail.backup</string>
     <key>OnDemand</key>
      <true/>
     <key>Program</key>
      <string>~/Backup/gmail/backup-gmail</string>
     <key>StartInterval</key>
      <integer>86400</integer>
    </dict>
    </plist>
    ```
