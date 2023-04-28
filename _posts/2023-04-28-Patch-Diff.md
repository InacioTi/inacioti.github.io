
---
title: Patch Diff
author: José Inácio
date: 2023-04-28
categories: [Papers, Reverse]
tags: [Reverse]
pin: true
---

Patch Diff (Diferença de patch), refere-se a uma técnica usada para comparar diferenças entre duas versões de código-fonte.

Basicamente, o Patch Diff é uma forma de representar as mudanças em um código-fonte de uma versão para outra. Ao comparar às duas versões, é possível identificar os trechos de código que foram alterados, removidos ou adicionados, seja para identificar correções de bugs, melhorias ou novos recursos. Essa técnica é amplamente utilizada em sistemas de controle de versão de software, como o Git, para permitir que os desenvolvedores colaborem de forma eficiente em um mesmo projeto e façam o acompanhamento das mudanças no código-fonte ao longo do tempo.

Além disso, é útil para realizar análises de segurança em sistemas que possam estar vulneráveis a ataques ou explorações. Por exemplo, pode-se comparar uma versão do software antes e depois de uma correção de segurança ter sido aplicada. Assim, com o entendimento do código vulnerável pode permitir a criação de exploits para exploração da mesma.

Para este exemplo usaremos dois códigos em C simples;

O codigo abaixo contem uma vulnerabilidade de BufferOverFlow;

```C
#include <stdio.h>
#include <string.h>


void vulnerable_function(char* input) {
    char buffer[10];
    strcpy(buffer, input);
    printf("Input received: %s\n", buffer);
}

int main() {
    char input_string[20];
    printf("Enter a string: ");
    scanf("%s", input_string);
    vulnerable_function(input_string);
    return 0;
}
```

Já o codigo abaixo contem o seu "patch" de correção;

```C
#include <stdio.h>
#include <string.h>

void vulnerable_function(char* input, size_t input_len) {
    if (input_len > 9) {
        input_len = 9;
    }
    char buffer[10];
    strncpy(buffer, input, input_len);
    buffer[input_len] = '\0';
    printf("Input received: %s\n", buffer);
}

int main() {
    char input_string[20];
    printf("Enter a string: ");
    scanf("%19s", input_string);
    vulnerable_function(input_string, strlen(input_string));
    return 0;
}
```

Para realizar o Patch Diff, usaremos as ferramentas abaixo;

<h5>Ghidra</h5>
Ghidra é uma ferramenta de engenharia reversa gratuita e de código aberta desenvolvida pela Agência de Segurança Nacional dos Estados Unidos. Neste contexto, o Ghidra foi usado para criar exportações binárias para ambos os arquivos para que eles pudessem ser comparados no BinDiff, vale lembrar que o pluguin "BinExport" deve ser adcionado ao Ghidra.

<h5>BinDiff</h5>
BinDiff é uma ferramenta de análise de binários que é amplamente utilizada por pesquisadores de segurança para comparar duas versões de um software e identificar as diferenças entre elas. Ele é desenvolvido pela Zynamics, uma empresa adquirida pela Google em 2011, e é amplamente utilizado em análises de vulnerabilidades de software, detecção de malware e engenharia reversa.

<h5>Binary Ninja</h5>
Binary Ninja é uma plataforma de engenharia reversa desenvolvida pela Vector 35 Inc. Ela pode desmontar um binário e exibir a desmontagem em visualizações lineares ou gráficas. Ele realiza uma análise profunda automatizada do código, gerando informações que ajudam a analisar um binário.

Em resumo, as técnicas de Patch Diff são valiosas ferramentas na análise de vulnerabilidades e no desenvolvimento de exploits de dia zero. Ao comparar as mudanças entre duas versões de um software ou sistema, é possível identificar vulnerabilidades introduzidas, corrigidas ou não corrigidas, permitindo que medidas adequadas de segurança sejam tomadas.

# Quadros

![Desktop View](/img/papers/PatchDiff/excel.png){: width="700" height="202" .w-75 .normal}

## REFERÊNCIA
<https://securityintelligence.com/posts/patch-tuesday-exploit-wednesday-pwning-windows-ancillary-function-driver-winsock/>
<https://www.zynamics.com/software.html>
<https://www.linkedin.com/pulse/cve-2023-24869-remote-procedure-call-runtime-code-execution-hawes/>
<https://ihack4falafel.github.io/Patch-Diffing-with-Ghidra/:

