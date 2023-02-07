---
title: Executando aplicações GUI em container no Linux
author: Hastur
date: 2022-04-29 01:00:00 -0300
categories: [Ferramentas, Tutoriais]
tags: [Docker, Container, Tutorial, Tool]
image: "/img/posts/gui_docker.jpg"
alt: "Docker"
---

Neste artigo, vamos entender o processo que permite que executemos aplicações que utilizam GUI de dentro de um container.

Este tipo de ação se torna útil no dia-a-dia, ainda falando em sec, pois permite que tenhamos todas as ferramentas necessárias para um teste ou análise de vulnerabilidade, sem que haja a necessidade de instalar ferramentas diretamente no SO ou o uso de virtualização.

O sistema final fica muito leve em comparação a um SO sendo executado em sua totalidade, seja como host ou virtualizado, além de ser extremamente minimalista, instalando somente as ferramentas necessárias. O que o torna performático e isolado do SO host (nem tanto).

Isso nos permite invocar aplicações tanto de terminal quanto GUI integradas ao host. Imagine chamar um `Burpsuite` de um container, e interceptar requisições do navegador do host, assim como invocar o `Wireshark` do container, para monitorar a rede do host.

O intuito deste artigo, não é demonstrar como funciona o `Docker`, ainda mais porque seus recursos são extensos, mas sim demonstrar como podemos utilizá-lo para fins de otimização de testes.

Porém, o que pra muitos pode ser banal, é de extrema importância que abordemos os conceitos básicos envolvidos em todo este processo.

# Containers

Dentro do unverso Linux, um container é uma tecnologia que permite isolar e empacotar uma aplicação e todo o seu ambiente em tempo de execução, ou seja todo o sistema de base e os arquivos necessários para execução de uma aplicação, são executados de forma isolada do sistema operacional. 

Isto facilita a movimentação da aplicação em questão, entre diferentes ambientes, pois passa a não depender de bibliotecas e versões de sistemas operacionais específicos. No cenário de desenvolvimento, um container pode transitar entre ambientes de dev, QA e prod mantendo sua integridade completa, independente da infraestrutura de cada ambiente.

Esta independência torna possível a execução de vários processos separadamente uns dos outros tendo melhor aproveitamento dos recursos de hardware e "melhorando" a segurança, ao mentê-los digitalmente em ambientes diferentes.

# Docker

`Docker` é uma tecnologia que utiliza o [kernel Linux](https://www.educative.io/edpresso/what-is-linux-kernel) e seus recursos, para segregar processos de forma que sejam executados de forma isolada e independente. O modelo de implantação do Docker se baseia em imagens, ou seja, para cada sistema e/ou aplicação, existe uma imagem que contém todo o ambiente para seu funcionamento.

![docker](/img/posts/gui_docker_01.png)

Além desta função primordial, o Docker traz uma série de funcionalidades que permitem o fácil gerenciamento dos containers, automatizando implantações, compartilhamento de recursos, arquivos e diretórios entre o container e o host, compartilhamento ou isolamento de redes entre diversos outros.

Entre os benefícios de utilizarmos um gerenciador de containers, podemos citar:

* Controle de versões de imagens
* Modularidade
* Escalabilidade

Existem outras ferramentas que oferecem as mesmas funcionalidades como `Kubernetes` e o `CRI-O`. Porém, devido ao seu ambiente "amigável", vamos focar este artigo em `Docker`. Podendo afirmar que a tecnologia Docker tem uma abordagem controlável, baseada em microserviços e eficiente.

# X11

`X Window System`, também chamado de `X11`, é um sistema de janelas *client/server* para exibição de bitmaps. O X11 é comumente implantado na maioria dos sistemas operacionas baseados em `UNIX` e já foi portado até mesmo para outros sistemas.

O *server* X11, de forma bem resumida, pode ser considerado como o sistema que exibe as janelas e manipula os dispositivos de entrada, como mouses, teclados e telas *touch screen*. Já os *clients* são os aplicativos em execução.

O X11 utiliza arquivos `UNIX Socket` que agem na comunicação entre processos dentro de uma mesma máquina de forma eficiente. O próprio [manual do unix socket](https://man7.org/linux/man-pages/man7/unix.7.html) o descreve como:

> *The AF_UNIX (also known as AF_LOCAL) socket family is used to communicate between processes on the same machine efficiently. Traditionally, UNIX domain sockets can be either unnamed, or bound to a filesystem pathname (marked as being of type socket). Linux also supports an abstract namespace which is ndependent of the filesystem.*

Assim como vários tipos de servidores, o X11 também trabalha com sistema de permissionamento, do qual pode ser gerenciado pelo comando `xhost`.

O `xhost` de acordo com seu [manual](https://linux.die.net/man/1/xhost) é o programa utilizado para adicionar e deletar *host names* ou *user names* da lista de permissões do X *server*.

Por padrão, o X *server* permite que somente o usuário local utilize seus recursos, é possível confirmar isso ao executar o comando `sudo xhost`.

![docker](/img/posts/gui_docker_02.png)

Conforme podemos ver, somente o usuário logado e seus processos tem permissão de utilizar o X11. Porém, é preciso permitir que toda a família de usuários locais, possam utilizar o X *server*. Para isso, pode-se utiliar o comando `xhost +local:*`.

![docker](/img/posts/gui_docker_03.png)

Como podemos ver, agora temos o `LOCAL` entre os usuários permitidos. Esta configuração é resetada toda vez que o sistema operacional é reinicializado.

Além destas configurações, existe uma variável de ambiente extremamente importante neste processo, a `$DISPLAY`. Esta variável de ambiente é utilizada pelo X11 para identificar nossos dispositivos de IO e sua interação com a tela. Normalmente esta variável de ambiente contém o valor `:0` em dispositivos Desktop, referenciando o monitor primário. Quando se utiliza uma sessão SSH com conexão X, o valor desta variável pode ser um número alto, pois ela indica para o X *server* que as aplicações devem receber seu *input* e *output* de conexões externas. Conforme observado abaixo.

![docker](/img/posts/gui_docker_04.png)

Por ultimo, é preciso encontrar o próprio `UNIX Socket` do X *server*. Este arquivo de socket pode ser encontrado no diretório `/tmp` conforme mostrado abaixo.

![docker](/img/posts/gui_docker_05.png)

Normalmente em sistemas baseados em UNIX, toda vez que o X *server* se inicia junto com o sistema operacional, este diretório é criado.

# Criando uma imagem personalizada

Quando fazemos o `pull` de uma imagem Docker, estamos basicamente capturando a imagem que contém somente os arquivos necessários para o funcionamento daquela aplicação. Por exemplo, uma imagem do servidor web `Apache`, virá somente com o kernell de uma distribuição Linux e os arquivos necessários para o funcionamento do próprio Apache, tornando a imagem leve o suficiente para ter somente alguns mega bytes.

Já quando fazemos o pull de uma imagem de uma distribuição Linux pura, por exemplo, estamos baixando somente o kernel compilado e um emulador de terminal, tornando a imagem extremamente leve.

Uma imagem de container pode ser usada para criar novas imagens personalizadas que contenha as instalações que precisamos, e cada imagem pode ser usada para o *deploy* de quantos containers forem necessários. E esta é a grande vantagem em relação ao minimalismo. Uma distribuição que contém somente o necessário e mais nada.

Para a prova de conceito deste artigo, vamos utilizar a imagem da distribuição Kali Linux que pode ser encontrada no [Docker Hub](https://hub.docker.com/r/kasmweb/core-kali-rolling). Esta imagem é atualizada constantemente e contém somente o *core* do Kali, sem absolutamente nenhuma ferramenta.

Para melhor gerenciamento e controle ao criar uma imagem, um dos recursos do Docker é o `Dockerfile`. Basicamente é um arquivo onde configuramos como queremos montar uma imagem, sua referência oficial pode ser encontrada [aqui](https://docs.docker.com/engine/reference/builder/). O script abaixo mostra o conteúdo do exemplo que iremos utilizar.

```docker
# informando qual a imagem base a ser utilizada
FROM kalilinux/kali-rolling

# criando um diretório de trabalho
WORKDIR /resources

# update da imagem
RUN apt update

# instalação de aplicações importantes para o X11
RUN apt install dbus-x11 packagekit-gtk3-module libcanberra-gtk3-0 -y

# instalando programas de teste
RUN apt install firefox-esr burpsuite -y
```

Onde neste arquivo:

1. `kalilinux/kali-rolling`: indica qual a imagem será utilizada como base para uma imagem personalizada
2. `/resources`: será o diretório de trabalho desta imagem, isso significa que toda vez que um container for invocado a partir desta imagem, o diretório principal de trabalho será esse. (podemos montar um volume do host neste diretório para compartilharmos recursos)
3. `dbus-x11`: é o *add-on* necessário para o D-Bus no X11. O D-Bus é um mecanismo de *middleware* que permite a comunicação entre multiplos processos executando simultaneamente na mesma máquina. Neste caso, ele fará este papel no X11, entre o host e o container.
4. `packagekit-gtk3-module`: é um pacote de fontes para melhorar a experiência.
5. `libcanberra-gtk3-0`: é a implementação que vai gerar sons de eventos em aplicações GUI, mais um pacote para melhorar a experiência.

Como não é possível a interação com o usuário durante a construção de uma imagem, é preciso que todas as instalações possuam a flag `-y` para que o não seja solicitada a confirmação. Também é importante que o primeiro comando a ser executado seja o `update` da distribuição, para garantir que os pacotes sejam carreegados do repositório.

Para fins de teste, vamos instalar somente o `Firefox` e o `Burpsuite`, após a comprovação da prova de conceito, podemos montar uma imagem com ferramentas do dia-a-dia.

Com o arquivo configurado, podemos executar o comando `sudo docker build -t kali .` de dentro do diretório onde o Dockerfile está.  
Neste caso, o `build` informa ao Docker para construir uma imagem, a flag `-t` diz para o Docker que vamos dar um nome para a imagem, neste caso `kali` e o `.` indica que é para buscar o Dockerfile no diretório atual.

![docker](/img/posts/gui_docker_06.png)

Como podemos ver, a primeira coisa que o Docker faz, é o `pull` da imagem do Kali Linux

![docker](/img/posts/gui_docker_07.png)

Logo após, ele inicia os comandos para *update* e instalação dos programas, este passo pode demorar um pouco.

Após a execução de todos os processos, podemos consultar as imagens existentes e verificar que a imagem `kali` foi criada, conforme mostrado abaixo.

![docker](/img/posts/gui_docker_08.png)

Neste ponto, temos uma imagem de Kali Linux extremamente minimalista para testes que contém somente os programas Firefox e Burpsuite.

# Invocando o bash de um container

Containers são dinâmicos, podem ser criados, destruídos, inicializados, parados, movidos e alterados.  
Podemos compartilhar recursos entre o host e um container, assim como podemos isolá-lo totalmente.

Quando inciamos um container, ele vai ler a imagem base e iniciar a aplicação invocada permanecendo em operação até que seja parado de alguma forma. Aí entra uma granda cautela necessária, se invocarmos várias aplicações de uma imagem, vários contaners serão criados e permanecerão em execução consumindo recursos, a menos que sejam parados ou destruídos.

Para lidar com este tipo de situação, podemos utilizar uma série de flags ao invocar um container. Abaixo, exemplifico a forma que `EU` utilizo na istuação específica abordada neste artigo.

```bash
sudo docker run --rm -it kali bash
```

Onde:

1. `run`: é o comando para o deploy de um container a partir de uma imagem.
2. `--rm`: esta é a flag importante, ela indica para o Docker, que após o encerramento do programa ou aplicação invocada, este container deve se auto destruir, desocupando a memória e o espaço em disco. Isso faz com que não seja necessária a preocupação com vários containers reduntantes executando em paralelo sem uso e torna as alterações não permanentes, ou seja, o container sempre será executado no estado inicial do Kali.
3. `-it`: a flag que faz o container ficar "interativo" (`-i` mantém o STDIN ativo e `-t` aloca um pseuto TTY).
4. `kali`: o nome da imagem que utilizaremos para invocar o container.
5. `bash`: o programa que queremos invocar, neste caso um simples terminal bash.

![docker](/img/posts/gui_docker_09.png)

Até então, tudo funcionando normalmente como qualquer container, porém, como utilizamos a flag `--rm` ao executar o comando `exit`, o container se auto destrói e nenhuma alteração é persistente.

Neste primeiro comando, utilizamos o comando `bash` para invocar o terminal, mas com todas as configurações que fizemos, podemos agora invocar um programa que utiliza GUI

# Invocando um programa GUI de um container

Conforme entendemos sobre o X11, precisamos compartilhar o recurso de `UNIX Socket` que se encontra em `/tmp/` entre o container e o host. O Docker, permite que compartilhemos diretórios e arquivos através de volumes, com este recurso, podemos "montar" um diretório do host em qualquer lugar do container.  
Também precisamos compartilhar a variável de ambiente `$DISPLAY` que o X11 irá utilizar para saber onde mandar a aplicação GUI. O comando fica desta forma:

```bash
sudo docker run --rm -it -v /tmp/.X11-unix:/tmp/.X11-unix -e DISPLAY=$DISPLAY -d kali firefox
```

Onde:

1. `-v /tmp/.X11-unix:/tmp/.X11-unix`: a flag `-v` monta um volume da máquina host para um diretório do container, no caso estamos montando o diretório `/tmp/.X11-unix` do host para o mesmo caminho dentro do container.
2. `-e DISPLAY=$DISPLAY`:  a flag `-e` cria uma variável de ambiente no container com o valor que passarmos, no caso estamos criando a `DISPLAY` dentro do container com o mesmo valor da `DISPLAY` do host.
3. `-d`: esta flag faz com que a execução ocorra em background sem comprometer o terminal.

Ao executar o comando, temos o Firefox invocado diretamente do container.

![docker](/img/posts/gui_docker_10.png)

Todo este processo, torna o container menos isolado do host, porém o objetivo desta prova de conceito não é subir uma aplicação, mas sim chamar aplicações GUI que possam ajudar no dia-a-dia sem que haja a necessidade da instalação na máquina host.

Ainda é possível compartilhar mais recursos com o container, para interagir com o host, por exemplo, podemos compartilhar as mesmas interfaces de rede do host com o container, e utilizar o Burpsuite do container para interceptar requisições do browser do host, podemos fazer isto com a flag `--net=host`. O comando fica desta forma:

```bash
sudo docker run --rm -it -v /tmp/.X11-unix:/tmp/.X11-unix -e DISPLAY=$DISPLAY --net=host --privileged -d kali burpsuite
```

Ao executar o comando e chamar o Firefox do host, podemos interagir entre as aplicações.

![docker](/img/posts/gui_docker_11.png)

Caso seja necessário persistir algum dado, ou compartilhar algum arquivo ou diretório entre o container e o host, podemos utilizar a flag `-v` novamente e montar um novo compartilhamento. Na verdade, esta foi a real razão da qual o comando `WORKDIR /resource` foi inserida no `Dockerfile`, pois podemos montar um diretório compartilhado lá de forma mais organizada.

# Automatizando a chamada

Como o comando fica relativamente grande, fiz um script para automatizar esta chamada onde, a depender do argumento, ele toma uma ação diferente, como chamar um terminal ou abrir uma aplicação GUI.

```bash
#!/bin/bash

dir=$HOME/pentest/
xh=$(sudo xhost | grep LOCAL | wc -l)

if [ $xh -eq 0 ]
then
		sudo xhost +Local:* >/dev/null
fi

if [ "$1" == "" ]
then
		echo -e "Use:\n\t$0 <command>"
		echo -e "Ex:\n\t$0 bash"
		echo -e "\t$0 burpsuite"
elif [ "$1" == "bash" ]
then
		sudo docker run --rm -it -v $dir:/resources -v /tmp/.X11-unix/:/tmp/.X11-unix/ --net=host --privileged -e DISPLAY=$DISPLAY kali $1
else
		sudo docker run --rm -v $dir:/resources -v /tmp/.X11-unix/:/tmp/.X11-unix/ --net=host -e DISPLAY=$DISPLAY --privileged -d kali $1 >/dev/null
fi
```
Este script nos permite chamar tanto o bash:

![docker](/img/posts/gui_docker_12.png)

Como chamar uma aplicação GUI:

![docker](/img/posts/gui_docker_13.png)

# Melhorando a utilidade

Neste artigo, fizemos um treste simples ao criar uma imagem que contém somente o Firefox e o Burpsuite, porém, esta imagem pode ser construida com toda e qualquer *tool*  necessária para o dia-a-dia, tanto com aplicações GUI quanto programas de terminal tornando versátil o uso do Kali Linux em ambientes distintos. Tudo a depender de como o Dockerfile é configurado.

Eu fiz um repositório no GitHub com a construção e automação deste recurso com algumas ferramentas mais habituais, o recurso pode ser encontrado no link abaixo:

* [KALI CONTAINER](https://github.com/h41stur/kali_container)

Espero que tenha ajudado de alguma forma!

HACK THE PLANET!!
