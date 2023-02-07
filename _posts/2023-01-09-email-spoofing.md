---
title: Encontrando possíveis vulnerabilidades de e-mail spoofing
author: H41stur
date: 2023-01-09 01:00:00 -0300
categories: [Estudos, E-mail Spoofing]
tags: [SMTP, DMARC, DKIM, E-mail Spoofing]
image: "/img/posts/Pasted%20image%2020230106122723.png"
alt: "Pentesting Android"
---

![E-mail Spoofing](/img/posts/Pasted%20image%2020230106122723.png)


- [Introdução](#introdução)
- [*E-mail Spoofing*](#e-mail-spoofing)
	- [Impacto técnico do *e-mail spoofing*](#impacto-técnico-do-e-mail-spoofing)
	- [Impacto de negócio do *e-mail spoofing*](#impacto-de-negócio-do-e-mail-spoofing)
- [SPF](#spf)
	- [Tipos de SPF](#tipos-de-spf)
	- [Atributos do SPF](#atributos-do-spf)
- [DKIM](#dkim)
- [DMARC](#dmarc)
- [Como um servidor de e-mails interpreta todas estas regras](#como-um-servidor-de-e-mails-interpreta-todas-estas-regras)
	- [Ordem de verificação de protocolos](#ordem-de-verificação-de-protocolos)
- [Como são feitos ataques de *e-mail spoofing*](#como-são-feitos-ataques-de-e-mail-spoofing)
- [Encontrando possíveis vulnerabilidades de *e-mail spoofing*](#encontrando-possíveis-vulnerabilidades-de-e-mail-spoofing)
	- [SPF com -all (*hard fail*)](#spf-com--all-hard-fail)
	- [SPF com ~all (*soft fail*)](#spf-com-all-soft-fail)
	- [SPF com ?all (*neutral*)](#spf-com-all-neutral)
	- [Outras configurações que podem levar ao *e-mail spoofing*](#outras-configurações-que-podem-levar-ao-e-mail-spoofing)
	- [Automatizando a enumeração das configurações](#automatizando-a-enumeração-das-configurações)
- [Boas práticas para dificultar o *e-mail spoofing*](#boas-práticas-para-dificultar-o-e-mail-spoofing)
- [Conclusão](#conclusão)
- [Referências](#referências)


## Introdução

Imagine explorar uma vulnerabilidade que, aparentemente, dentro do seu escopo de conhecimento, não seria possível de explorar, e como consequência, não saber muito bem o que foi feito.

Pois bem, assim como toda base de conhecimento parte da ignorância, este estudo surgiu de uma situação parecida, onde, em minha singela falta de conhecimento mais profundo, consegui falsificar um e-mail, mesmo pensando que não seria possível. Então surgiu a dúvida: como eu fiz isso?

E na busca da resposta, muita coisa foi aprendida sobre protocolos e configurações que tentarei deixar da forma mais detalhada no decorrer deste artigo.

Como de costume, gosto de deixar um bom embasamento antes de falar sobre vias de fato, portanto, partimos do princípio, entendendo os tópicos base para o assunto.

## *E-mail Spoofing*

O *e-mail spoofing*, ou falsificação de *e-mail*, é a prática de enviar um *e-mail* com o endereço de remetente falsificado. Isso pode ser feito de várias maneiras, mas a intenção geral é fazer com que o *e-mail* pareça ter sido enviado por alguém diferente do remetente real e com um nível aparente de autenticidade elevado.

Um dos métodos mais comuns de *e-mail spoofing* é a alteração do campo "*From*" de um *e-mail*. Quando você recebe um *e-mail*, o endereço no campo "*From*" é o endereço que o remetente quer que você acredite que enviou o *e-mail*. No entanto, esse endereço pode facilmente ser alterado para que pareça que foi enviado por outra pessoa.

Outra maneira de fazer *e-mail spoofing* é alterando o endereço IP do remetente. O endereço IP é um número único atribuído a cada dispositivo que se conecta à internet. Quando um *e-mail* é enviado, o servidor de *e-mail* do remetente inclui o endereço IP do remetente no cabeçalho do *e-mail*. Se um remetente alterar o seu endereço IP para fazer com que pareça que o *e-mail* foi enviado de outro lugar, isso pode ser considerado *e-mail spoofing*.

O objetivo do *e-mail spoofing* geralmente é induzir o destinatário a acreditar que o *e-mail* é legítimo e realizar alguma ação, como clicar em um link, baixar um anexo ou divulgar informações confidenciais.

Além do impacto financeiro direto, as empresas vítimas de *e-mail spoofing* também podem enfrentar custos indiretos, como tempo e recursos necessários para investigar e remediar o ataque, bem como possíveis consequências legais e regulatórias. 

### Impacto técnico do *e-mail spoofing*

O impacto técnico da falsificação de *e-mail* pode ser significativo, pois permite que os invasores ganhem a confiança do destinatário e possam induzi-lo a divulgar informações confidenciais, como senhas ou informações financeiras. Ele também pode ser usado para espalhar *malware* ou ataques de *phishing* para um público mais amplo, pois os *e-mails* fraudulentos podem ser mais propensos a serem abertos e clicados pelo destinatário devido à percepção de autenticidade do remetente.

Além disso, a falsificação de *e-mail* pode ser usada para contornar os filtros de *spam* e enviar *e-mails* maliciosos diretamente para a caixa de entrada do destinatário, pois os filtros podem não detectar a natureza fraudulenta do *e-mail*. Isso pode permitir que os invasores ignorem as medidas de segurança e obtenham acesso a sistemas ou dados confidenciais.

### Impacto de negócio do *e-mail spoofing*

Esse tipo de ataque pode ter impactos de negócio significativos, incluindo:

1. **Perda de informações confidenciais**: se um funcionário for vítima de um *e-mail* falsificado e clicar em um link ou anexo malicioso, isso poderá resultar na perda de informações confidenciais da empresa, como dados financeiros ou registros de clientes.

2. **Danos à reputação**: Se um *e-mail* falsificado for enviado a clientes ou parceiros, isso poderá prejudicar a reputação da empresa. Os clientes podem perder a confiança na empresa e parar de fazer negócios com ela, enquanto os parceiros podem se recusar a trabalhar com uma empresa que foi vítima de um ataque cibernético.

3. **Perda financeira**: em alguns casos, *e-mails* falsificados podem ser usados ​​para induzir os funcionários a transferir dinheiro para o invasor. Isso pode resultar em perdas financeiras significativas para a empresa.

4. **Consequências legais**: Se uma empresa sofrer uma violação de dados como resultado de um ataque de *e-mail spoofing*, poderá enfrentar consequências legais, incluindo multas e penalidades.

Nesta altura acredito que o ataque de *e-mail spoofing* ficou claro assim como sua criticidade em um cenário corporativo e pessoal.

## SPF

SPF (*Sender Policy Framework*) é um mecanismo de autenticação de *e-mail* usado para ajudar a proteger os destinatários de *e-mails* falsificados ou "*spoofed mails*". Ele permite que os administradores de domínio especifiquem quais servidores de *e-mail* são autorizados a enviar *e-mails* em nome de um domínio específico.

Quando um servidor de *e-mail* recebe um *e-mail*, ele verifica o registro SPF do domínio do remetente para ver se o servidor que enviou o *e-mail* está na lista de servidores autorizados. Se o servidor enviou o *e-mail* não estiver na lista, o *e-mail* pode ser marcado como *spam* ou rejeitado completamente.

O SPF é um mecanismo eficaz para ajudar a proteger os destinatários de *e-mails* falsificados, pois permite que os administradores de domínio especifiquem quais servidores são autorizados a enviar *e-mails* em nome de seu domínio. No entanto, é importante lembrar que o SPF sozinho não é uma solução completa para a prevenção de *e-mail spoofing* e deve ser usado em conjunto com outras medidas de segurança, como o **DMARC** (*Domain-based Message Authentication, Reporting and Conformance*).

### Tipos de SPF

Existem dois tipos principais de SPF: o **SPF básico** e o **SPF estendido**.

O **SPF básico** é o tipo mais simples de SPF e permite que os administradores de domínio especifiquem uma lista de servidores de *e-mail* autorizados a enviar *e-mails* em nome de seu domínio. Essa lista é incluída em um registro TXT no DNS (*Domain Name System*) do domínio.

O **SPF estendido** é uma versão mais avançada do SPF que permite aos administradores de domínio especificar uma lista mais detalhada de servidores de *e-mail* autorizados a enviar *e-mails* em nome de seu domínio. Ele também permite aos administradores especificar ações a serem tomadas quando um *e-mail* é enviado por um servidor não autorizado. As ações podem incluir marcar o *e-mail* como *spam* ou rejeitá-lo completamente.

### Atributos do SPF

Existem vários atributos que podem ser incluídos em um registro SPF para especificar quais servidores são autorizados a enviar *e-mails* em nome de um domínio. Alguns dos principais atributos incluem:

- **ip4**: Especifica um endereço IP IPv4 autorizado a enviar *e-mails* em nome do domínio.
- **ip6**: Especifica um endereço IP IPv6 autorizado a enviar *e-mails* em nome do domínio.
- **a**: Especifica um nome de host autorizado a enviar *e-mails* em nome do domínio.
- **mx**: Especifica um servidor de *e-mail* autorizado a enviar *e-mails* em nome do domínio.
- **include**: Permite que um domínio inclua outro domínio em sua política SPF. Geralmente utilizado se um domínio tiver vários servidores de *e-mail* que enviam *e-mails* em seu nome.
- **all**: O atributo **all** é usado no registro SPF para especificar a ação a ser tomada quando um *e-mail* é enviado por um servidor não autorizado. Existem quatro opções principais para o atributo **all**:

	1. **-all (*hard fail*)**: Marca o e-mail como spam. Isso é usado quando o remetente quer que todos os *e-mails* enviados por servidores de e-mail não autorizados sejam marcados como spam.
	2. **~all (*soft fail*)**: Marca o *e-mail* como *spam* com probabilidade baixa. Isso é usado quando o remetente quer permitir que alguns *e-mails* enviados por servidores de *e-mail* não autorizados sejam entregues, mas quer que a maioria seja marcada como *spam*.
	3. **+all (*pass*)**: Permite o *e-mail*. Isso é usado quando o remetente quer permitir que todos os *e-mails* sejam entregues, independentemente de qual servidor os tenha enviado.
	4. **?all (*neutral*)**: Quando esta opção está configurada, o *e-mail* é tratado como se não existisse um registro SPF. Geralmente utilizado quando um domínio não pode, ou não quer definir se um determinado servidor está ou não autorizado a enviar *e-mails* em nome do domínio.

É importante lembrar que o atributo **all** deve ser usado com cuidado, pois ele determina como os servidores de *e-mail* lidam com os *e-mails* enviados por servidores não autorizados. Se o atributo for configurado incorretamente, isso pode levar a *e-mails* legítimos sendo marcados como *spam* ou rejeitados completamente.

Estes são apenas alguns dos atributos mais comuns que podem ser incluídos em um registro SPF. Existem outros atributos disponíveis também. É importante lembrar que os registros SPF devem ser mantidos atualizados para garantir que apenas os servidores de *e-mail* autorizados estejam incluídos na política.

## DKIM

DKIM (*DomainKeys Identified Mail*) é uma técnica de autenticação de *e-mail* que permite verificar a integridade e a origem de um *e-mail*. Ele foi desenvolvido como uma maneira de combater o *spam* e a fraude por *e-mail*, permitindo que os destinatários verifiquem se um *e-mail* foi realmente enviado pelo remetente que alega ter enviado.

O processo de configuração do DKIM envolve a geração de uma chave privada e pública. A chave privada é usada pelo remetente para assinar os *e-mails* que enviam, enquanto a chave pública é publicada no DNS do domínio do remetente. Quando um *e-mail* é enviado, o remetente adiciona um cabeçalho ao *e-mail* contendo um registro criptografado usando a chave privada.

Quando o *e-mail* é recebido pelo destinatário, o servidor de *e-mail* do destinatário procura a chave pública no DNS do domínio do remetente. Se a chave pública estiver disponível, o servidor de *e-mail* do destinatário usa a chave pública para decifrar o registro criptografado no cabeçalho do *e-mail*. Se o registro decifrado corresponder ao conteúdo do *e-mail*, este é considerado autêntico. Se houver alguma diferença entre o registro decifrado e o conteúdo do *e-mail*, o *e-mail* pode ser considerado alterado ou falsificado.

O objetivo do DKIM é garantir que os *e-mails* não sejam alterados durante a transmissão e que eles realmente venham da organização que diz ter enviado. Isso pode ajudar a proteger contra *spam*, *phishing* e outras formas de fraude por *e-mail*.

## DMARC

DMARC (*Domain-based Message Authentication, Reporting & Conformance*) é um protocolo de segurança de *e-mail* que permite que os remetentes especifiquem qual mecanismo de autenticação de *e-mail* deve ser usado quando os *e-mails* são enviados de um determinado domínio. Ele também permite que os remetentes recebam relatórios sobre os *e-mails* que são enviados em seu nome e que falham na verificação de autenticidade. Isso pode ajudar os remetentes a detectar e prevenir a fraude por *e-mail*, como *spam* e *phishing*.

O DMARC funciona verificando se os mecanismos de autenticação de *e-mail*, como SPF e DKIM, passam na verificação de autenticidade. Se um *e-mail* falhar na verificação de autenticidade, o remetente pode optar por rejeitar o *e-mail* ou colocá-lo na caixa de *spam*.

Para configurar o DMARC, os remetentes precisam adicionar um registro TXT ao DNS do seu domínio que especifique qual mecanismo de autenticação de *e-mail* deve ser usado e como os *e-mails* que falharem na verificação de autenticidade devem ser tratados. Os destinatários podem então verificar o registro DMARC para garantir que o *e-mail* foi enviado por um remetente legítimo.

Abaixo um exemplo de registro TXT no DNS para configurar o DMARC:

```
_dmarc.seudominio.com.br. IN TXT "v=DMARC1; p=reject; pct=100; rua=mailto:seuemail@seudominio.com.br"
```

O registro DMARC é composto por vários campos que especificam como o DMARC deve ser usado. Alguns dos campos mais comuns e suas respectivas finalidades são:

- **v=DMARC1 (*version*)**: Especifica qual versão do DMARC está sendo usada. Atualmente, a única versão disponível é DMARC1.    
- **p= (*policy*)**: Especifica como os *e-mails* que falharem na verificação de autenticidade devem ser tratados. As opções são "***none***" (não fazer nada), "***quarantine***" (colocar na caixa de *spam*) e "***reject***" (rejeitar o *e-mail*).    
- **pct= (*percentage*)**: Especifica a porcentagem de *e-mails* que devem ser verificados pelo DMARC. O valor padrão é 100, o que significa que todos os *e-mails* serão verificados.    
- **sp= (*subdomain policy*)**: Especifica qual mecanismo de autenticação de *e-mail* deve ser usado para os *e-mails* enviados de subdomínios. As opções são "***none***" (não usar autenticação de *e-mail*), "***quarantine***" (colocar na caixa de *spam*) e "***reject***" (rejeitar o *e-mail*).    
- **adkim= (*DKIM alignment*)**: Especifica qual modo de alinhamento deve ser usado para o DKIM. As opções são "**r**" (rigoroso) e "**s**" (relaxado). O alinhamento rigoroso exige que o remetente e o destinatário usem o mesmo domínio, enquanto o alinhamento relaxado permite que o remetente e o destinatário usem domínios diferentes.    
- **aspf= (*SPF alignment*)**: Especifica qual modo de alinhamento deve ser usado para o SPF. As opções são "**r**" (rigoroso) e "**s**" (relaxado). O alinhamento rigoroso exige que o remetente e o destinatário usem o mesmo domínio, enquanto o alinhamento relaxado permite que o remetente e o destinatário usem domínios diferentes.    
- **rua= (*aggregated reports*)**: Especifica o endereço de *e-mail* onde os relatórios DMARC devem ser enviados.    
- **ruf= (*forensic reports*)**: Especifica o endereço de *e-mail* onde os relatórios DMARC de falha de autenticidade devem ser enviados.

## Como um servidor de e-mails interpreta todas estas regras

De fato, pela criticidade que o *e-mail spoofing* pode atingir, foram criadas vários protocolos para se certificar da autenticidade de *e-mails* recebidos e métodos para diminuir este tipo de ação. Porém, dada a delicadeza do processo de envio e recebimento de *e-mails*, que pode ter diversas regras de negócio, várias opções foram criadas para cada protocolo, que podem e são interpretadas tanto pelo servidor de recebimento, quanto de envio de *e-mails*.

As configurações de cada protocolo são estritamente feitas, com base na regra de negócio. Enquanto uma configuração extremamente restrita pode ocasionar problemas de recebimento de *e-mails* autênticos, uma configuração relaxada pode facilitar a falsificação de *e-mails*, e por sua vez, uma configuração moderada pode evitar problemas de envio/recebimento assim como também abrir caminho para a falsificação de *e-mails*.

### Ordem de verificação de protocolos

Não há uma ordem específica em que os servidores de *e-mail* devem verificar o DMARC ou o SPF. Alguns servidores podem verificar o SPF primeiro, enquanto outros podem verificar o DMARC primeiro. Alguns servidores também podem usar outras técnicas de autenticação, como a verificação de chave DKIM, antes de verificar o SPF ou o DMARC.

A ordem em que os servidores de *e-mail* verificam os diferentes métodos de autenticação de *e-mail* pode variar dependendo de vários fatores, como as configurações do servidor, as políticas de segurança do *e-mail* da empresa e as práticas recomendadas pelos fornecedores de serviços de *e-mail*.

## Como são feitos ataques de *e-mail spoofing*

Conforme visto anteriormente, um ataque de *e-mail spoofing* pode ser feito ao modificar informações de um cabeçalho de *e-mail*. Os atacantes podem modificar o cabeçalho do *e-mail* para fazer com que pareça que o *e-mail* foi enviado por outra pessoa ou empresa. 

Isso pode ser feito diretamente no servidor de envio de *e-mails*, ou **SMTP** (*Simple Mail Transfer Protocol*). Existem plataforma conhecidas e específicas que "automatizam" estas alterações no cabeçalho permitindo a exploração da vulnerabilidade, como o [Emkey's Mailer](https://emkei.cz/) e o [Anonymailer](https://anonymailer.net/).

Este ataque também pode surgir de um servidor SMTP "mal configurado" que permite a alteração de informações de cabeçalho ao enviar *e-mails*.

## Encontrando possíveis vulnerabilidades de *e-mail spoofing*

Uma vez que sabemos da existência de diversas combinações de configuração, principalmente entre SPF e DMARC, e de que a checagem ocorre nas duas pontas, destinatário e remetente, a possibilidade de uma exploração de *e-mail spoofing* bem-sucedida se torna uma questão de "ligar os pontos".

Mas é importante ressaltar que, como esta análise é feita exclusivamente nas configurações de protocolos, mesmo que os registros sejam aparentemente vulneráveis a *spoofing*, ainda há grandes possibilidades de serviços de terceiros mitigarem a exploração, como filtros e *firewalls* de e-mail. Porém, não impede que uma tentativa seja feita em uma situação de um *pentest* ou qualquer outro teste **dentro de parâmetros legais**.

Partindo deste princípio, e de experiência no mundo real, é possível enumerar possíveis vetores de ataque em potencial. Sabendo que o SPF tem quantidade reduzida de parâmetros em comparação ao DMARC, a seguir serão expressas possibilidades com base nas diferentes variações da propriedade **all**.

### SPF com -all (*hard fail*)

Durante muito tempo, eu mesmo acreditei que se o atributo **-all** do SPF era suficiente para bloquear um ataque de *e-mail spoofing*, porém com a presença (ou não) de algumas configurações de DMARC, ainda é possível realizar um ataque.

A tabela abaixo, mostra combinações de atributos DMARC e possíveis consequências:

| **Configurações DMARC**                                                    | **Possível consequência**                                                                        |
| ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| **p=** e **aspf=** configurados e **sp=none**                          | Possível *spoofing* de subdomínio                                                            |
| ausência de **aspf** e **sp=none**                                     | Possível *spoofing* de subdomínio                                                            |
| **p=none** e **aspf=r** ou ausência de **aspf=**                       | *Spoofing* possível a depender da política do destinatário                                   |
| **p=none**, **aspf=r** e **sp=reject** ou **sp=quarentine**            | Possível *spoofing* dentro do domínio (destinatário e remetente dentro da mesma organização) |
| **p=none**, ausência de **aspf=** e **sp=reject** ou **sp=quarentine** | Possível *spoofing* dentro do domínio (destinatário e remetente dentro da mesma organização) |
| **p=none**, **sp=none** e ausência de **aspf=**                        | Possível *spoofing* de subdomínio                                                            |                                                                       |                                                                                              |

### SPF com ~all (*soft fail*)

O atributo **~all** por sí só já é mais flexível, permitindo possibilidades de *spoofing* mesmo com regras mais rígidas no DMARC. 

A tabela abaixo, mostra combinações de atributos DMARC e possíveis consequências:

| **Configurações DMARC**                                               | **Possível consequência**                                                                                                        |
| --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **p=none** **sp=reject** ou **sp=quarentine**                         | Possível *spoofing* dentro do domínio (destinatário e remetente dentro da mesma organização)                                     |
| **p=none** e ausência de **sp=**                                      | Possível *spoofing*                                                                                                              |
| **p=none** e **sp=none**                                              | Possível *spoofing* de subdomínio e possível _spoofing_ dentro do domínio (destinatário e remetente dentro da mesma organização) |
| **p=reject** ou **p=quarentine**, ausência de **aspf=** e **sp=none** | Possível *spoofing* de subdomínio                                                                                                |
| **p=reject** ou **p=quarentine**, **aspf=** configurado e **sp=none** | Possível *spoofing* de subdomínio                                                                                                |

### SPF com ?all (*neutral*)

O atributo **~all** faz com que o registro SPF de fato exista (o DMARC exige que o SPF ou DKIM existam para que seja configurado), mas não impõe nenhuma regra, como se não estivesse presente.

Esta configuração, faz com que a autenticação seja feita somente pelo DMARC ou DKIM, quando presente.

A tabela abaixo, mostra combinações de atributos DMARC e possíveis consequências:

| **Configurações DMARC**                                                | **Possível consequência**                                                                    |
| ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| **p=reject** ou **p=quarentine**, **aspf=** configurado e **sp=none**  | Possível *spoofing* de subdomínio a depender da política do destinatário                     |
| **p=reject** ou **p=quarentine**, ausência de **aspf=** e **sp=none**  | Possível *spoofing* de subdomínio a depender da política do destinatário                     |
| **p=none**, **aspf=r** e ausência de **sp=**                           | Possível *spoofing*                                                                          |
| **p=none**, **aspf=r** e **sp=none**                                   | Possível *spoofing* de subdomínio                                                            |
| **p=none**, **aspf=s** ou ausência de **aspf=** e **sp=none**          | Possível *spoofing* de subdomínio                                                            |
| **p=none**, **aspf=s** ou ausência de **aspf=** e ausência de **sp=**  | Possível *spoofing* de subdomínio a depender da política do destinatário                     |
| **p=none**, **aspf=** configurado e **sp=reject** ou **sp=quarentine** | Possível *spoofing* dentro do domínio (destinatário e remetente dentro da mesma organização) |
| **p=none**, auseência de **aspf=** e **sp=reject**                     | Possível *spoofing* dentro do domínio (destinatário e remetente dentro da mesma organização)                                                                                             |

### Outras configurações que podem levar ao *e-mail spoofing*

Além destas combinações de configurações possíveis entre SPF e DMARC, existem outros fatores que podem possibilitar a falsificação de *e-mails*. Abaixo um descritivo:

- **pct=** com qualquer valor menor que 100: Significa que não serão todos os *e-mails* que passarão pela validação de autenticidade, logo pode levar ao *spoofing*.
- Registro SPF com vários domínios no ***include***: A existência de vários domínios permitidos no envio de *e-mails* pode levar a divergências de configuração no registro TXT levando ao *spoofing.*
- Mais de um atributo **all** no mesmo registro TXT sem o **p=** no DMARC.
- Ausência de registro SPF e ausência do **p=** no DMARC.
- Qualquer opção de atributo **all** sem o **p=** no DMARC.

### Automatizando a enumeração das configurações

Todas as informações sobre as configurações e atributos são públicas e de fácil acesso/leitura (menos a chave privada do DKIM) através de consultas nos registros TXT do DNS e DMARC. Estas informações precisam desta exposição justamente para que todas as pontas de uma comunicação via *e-mail* possam consultá-las.

É possível consultá-las em serviços online como o [MX Toolbox](https://mxtoolbox.com/) assim como ferramentas normalmente encontradas em sistema operacional Linux (mas não limitado somente a eles), como `host` e `dig`, e outras ferramentas criadas especificamente com esta finalidade.

A checagem destas combinações também foi implementada em uma das ferramentas desenvolvida por mim. [Nina](https://github.com/h41stur/nina) possui um módulo que usa basicamente todo o conteúdo deste artigo para cruzar informações e analisá-las.

Abaixo um demonstrativo:

![Output da ferramenta](/img/posts/Pasted%20image%2020230106212409.png)

## Boas práticas para dificultar o *e-mail spoofing*

Conforme visto, quando se trata de uma análise puramente técnica, a melhor prática para evitar falsificações de *e-mail*, são as regras de configuração mais restritas possíveis, porém, no mundo real existem as **regras de negócio**, que são o verdadeiro motivo por trás de quase tudo, e ditam como o comportamento deve ser direcionado.

Portanto, existem boas práticas que devem ser avaliadas para garantir o máximo de proteção possível, sem divergirem com as regras de negócio. Algumas delas são:

1. Configurar o DMARC para o domínio: O primeiro passo é adicionar o registro DMARC ao arquivo DNS do domínio. Isso permitirá que os servidores de *e-mail* verifiquem as políticas de autenticação para *e-mails* enviados em nome do domínio.
    
2. Definir as políticas de autenticação de *e-mail*: No registro DMARC, definir as políticas de autenticação de *e-mail* que sejam aplicadas aos e*-mails* enviados em nome do domínio. Isso pode incluir exigir que os *e-mails* sejam assinados com chave DKIM e verificados pelo SPF.
    
3. Ativar os relatórios de usagem: Ativar os relatórios de usagem (RUA e RUF) para receber informações sobre os *e-mails* enviados em nome do domínio. Isso permitirá o monitoramento do uso de *e-mail* do domínio e a detecção de possíveis ataques de *spoofing*.
    
4. Configurar as ações a serem tomadas em caso de falha: No registro DMARC, definir as ações a serem tomadas em caso de falha na verificação de autenticidade de *e-mail*. Isso pode incluir marcar os *e-mails* como *spam* ou rejeitar os *e-mails* que não cumprem as diretivas.

5. Ativar a verificação de chave DKIM: A verificação de chave DKIM é outra ferramenta de segurança de *e-mail* que ajuda a proteger contra o *spoofing*. 

Além de configurações, é preciso medidas de fortalecimento comportamental por parte de usuários, algumas dessas medidas incluem verificar a veracidade de todos os e-mails recebidos, não clicar em links ou baixar arquivos de e-mails desconhecidos, usar ferramentas de segurança de e-mail e manter o software de segurança atualizado.

## Conclusão

Este estudo surgiu de uma exploração bem-sucedida em um servidor no qual eu acreditava não ser possível falsificar um *e-mail*. Na busca pela resposta de como isto aconteceu, o conteúdo resumido neste artigo foi adquirido de diversas fontes e documentações.

O *e-mail spoofing* é uma ameaça constante na internet que pode levar a um comprometimento crítiuco da imagem de uma corporação, e, muitas vezes, difícil de detectar e/ou rastrear. No entanto, tomando as medidas de segurança adequadas, tanto os usuários quanto os administradores de domínio podem ajudar a proteger-se contra esses tipos de ataques e manter sua informação pessoal segura.


## Referências

- [RFC-7208](https://www.rfc-editor.org/rfc/rfc7208)
- [How Microsoft 365 uses Sender Policy Framework (SPF) to prevent spoofing](https://learn.microsoft.com/en-us/microsoft-365/security/office-365-security/email-authentication-anti-spoofing?view=o365-worldwide)
- [Real or Fake? Spoof-Proofing Email With SPF, DKIM, and DMARC](https://www.trustedsec.com/blog/real-or-fake-spoof-proofing-email-with-spf-dkim-and-dmarc/)
