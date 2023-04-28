---
title: Patch Diffing Basic
author: José Inácio
date: 2023-04-28
categories: [Papers, Reverse]
tags: [Reverse]
pin: true
---

Patch Diffing (Diferença de patch), refere-se a uma técnica usada para comparar diferenças entre duas versões de código-fonte.

Basicamente, o Patch Diffing é uma forma de representar as mudanças em um código-fonte de uma versão para outra. Ao comparar às duas versões, é possível identificar os trechos de código que foram alterados, removidos ou adicionados, seja para identificar correções de bugs, melhorias ou novos recursos. Essa técnica é amplamente utilizada em sistemas de controle de versão de software, como o Git, para permitir que os desenvolvedores colaborem de forma eficiente em um mesmo projeto e façam o acompanhamento das mudanças no código-fonte ao longo do tempo.

Além disso, é útil para realizar análises de segurança em sistemas que possam estar vulneráveis a ataques ou explorações. Por exemplo, pode-se comparar uma versão do software antes e depois de uma correção de segurança ter sido aplicada. Assim, com o entendimento do código vulnerável pode permitir a criação de exploits para exploração da mesma.

Para este exemplo usaremos dois códigos em C simples um contento um vulnerabilidade e outro com sua correção, chamerei de Codigo1 e Codigo2.

Para realizar o Patch Diff, usaremos as ferramentas abaixo;

<h5>Ghidra</h5>
Ghidra é uma ferramenta de engenharia reversa gratuita e de código aberta desenvolvida pela Agência de Segurança Nacional dos Estados Unidos. Neste contexto, o Ghidra foi usado para criar exportações binárias para ambos os arquivos para que eles pudessem ser comparados no BinDiff, vale lembrar que o pluguin "BinExport" deve ser adcionado ao Ghidra.

<h5>BinDiff</h5>
BinDiff é uma ferramenta de análise de binários que é amplamente utilizada por pesquisadores de segurança para comparar duas versões de um software e identificar as diferenças entre elas. Ele é desenvolvido pela Zynamics, uma empresa adquirida pela Google em 2011, e é amplamente utilizado em análises de vulnerabilidades de software, detecção de malware e engenharia reversa. Existem alternativas gratuitas ao BinDiff: DarunGrim e PatchDiff2 são especializados em Patch Diffing.

<h5>Binary Ninja</h5>
Binary Ninja é uma plataforma de engenharia reversa desenvolvida pela Vector 35 Inc. Ela pode desmontar um binário e exibir a desmontagem em visualizações lineares ou gráficas. Ele realiza uma análise profunda automatizada do código, gerando informações que ajudam a analisar um binário.


# BinDiff

Uma boa abordagem ao executar a diferenciação de patches é procurar o par de funções que mostra/contém a maioria das modificações. Isso pode ser feito usando o valor "Similaridade" em BinDiff. A função com o menor valor de similaridade é "vulnerable_function()"

![Desktop View](/img/patchdiff/BINDIFF-SIMI.PNG){: width="700" height="202" .w-75 .normal}

A comparação do gráfico de fluxo

![Desktop View](/img/patchdiff/BINDIFF-VIEW.PNG){: width="700" height="202" .w-75 .normal}

Olhando á função vulnerable_function do Codigo1 pelo BinaryNinja, percebemos que não é feita nenhuma verificação do tamanho da entrada, Além de ultilizar a função strcpy que não verifica o tamanho do buffer de destino antes de copiar os dados da origem para ele. Portanto, se um usuário inserir uma string maior que 10 bytes ocorrerá um BufferOverflow.

![Desktop View](/img/patchdiff/Binaryninja-vunc.PNG){: width="700" height="202" .w-75 .normal}

Já nesta versão corrigida no Codigo2, vemos que o tamanho máximo da string de entrada é limitado a 9 caracteres. Além disso, a função strncpy é utilizada em vez de strcpy. Isso garante que o buffer nunca será preenchido com mais caracteres do que o permitido.

![Desktop View](/img/patchdiff/Binaryninja-vuncOK.PNG){: width="700" height="202" .w-75 .normal}

Em resumo, as técnicas de Patch Diffing são valiosas ferramentas na análise de vulnerabilidades e no desenvolvimento de exploits de dia zero. Ao comparar as mudanças entre duas versões de um software ou sistema, é possível identificar vulnerabilidades introduzidas, corrigidas ou não corrigidas, permitindo que medidas adequadas de segurança sejam tomadas.

## REFERÊNCIA
<https://securityintelligence.com/posts/patch-tuesday-exploit-wednesday-pwning-windows-ancillary-function-driver-winsock>
<https://www.zynamics.com/software.html>
<https://www.linkedin.com/pulse/cve-2023-24869-remote-procedure-call-runtime-code-execution-hawes>
<https://ihack4falafel.github.io/Patch-Diffing-with-Ghidra>

