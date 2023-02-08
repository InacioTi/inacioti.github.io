---
title: Threat hunting
author: José Inácio
date: 2022-04-26
categories: [Papers, Threat hunting]
tags: [Threat hunting, Windows]
pin: true
---


Sabemos que o monitoramento desempenha um papel crucial na detecção e análise atividades maliciosas em um sistema. Com isso, resolvi criar esse laboratório para estudo pessoal com o intuito de monitorar e detectar logs de eventos, relacionados a atividades maliciosas usando o ELK Stack, usando o winlogbeat como ferramenta de envio de logs e configurando Sysmon para ajudar a identificar atividades de anomalias.

Resolvi criar algo simplista, sem muito recurso, com o objetivo único de obter um maior entendimento do monitoramento e gerenciamento de eventos. Então, sem muita enrolação, ideia para este laboratório foi a seguinte;

1. Criar uma VM Windows, para ser o “alvo”.

2. Utilizar o ELK Stack para registrar os logs desse “alvo”.

3. Criar um arquivo excel com macro contendo um execução de powershell simples

4. Criar um “ataque” de e-mail phising para envio desse arquivo

5. Neste “alvo” fazer o acesso ao e-mail e o download de execução da planilha “Maliciosa”

Verificar os logs obtidos do “alvo”.

## ELK Stack

Resumidamente o ELK Stack é um conjunto de ferramentas gratuitas e abertas para ingestão, enriquecimento, armazenamento, análise e visualização de dados. É chamada de ELK pelas iniciais de Elasticsearch, Logstash e Kibana.

1. Elasticsearch: É um mecanismo de busca tranquilo que armazena ou detém todos os Dados coletados.
2. Logstash: É o componente de processamento de dados que envia dados de entrada para elasticesearch.
3. Kibana: Uma interface web para pesquisar e visualizar logs.

Essa arquitetura geralmente tem algum tipo de log/evento encaminhado usando algum Beat(são agentes instalados nos hosts para coletar diferentes tipos de informações a serem direcionadas para dentro da stack) rodando em cada host monitorado, que envia os dados para o Logstash, que os analisa, enriquece e os envia para um banco de dados não relacional (neste caso Elastic) que pode ser consultada e visualizada através de dashboards (Kibana).

Exemplo do esquema abaixo usando o FileBeat

![Desktop View](/img/papers/elk/filebeat.png){: width="700" height="202" .w-75 .normal}

No entanto, este cenário demanda que seja 100% manual, é preciso consultar e analisar manualmente os dados para detectar eventos potencialmente maliciosos.

Neste caso, utilizaremos uma o Elastic-SIEM em Docker, que nos possibilita usar as funcionalidades do Elastic Security.

O laboratório que montamos até agora é uma versão simplificada da plataforma HELK, criada por Roberto Rodrigues. Para quem não conhece, dê uma olhada, é incrível. Possui recursos de processamento distribuído e aprendizado de máquina, Apache Spark, Hadoop, GraphFrames e Jupiter Notebooks.

<h2 data-toc-skip>Preparando o “Alvo”</h2>

Gostaria de deixar claro que o intuito não será abordar o passo a passo de Instalação/configuração de cada item usado, então deixarei somente os links que por si só já descrevem o seu uso.

Já com Elastic Siem rodando em Docker, partimos para o “alvo” que neste caso será uma VM com Windows 11, e nela usaremos o winlogbeat e o sysmon, para que os logs sejam coletados e enviados, para o nosso Elastic Siem.

Sysmon é um software que faz parte da suite Sysinternals da Microsoft. Ele em si é um serviço do Windows que, após ativado, monitora as diversas atividades que não são logadas por padrão, no sistema de eventos do Windows.

![Desktop View](/img/papers/elk/sysmon.png){: width="700" height="202" .w-75 .normal}-

Winlogbeat: Faz parte do Beats mencionado acima, é um agente cujo objetivo é coletar dados de arquivos de logs do Windows

Neste caso esta configurado abaixo para coletar logs do Sysmon.

![Desktop View](/img/papers/elk/winlogbeat.png){: width="700" height="202" .w-75 .normal}

<h2 data-toc-skip>Habilitando Logs do Powershell</h2>

Para que fosse possível ter todo controle da execução do powershell precisamos habilita-lo, assim podemos ter em registro os detalhes da execução conforme o PowerShell é executado, incluindo inicialização de variável e invocações de comando. O log do módulo registrará partes de scripts, alguns códigos desofuscados e alguns dados formatados para saída.

Esse log capturará alguns detalhes perdidos por outras fontes de log do PowerShell, embora possa não capturar de forma confiável os comandos executados.

Para habilitar o log do módulo:

<Nas configurações do GPO “Windows PowerShell”, defina “Ativar Log do Módulo” como habilitado.

2. No painel “Opções”, clique no botão para mostrar o Nome do Módulo.

3. Na janela Module Names, digite * para registrar todos os módulos.

4. Clique em “OK” na janela “Nomes dos módulos”.

5. Clique em “OK” na janela “Module Logging”.

![Desktop View](/img/papers/elk/polygroup.png){: width="700" height="202" .w-75 .normal}

<h2 data-toc-skip>Criando o arquivo Malicioso</h2>

Criamos um arquivo Excel com um código simples de VBA, que basicamente gerar um alerta com o usuário “logado” na maquina e executa um Powershell que usa a classe Win32_Desktop WMI para representa as características comuns da área de trabalho de um usuário.

```bash
Private Sub Workbook_Open()
 MsgBox UsuarioRede
End Sub Function UsuarioRede() As String
 Dim GetUserN
 Dim Objnetwork
 Dim pscmd
 Set ObjNetwork = createObject(“WScript.Network”)
 GetUserN = ObjNetwork.UserName
 UsuarioRede = GetUserN
 pscmd = “pscmd = “PowerShell -Command “”{Get-CimInstance -ClassName Win32_Desktop}””””
 Shell pscmdEnd Function
```

<h2 data-toc-skip>Enviando o arquivo Malicioso</h2>

Para fazer o envido do documento simulando um e-mail de phising, usarei a ferramenta SET (Social Engineering Toolkit, Sim, a mesma usada pelo Mr. Robot rsrs). O SET é uma ferramenta de código aberto orientada a Python, destinada a testes de penetração em torno da Engenharia Social.

Criei uma conta fictícia para o envio do e-mail contendo o arquivo.

![Desktop View](/img/papers/elk/set.png){: width="700" height="202" .w-75 .normal}

<h2 data-toc-skip>Baixando o arquivo e executando</h2>

Como neste laboratório a ideia e ter uma visão básica, e usa-lo para implementar novas formas, o “ataque” não foi algo sofisticado e por isso não teve necessidade de se tentar ofuscar o código para que não fosse detectado por sistema de AV.

Então para que isso funcionasse da forma simples, desabilitei o firewall e defender do Windows, e desabilitei também o “trusted” da macro no Excel.

![Desktop View](/img/papers/elk/exceltrusted.png){: width="700" height="202" .w-75 .normal}

![Desktop View](/img/papers/elk/excelmacro.png){: width="700" height="202" .w-75 .normal}

Feito isso, partimos para o download do arquivo malicioso, e posteriormente o executamos

![Desktop View](/img/papers/elk/email.png){: width="700" height="202" .w-75 .normal}

![Desktop View](/img/papers/elk/excel.png){: width="700" height="202" .w-75 .normal}

<h2 data-toc-skip>Verificando Eventos</h2>

Olhando os eventos gerados após a execução do arquivo, podemos ver nos logs do Symon alguns informações interessantes;

ParentImage: Caminho do arquivo que gerou/criou o processo principal

ParentCommandLine : Argumentos que foram passados para o executável associado ao processo pai

OriginalFileName : Nome do arquivo original do cabeçalho PE, adicionado na compilação

CommandLine : Argumentos que foram passados para o executável associado ao processo principal

É possível ver que o ParentImage mostra o caminho do office16\execel.exe, e que o ParentCommandLine diz que o Excel quem executou o arquivo documentos.xlsm(Arquivo malicioso). Mas pra cima e possível ver que o OriginalFileName é o PowerShell.exe, e que o CommandLine é o comando que “Malicioso” que foi inserido na macro do arquivo.

![Desktop View](/img/papers/elk/eventViwer.png){: width="700" height="202" .w-75 .normal}

<h2 data-toc-skip>Fazendo Pesquisa no Elastic ELK</h2>

No Discover, selecionei o índice para winloagbeat, isso carregará os dados indexados com o índice selecionado.

![Desktop View](/img/papers/elk/elkDiscorver.png){: width="700" height="202" .w-75 .normal}

![Desktop View](/img/papers/elk/elkWinlogbeat.png){: width="700" height="202" .w-75 .normal}

Existem 2 maneiras de usar a opção de pesquisa.

Kibana Query Language (KQL): A Kibana Query Language (KQL) é uma sintaxe simples para filtrar dados do Elasticsearch usando pesquisa de texto livre ou pesquisa baseada em campo.

Lucene query syntax: a sintaxe de consulta do Lucene está disponível para usuários do Kibana que optam por não usar a linguagem de consulta do Kibana.

Faremos a consulta como KQL e buscaremos pelo seguinte;

```bash
winlog.channel:”Microsoft-Windows-Sysmon/Operational” and winlog.event_data.Image:*powershell* and winlog.event_data.ParentCommandLine:*Office*
```

![Desktop View](/img/papers/elk/elkLogs.png){: width="700" height="202" .w-75 .normal}


<h2 data-toc-skip>Detections</h2>

Agora queremos ter um regra de detecção baseado neste evento de origem suspeita, e criar alertas para quando as condições dessa regra forem atendidas.

In Security → Detections → manage detection rules, crie uma nova

![Desktop View](/img/papers/elk/elkDetect.png){: width="700" height="202" .w-75 .normal}


Podemos verificar regras pesquisam índices periodicamente (como endgame-* e filebeat-*) neste caso, queremos pesquisar apenas no índice winlogbeat.

![Desktop View](/img/papers/elk/elkIndice.png){: width="700" height="202" .w-75 .normal}

Quando um alerta é criado, seu status é Aberto. Para ajudar a rastrear investigações, o status de um alerta pode ser definido como Aberto, Reconhecido ou Fechado.

Quando executamos o arquivo excel no “alvo”, ele gerou o alerta.

![Desktop View](/img/papers/elk/alert.png){: width="700" height="202" .w-75 .normal}

<h2 data-toc-skip>Vendo descrição</h2>

Podemos ver a descrição deste alerta

![Desktop View](/img/papers/elk/alertDetail.png){: width="700" height="202" .w-75 .normal}

<h2 data-toc-skip>Criando Caso</h2>

Os casos são usados ​​para abrir e rastrear problemas de segurança diretamente no aplicativo Elastic Security. Todos os casos listam o relator original e todos os usuários que contribuem para um caso (participantes). Os comentários suportam a sintaxe Markdown e permitem vincular a Timelines salvas.

![Desktop View](/img/papers/elk/CreateCase.png){: width="700" height="202" .w-75 .normal}

![Desktop View](/img/papers/elk/CreateCaseDetails.png){: width="700" height="202" .w-75 .normal}

Neste pequeno laboratório de estudo de caso, o objetivo foi estudar a aplicabilidade do ELK Stack em um cenário, de forma que os logs de eventos de segurança importantes sejam tratados. Gerando alertas que podem ser gerenciados, mas a ideia e continuar esse laboratório e implementar pouco a pouco novas Features, baseando no projeto HELK.

## REFERENCIA

<https://imsafe.com.br/phishing-pratico-com-hiddeneye/>

<https://enigma0x3.net/2016/03/15/phishing-with-empire/>

<https://infosecwriteups.com/sending-emails-using-social-engineering-toolkit-setoolkit-97427712c809>

<https://infosecwriteups.com/sending-emails-using-social-engineering-toolkit-setoolkit-97427712c809>

<https://www.hackingarticles.in/threat-hunting-log-monitoring-lab-setup-with-elk/>

<https://www.digitalocean.com/community/tutorials/how-to-create-rules-timelines-and-cases-from-suricata-events-using-kibana-s-siem-apps>

<https://cyberwardog.blogspot.com/2017/02/setting-up-pentesting-i-mean-threat_87.html>

<https://rootdse.org/posts/understanding-sysmon-events/>
