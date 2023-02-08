---
title: Hardening Windows Server
author: José Inácio
date: 2022-04-12
categories: [Papers, Tutorial]
tags: [Windows,Hardening]
math: true
mermaid: true
---


Neste meu primeiro artigo, abordarei um termo revisado recentemente de um curso ao qual estou, Server Hardening.

Sendo bem simplista, Hardening seria um conjunto de processos de reforço de segurança aplicadas ao sistema. A realização de hardening em sistemas não é apenas uma boa prática, em algumas áreas pode ser uma exigência regulamentar, o objetivo é minimizar os riscos à segurança.

A ISO/IEC 27002 é uma das normas componentes da série ISO/IEC 27000 que mais abrangem aos processos de hardening. Apresentando um conjunto de boas práticas, para justamente estabelecer controles de segurança da informação.

Os Benchmarks do CIS são um recurso abrangente de documentos que abrangem muitos sistemas operacionais e aplicativos.

Para sistemas Windows, a Microsoft publica linhas de base e ferramentas de segurança para verificar a conformidade dos sistemas em relação a eles.

OpenSCAP para sistemas Linux. que fornece ferramentas de código aberto para identificar e corrigir problemas de segurança.

Neste pequeno artigo iremos focar no Hardening do Windows Server, que envolve identificar e remediar vulnerabilidades de segurança.

Aqui estão algumas práticas de Hardening do Windows Server. Lembre-se é somente um ponto de partida, deve-se revisar e alterar de acordo com as necessidades da organização.

<h2 data-toc-skip>Segurança Organizacional</h2>

- Mantenha um registro de inventário para cada servidor.
- Teste minuciosamente e valide cada alteração proposta para hardware ou software do servidor antes de fazer a mudança no ambiente de produção.
- Realizar regularmente uma avaliação de risco.


<h2 data-toc-skip>Preparação do servidor do Windows</h2>


- Defina uma senha de BIOS/firmware para evitar alterações não autorizadas nas configurações de inicialização do servidor.
- Desabilitar a logon administrativa automática no console de recuperação.
- Configure a ordem de inicialização do dispositivo para evitar a inicialização não autorizada de mídia alternativa.

<h2 data-toc-skip>Instalação do servidor Windows</h2>

- Certifique-se de que todos os patches, hotfixes e pacotes de serviço apropriados sejam aplicados prontamente.
- Depois de instalar o Windows Server, atualize-o imediatamente com os patches mais recentes via WSUS ou SCCM. (Os patches de segurança resolvem vulnerabilidades conhecidas que os atacantes poderiam explorar para comprometer um sistema.)
- Habilite a notificação automática da disponibilidade do patch. Sempre que um patch é liberado, ele deve ser analisado, testado e aplicado em tempo hábil usando WSUS ou SCCM.

<h2 data-toc-skip>Segurança da conta de usuário</h2>

- Certifique-se de que suas senhas administrativas e do sistema atendam às melhores práticas de senha.
- Certifique-se de que todas as senhas sejam alteradas a cada 90 dias.
- Proibir os usuários de criar e fazer login com contas Microsoft.
- Desabilitar a conta do convidado.
- Não permita que permissões “todos” se apliquem a usuários anônimos.
- Não permita a enumeração anônima de contas e ações SAM.
- Desativar a tradução anônima SID/Name.
- Desabilitar ou excluir prontamente contas de usuário não usadas.

<h2 data-toc-skip>Configuração de segurança de rede</h2>

- Habilite o firewall do Windows em todos os perfis (domínio, privado, público) e configure-o para bloquear o tráfego de entrada por padrão.
- Execute o bloqueio da porta no nível de configuração da rede.
- Restringir a capacidade de acessar cada computador da rede apenas para usuários autenticados.
- Se o RDP for utilizado, defina o nível de criptografia de conexão RDP para alto.
- Remova a ativação do LMhosts lookup.
- Desabilitar o NetBIOS no TCP/IP.
- Remova ncacn_ip_tcp.
- Configure tanto o Microsoft Network Client quanto o Microsoft Network Server para sempre assinar comunicações digitalmente.
- Desativar o envio de senhas não criptografadas para servidores SMB de terceiros.
- Permita que o Sistema Local use a identidade do computador para NTLM.
- Configure tipos de criptografia permitidos para Kerberos.
- Não armazene os valores de hash do LAN Manager.
- Defina o nível de autenticação do LAN Manager para permitir apenas NTLMv2 e recusar LM e NTLM.
- Remova o compartilhamento de arquivos e impressões das configurações da rede. O compartilhamento de arquivos e impressões pode permitir que qualquer pessoa se conecte a um servidor e acesse dados críticos sem exigir um ID ou senha do usuário.

<h2 data-toc-skip>Configuração de segurança do registro</h2>

Certifique-se de que todos os administradores aproveitem o tempo para entender completamente como funciona o registro e o propósito de cada uma de suas várias chaves. Muitas das vulnerabilidades no sistema operacional Windows podem ser corrigidas alterando chaves específicas, conforme exemplos abaixo.

- Defina o AutoShareServer para 0.
- Defina AutoShareWks para 0.

Isso desabilitaria os compartilhamentos administrativos (ou seja: o compartilhamento c$ oculto).

- Exclua todos os dados de valor DENTRO da tecla NullSessionPipes.
- Exclua todos os dados de valor DENTRO da tecla NullSessionShares.

Isso remove o acesso de NullSessions. Entre outras coisas, remover o acesso impediria um usuário remoto de enumerar contas de usuário e compartilhamentos em seu sistema.

<h2 data-toc-skip>Configurações gerais de segurança</h2>

- Remover todos os serviços desnecessários do sistema.
- Habilite o EFS (Encrypting File System, sistema de arquivos de criptografia incorporado) com NTFS ou BitLocker no Windows Server.
- Não use AUTORUN. Caso contrário, o código não confiável pode ser executado sem o conhecimento direto do usuário; por exemplo, os atacantes podem colocar um CD na máquina e fazer com que seu próprio script seja executado.
- Exibir um aviso legal antes do login do usuário exemplo: “O uso não autorizado deste computador e recursos de rede é proibido…”
- Exija ctrl+Alt+Del para logins interativos.
- Configure um limite de inatividade da máquina para proteger sessões ociosas.
- Certifique-se de que todos os volumes estejam usando o sistema de arquivos NTFS.
- Configure permissões locais de arquivo/pasta. Outro procedimento de segurança importante, mas muitas vezes negligenciado, é bloquear as permissões de nível de arquivo para o servidor. Por padrão, o Windows não aplica restrições específicas em nenhum arquivo ou pasta local; o grupo Todos recebe permissões completas para a maioria da máquina. Remova esse grupo e, em vez disso, conceda acesso a arquivos e pastas usando grupos baseados em papel com base no princípio de menor privilégio. Todas as tentativas devem ser feitas para remover o Guest, Everyone e ANONYMOUS LOGON das listas de direitos do usuário.
- Defina a data/hora do sistema e configure-a para sincronizar com servidores NTP.

<h2 data-toc-skip>Configurações de políticas de auditoria</h2>

- Habilitar a política de auditoria de acordo com as melhores práticas da política de auditoria.
- Configure o envio de log para algum SIEM para monitoramento.
- Guia de segurança de software
- Instale e habilite o software antivírus. Configure-o para atualizar diariamente.
- Instale e habilite o software anti-spyware. Configure-o para atualizar diariamente.
- Instale software para verificar a integridade de arquivos críticos do sistema operacional. O Windows tem um recurso chamado Windows Resource Protection que verifica automaticamente certos arquivos-chave e os substitui se eles forem corrompidos.

## REFERÊNCIA

<https://www.linkedin.com/pulse/hardening-de-servidores-leandro-cabral/?originalSubdomain=pt>

<https://blog.convisoappsec.com/hardening-de-sistemas-o-que-e-e-como-executar/>

<https://www.osnn.net/threads/help-with-security-settings.42359/>

<http://www.security-forums.com/viewtopic.php?printertopic=1&t=31900&start=0&postdays=0&postorder=asc&vote=viewresult&sid=459ea285cede97e536ad6947328e47>
