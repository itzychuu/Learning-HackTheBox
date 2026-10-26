Section 6 Basic Tools

# Service Scanning --- Capsule Notes

> **Purpose:** Compact revision notes for identifying hosts, discovering
> services, fingerprinting versions, and enumerating FTP, SMB, and SNMP.
>
> **Lab safety:** Run scans and enumeration only against systems you own
> or are explicitly authorized to assess. IP addresses and credentials
> below are examples copied from the supplied material.

## 1. Core Concepts

-   **Service:** An application running on a computer that provides a
    function to other users/computers.
-   **Server:** A machine hosting services.
-   **Service enumeration:** Gathering information about exposed
    services, versions, configuration, shares, and possible weaknesses.
-   **IP address:** Identifies a host on a network.
-   **Port:** Identifies a network service endpoint.
-   **Port range:** `1–65535`; well-known ports are `1–1023`.
-   **Port 0:** Reserved in TCP/IP; not used as a normal TCP/UDP
    destination port. Binding to port `0` commonly asks the OS to choose
    an available ephemeral port.
-   Remote access requires the correct **IP + port + protocol/client
    understood by the service**.

### Common ports in this lesson

  ------------------------------------------------------------------------
                          Port Service               Typical use / clue
  ---------------------------- --------------------- ---------------------
                        21/TCP FTP                   File transfer; check
                                                     for anonymous access

                        22/TCP SSH                   Remote shell; common
                                                     on Unix/Linux, but
                                                     also available on
                                                     Windows

                        80/TCP HTTP                  Web server

                       139/TCP NetBIOS session       Often associated with
                               service               SMB

                       445/TCP SMB / Microsoft-DS    Windows file sharing;
                                                     Samba may expose it
                                                     on Linux

                      3389/TCP RDP                   Remote Desktop; often
                                                     suggests Windows
  ------------------------------------------------------------------------

**Important:** A port-to-service mapping is only a clue. The actual
service can differ, so fingerprint it rather than assuming.

## 2. Nmap Essentials

Nmap (Network Mapper) automates port scanning and service discovery.

### Basic scan

``` bash
nmap 10.129.42.253
```

-   By default, scans the **1,000 most common ports**.
-   Performs a TCP scan unless a different scan type is requested.
-   Example output from the source:

``` text
PORT    STATE SERVICE
21/tcp  open  ftp
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

### Reading Nmap output

-   **PORT:** Port number and transport protocol, e.g. `21/tcp`.
-   **STATE:** Whether the port is `open`, `closed`, or `filtered`.
    -   `open`: A service appears to be accepting connections.
    -   `closed`: The host is reachable, but no service is listening on
        that port.
    -   `filtered`: Nmap cannot determine the state reliably, often
        because a firewall/filter blocks probes.
-   **SERVICE:** Name commonly associated with the port; not definitive
    proof of the running application.
-   **VERSION:** Appears when version detection identifies
    service/product details.

### Scan all TCP ports, run default scripts, and detect versions

``` bash
nmap -sV -sC -p- 10.129.42.253
```

  Option   Meaning
  -------- ------------------------------------------------------
  `-sV`    Probe open ports to identify service/product/version
  `-sC`    Run Nmap's default script set
  `-p-`    Scan TCP ports 1--65535

This scan can take significantly longer than a default scan because it
checks every TCP port and performs additional script/version probes.

### Example findings from the detailed scan

``` text
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 3.0.3
22/tcp  open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.1
80/tcp  open  http        Apache httpd 2.4.41 ((Ubuntu))
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2
```

Other example script results: - FTP anonymous login was allowed and a
`pub` directory was listed. - HTTP title: `PHP 7.4.3 - phpinfo()`. - SMB
hostname: `GS-SVCSCAN`. - SMB2 message signing was enabled but not
required.

### OS inference from service versions

-   Example: `OpenSSH 8.2p1 Ubuntu 4ubuntu0.1` can be cross-referenced
    with package information to infer Ubuntu 20.04 (Focal Fossa).
-   Ubuntu 20.04's release date is shown in the source as **23 April
    2020**.
-   **Caution:** Version-based OS inference is not conclusive; newer
    packages can be installed on older operating systems.
-   A PHPInfo page can reveal PHP version and server/configuration
    details. Example URL:

``` text
http://10.129.42.253/index.php
```

## 3. Nmap Scripting Engine (NSE)

-   `-sC` runs useful default scripts.
-   Specific scripts can enumerate a service or check for a known
    configuration/vulnerability.
-   The source gives Citrix-related NSE scripts as an example.

### Find available Citrix scripts

``` bash
locate scripts/citrix
```

Example paths returned:

``` text
/usr/share/nmap/scripts/citrix-brute-xml.nse
/usr/share/nmap/scripts/citrix-enum-apps-xml.nse
/usr/share/nmap/scripts/citrix-enum-apps.nse
/usr/share/nmap/scripts/citrix-enum-servers-xml.nse
/usr/share/nmap/scripts/citrix-enum-servers.nse
```

### General NSE syntax

``` bash
nmap --script <script name> -p<port> <host>
```

Use the script appropriate to the service and authorized assessment
objective.

## 4. Banner Grabbing

**Banner grabbing** means collecting identification text returned by a
service, often including its product/version.

### Use Nmap

``` bash
nmap -sV --script=banner <target>
```

### Connect manually with Netcat

``` bash
nc -nv 10.129.42.253 21
```

Example response:

``` text
(UNKNOWN) [10.129.42.253] 21 (ftp) open
220 (vsFTPd 3.0.3)
```

This banner identifies the FTP server as `vsFTPd 3.0.3`.

### Scan a subnet for FTP banners (source example)

``` bash
nmap -sV --script=banner -p21 10.10.10.0/24
```

-   `-p21` restricts the scan to port 21.
-   `10.10.10.0/24` specifies the example subnet.
-   Only use subnet scanning within an authorized scope.

## 5. FTP Enumeration

**FTP (File Transfer Protocol)** is commonly associated with TCP port
`21`. Enumeration checks the server version, authentication
configuration, directories, and accessible files.

### Identify FTP and check anonymous access

``` bash
nmap -sC -sV -p21 10.129.42.253
```

Example result:

``` text
21/tcp open  ftp vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 ftp      ftp          4096 Dec 19 23:50 pub
```

`Anonymous FTP login allowed` means the server accepts anonymous
authentication. The `pub` directory is visible in this example.

### Connect to FTP

``` bash
ftp -p 10.129.42.253
```

-   `-p` requests passive mode.
-   At the username prompt, the example uses `anonymous`.

### Common FTP interactive commands

  -----------------------------------------------------------------------
  Command                             Purpose
  ----------------------------------- -----------------------------------
  `ls`                                List remote directory contents

  `cd pub`                            Change to the remote `pub`
                                      directory

  `get login.txt`                     Download `login.txt` to the local
                                      working directory

  `exit`                              End the FTP session
  -----------------------------------------------------------------------

### Source example session

``` text
ftp> ls
drwxr-xr-x    2 ftp      ftp          4096 Feb 25 19:25 pub

ftp> cd pub
ftp> ls
-rw-r--r--    1 ftp      ftp            18 Feb 25 19:25 login.txt

ftp> get login.txt
18 bytes received in 0.00 secs (165.8314 kB/s)

ftp> exit
221 Goodbye.
```

Inspect the downloaded file locally:

``` bash
cat login.txt
```

The supplied example file contains:

``` text
admin:ftp@dmin123
```

**Security lesson:** Anonymous access and exposed credential files can
lead to further access. Treat discovered credentials as sensitive; do
not reuse or attempt them outside the authorized lab.

## 6. SMB Enumeration

**SMB (Server Message Block)** supports network file/printer sharing and
is common on Windows. Linux systems may provide SMB through **Samba**.

Potential findings include: - Shared folders containing sensitive files
or credentials. - OS, hostname, workgroup, and Samba version. - Weak
share permissions or legacy SMB vulnerabilities. - Some legacy SMB
versions/configurations may be exposed to vulnerabilities such as
EternalBlue; verify carefully rather than assuming vulnerability from
the port alone.

### Discover OS details with Nmap

``` bash
nmap --script smb-os-discovery.nse -p445 10.10.10.40
```

Example output:

``` text
445/tcp open  microsoft-ds

Host script results:
| smb-os-discovery:
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: CEO-PC
|   NetBIOS computer name: CEO-PC
|   Workgroup: WORKGROUP
```

This example reports Windows 7 Professional. Additional authorized
checks would be needed to determine whether a particular vulnerability
is present.

### Enumerate SMB service details

``` bash
nmap -A -p445 10.129.42.253
```

-   `-A` enables a collection of advanced detection features, including
    OS/version detection, scripts, and traceroute.
-   Example findings: Samba `4.6.2`, hostname `GS-SVCSCAN`, and SMB2
    message signing enabled but not required.
-   OS guesses may be unreliable when scan conditions are limited.

### List available shares without a password prompt

``` bash
smbclient -N -L \\\\10.129.42.253
```

  Option   Meaning
  -------- --------------------------------
  `-L`     List shares on the remote host
  `-N`     Suppress the password prompt

Example share listing:

``` text
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
users           Disk
IPC$            IPC       IPC Service (gs-svcscan server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
```

The example identifies a non-default share named `users`.

### Connect to a share

``` bash
smbclient \\\\10.129.42.253\\users
```

The source's unauthenticated attempt returns:

``` text
smb: \> ls
NT_STATUS_ACCESS_DENIED listing *
```

This indicates that listing the share was denied without suitable
access.

### Connect using the example lab account

``` bash
smbclient -U bob \\\\10.129.42.253\\users
```

The source example uses the lab credential `bob:Welcome1`. After
connecting, the commands are:

``` text
smb: \> ls
smb: \> cd bob
smb: \bob\> ls
smb: \bob\> get passwords.txt
```

The example share contains `bob/passwords.txt`, which is downloaded with
`get`.

**SMB command recap** - `smbclient -N -L ...` --- list shares without
prompting for a password. - `smbclient -U <user> ...` --- connect using
a named user. - `ls` --- list remote files/directories. -
`cd <directory>` --- change remote directory. - `get <file>` ---
download a remote file. - `exit` --- disconnect.

## 7. SNMP Enumeration

**SNMP (Simple Network Management Protocol)** exposes device management
information and statistics.

-   SNMP v1/v2c commonly use a **community string** as the access value,
    transmitted in plaintext.
-   Default community strings such as `public` and `private` may be left
    unchanged.
-   SNMPv3 adds authentication and encryption capabilities.
-   Useful information can include system names, software versions,
    process parameters, routing information, and services bound to other
    interfaces.
-   Process parameters may accidentally expose credentials; credentials
    may be reused across services.

### Query the system name with `snmpwalk`

``` bash
snmpwalk -v 2c -c public 10.129.42.253 1.3.6.1.2.1.1.5.0
```

  Part                  Meaning
  --------------------- ---------------------------------------
  `-v 2c`               Use SNMP version 2c
  `-c public`           Use the `public` community string
  `1.3.6.1.2.1.1.5.0`   OID for the system name (`sysName.0`)

Example result:

``` text
iso.3.6.1.2.1.1.5.0 = STRING: "gs-svcscan"
```

### Example unsuccessful query

``` bash
snmpwalk -v 2c -c private 10.129.42.253
```

Example response:

``` text
Timeout: No Response from 10.129.42.253
```

A timeout does not, by itself, prove why the query failed.

### Check common community strings with `onesixtyone`

The source describes using `onesixtyone` with a dictionary file of
community strings.

``` bash
onesixtyone -c dict.txt 10.129.42.254
```

Example result:

``` text
Scanning 1 hosts, 51 communities
10.129.42.254 [public] Linux gs-svcscan 5.4.0-66-generic #74-Ubuntu SMP Wed Jan 27 22:54:38 UTC 2021 x86_64
```

Here, `-c dict.txt` supplies the candidate community strings. Use this
only in an explicitly authorized lab or assessment.

## 8. Fast Command Reference

  ------------------------------------------------------------------------------------------------
  Goal                                Command
  ----------------------------------- ------------------------------------------------------------
  Default Nmap scan                   `nmap 10.129.42.253`

  All TCP ports + versions + default  `nmap -sV -sC -p- 10.129.42.253`
  scripts                             

  Find Citrix NSE scripts             `locate scripts/citrix`

  Run a specific NSE script           `nmap --script <script name> -p<port> <host>`

  Grab a service banner with Nmap     `nmap -sV --script=banner <target>`

  Connect to FTP banner manually      `nc -nv 10.129.42.253 21`

  Scan FTP port on example subnet     `nmap -sV --script=banner -p21 10.10.10.0/24`

  Enumerate FTP                       `nmap -sC -sV -p21 10.129.42.253`

  Connect to FTP                      `ftp -p 10.129.42.253`

  Read downloaded text file           `cat login.txt`

  SMB OS discovery                    `nmap --script smb-os-discovery.nse -p445 10.10.10.40`

  Aggressive SMB scan                 `nmap -A -p445 10.129.42.253`

  List SMB shares                     `smbclient -N -L \\\\10.129.42.253`

  Connect to SMB share                `smbclient \\\\10.129.42.253\\users`

  Connect as SMB user                 `smbclient -U bob \\\\10.129.42.253\\users`

  Query SNMP system name              `snmpwalk -v 2c -c public 10.129.42.253 1.3.6.1.2.1.1.5.0`

  Test another example SNMP community `snmpwalk -v 2c -c private 10.129.42.253`

  Try community strings from          `onesixtyone -c dict.txt 10.129.42.254`
  dictionary                          
  ------------------------------------------------------------------------------------------------

## 9. Exam / Lab Takeaways

1.  Start with a basic Nmap scan, then expand to all ports and
    service/version detection as needed.
2.  Do not treat a port's conventional service name as proof of what is
    actually running.
3.  `-sV` identifies service versions; `-sC` runs default scripts; `-p-`
    scans all TCP ports.
4.  Banners, HTTP titles, and service versions can help fingerprint
    software and sometimes suggest an OS---but inferences can be wrong.
5.  FTP enumeration checks anonymous login, directories, and
    downloadable files.
6.  SMB enumeration checks OS details, available shares, authentication,
    permissions, and interesting files.
7.  SNMP can reveal system and network information; weak/default
    community strings are a configuration risk.
8.  Record findings accurately and validate them. An open port or
    version number alone does not confirm a vulnerability.
9.  Perform scanning, authentication checks, and enumeration only with
    explicit authorization.

## 10. Source Coverage Note

These capsule notes are based on the supplied **Service Scanning**
material. Commands, principal examples, and the example outputs relevant
to Nmap, NSE, banner grabbing, FTP, SMB, and SNMP have been retained in
a compact study format. Explanatory wording has been condensed for
revision.


1 - SSH (Secure Shell)
Secure Shell (SSH) is a network protocol that runs on port 22 by default and provides users such as system administrators a secure way to access a computer remotely.
eg:
ItzKai1920@htb[/htb]$ ssh Bob@10.10.10.10
Bob@remotehost's password: *********

2 - Netcat
Netcat, ncat, or nc, is an excellent network utility for interacting with TCP/UDP ports. It can be used for many things during a pentest
Banner Grabbing using netcat
Syntax; netcat <ip_addr> <port_no>
eg:
ItzKai1920@htb[/htb]$ netcat 10.10.10.10 22
SSH-2.0-OpenSSH_8.4p1 Debian-3

First task in section 6
Grab the banner of a target ip with portno using pwnbox
command: netcat <ip_addr> <port_no>

Section 7
Question 1:
  nmap -sV -p8080 <ip_addr>

Question 2:
  nmap -A <ip_addr>

Question 3:
  smbclient -N -L \\\\<ip_addr>
  smbclient -U bob\\\\<ip_addr>\\<list_from_the_previous_comment>
  enter password: Welcome1
  ls
  cd flag
  get flag.txt
  exit 
  cat flag.txt
  
  
