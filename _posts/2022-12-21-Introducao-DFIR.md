---
title: Introducao ao DFIR
author: José Inácio
date: 2022-12-21
categories: [Papers, Introducao ao DFIR]
tags: [DFIR]
pin: true
---


Com base nos últimos estudos que venho realizando sobre o assunto, me deparei com várias fontes de informações sobre o tema. Para fixação de aprendizado, resolvi tomar nota do que pude absorver durante esse processo.

## DFIR?

DFIR (Digital Forensics Incident Response) é uma das áreas de segurança cibernética. Uma equipe de DFIR é responsável por gerenciar a resposta a incidentes de segurança, incluindo a coleta de evidências de incidentes, a correção de seu impacto e a implementação de controles para evitar que ocorram novamente no futuro.


Podemos dividir o DFIR em dois componentes principais:

- Forense Digital: Examina os dados do sistema, a atividade do usuário e outras evidências digitais para determinar se um ataque está em andamento e quem está por trás da atividade.
- Resposta a incidentes: Oprocesso geral que as organizações seguirão para preparar, detectar, conter e recuperar de violações de dados.

Devido ao aumento contínuo de ataques a endpoints, o DFIR tornou-se uma competência central na estratégia de segurança de uma organização e nos recursos de caça a ameaças. A mudança para a nuvem e a adoção do trabalho remoto aumentaram a necessidade das organizações garantirem que todos os dispositivos conectados à rede estejam protegidos contra uma variedade de ameaças.

Embora o DFIR se encaixe como um recurso de segurança reativo, o uso de ferramentas sofisticadas, como inteligência artificial (IA) e/ou aprendizado de máquina (ML) permitem que algumas organizações aproveitem as atividades do DFIR para medidas preventivas. Nesse caso, o DFIR também pode ser considerado um componente de uma política de segurança proativa.



<h2 data-toc-skip>Como usar a perícia digital no planejamento de resposta a incidentes ?</h2>

A perícia digital fornece as informações e evidências necessárias exigidas por um Centro de Estudos para Resposta e Tratamento de Incidentes em Computadores (CERT) ou Grupo de Resposta a Incidentes de Segurança (CSIRT) para responder a um incidente de segurança.

A perícia digital pode incluir:

- Análise forense do sistema de arquivos
- Análise forense de memória
- Análise forense de rede
- Análise de log

Além de ajudar as equipes a responder a ataques, a perícia digital desempenha um papel importante em todo o processo de correção, podendo auxiliar a desenvolver e fortalecer medidas preventivas de segurança, isso permite que a organização reduza o risco geral e acelere os tempos de resposta futuros.

Dito isso, abaixo segue algumas ferramentas retiradas originalmente da coleção disponibilizada pelo [Meirwah](https://github.com/meirwah/awesome-incident-response).

Ferramentas para simulação adversária;

- [APTSimulator](https://github.com/NextronSystems/APTSimulator)
- [Caldera](https://github.com/mitre/caldera)
- [Red Team Automation (RTA) — O RTA](https://github.com/endgameinc/RTA)

Ferramenta de criação de imagem de disco;

- [AccessData FTK Imager](http://accessdata.com/product-download/?/support/adownloads#FTKImager)
- [Bitscout — Bitscout](https://github.com/vitaly-kamluk/bitscout)
- [GetData Forensic Imager](http://www.forensicimager.com/)
- [Guymager](http://guymager.sourceforge.net/)

Ferramentas de análise de log;

- [APT Hunter — APT-Hunter](https://github.com/ahmedkhlief/APT-Hunter)
- [Event Log Explorer](https://eventlogxp.com/)
- [Observador de Log de Eventos](https://lizard-labs.com/event_log_observer.aspx)
- [Kaspersky CyberTrace](https://support.kaspersky.com/13850)
- [LogonTracer](https://github.com/JPCERTCC/LogonTracer)
- [WELA](https://github.com/Yamato-Security/WELA)

Ferramentas de análise de memória;

- [AVML](https://github.com/microsoft/avml)
- [Evolve](https://github.com/JamesHabben/evolve)
- [inVtero.net](https://github.com/ShaneK2/inVtero.net)
- [LiME](https://github.com/504ensicsLabs/LiME)
- [Memoryze](https://www.fireeye.com/services/freeware/memoryze.html)

Coleta de evidências;

- [bulk_extractor](https://github.com/simsong/bulk_extractor)
- [CyLR](https://github.com/orlikoski/CyLR)
- [UAC — UAC](https://github.com/tclahr/uac)
- [DFIR ORC — DFIR ORC](https://dfir-orc.github.io/)
- [Fibratus](https://github.com/rabbitstack/fibratus)
- [Hoarder](https://github.com/muteb/Hoarder)

Gerenciamento de incidentes;

- [Catalyst](https://github.com/SecurityBrewery/catalyst)
- [CORTEX XSOAR](https://www.paloaltonetworks.com/cortex/xsoar)
- [Resposta Rápida a Incidentes (FIR)](https://github.com/certsocietegenerale/FIR/)
- [RTIR](https://www.bestpractical.com/rtir/)

Ferramenta Sandbox/Reverse;

- [AMAaaS](https://amaaas.com/index.php/AMAaaS/dashboard)
- [Any Run](https://app.any.run/)
- [CAPEv2](https://github.com/kevoreilly/CAPEv2)
- [Cuckoo](https://github.com/cuckoosandbox/cuckoo)
- [Ghidra](https://github.com/NationalSecurityAgency/ghidra)
- [Virustotal](https://www.virustotal.com/)

Ferramentas “all-in-one”;

- [GRR Rapid Response](https://github.com/google/grr)
- [IRIS — IRIS](https://github.com/dfir-iris/iris-web)
- [Sleuth Kit & Autopsy](http://www.sleuthkit.org/)
- [TheHive / Cortex](https://thehive-project.org/)
- [Velociraptor](https://github.com/Velocidex/velociraptor)
- [X-Ways Forensics](http://www.x-ways.net/forensics/)
- [Diffy](https://github.com/Netflix-Skunkworks/diffy)
- [Digital Survey and Analysis Appliance (ADIA) — Dispositivo](https://forensics.cert.org/#ADIA)
- [Computer Aided Investigative Environment (CAINE)](http://www.caine-live.net/index.html)
- [Security Onion .](https://github.com/Security-Onion-Solutions/security-onion)
- [HELK](https://github.com/Cyb3rWard0g/HELK)
- [RedHunt-OS](https://github.com/redhuntlabs/RedHunt-OS)

PLaybook;

- [GuardSIght Playbook Battle Cards](https://github.com/guardsight/gsvsoc_cirt-playbook-battle-cards)
- [IRM](https://github.com/certsocietegenerale/IRM)
- [ThreatHunter-Playbook](https://github.com/OTRF/ThreatHunter-Playbook)

Bases de conhecimentos (Livros,videos,comunidade);

- [Resposta Aplicada a Incidentes — Livro de Steve Anson.](https://www.amazon.com/Applied-Incident-Response-Steve-Anson/dp/1119560268/)
- [Arte da Forense da Memória — Detectando Malware e Ameaças na Memória do Windows, Linux e Mac.](https://www.amazon.com/Art-Memory-Forensics-Detecting-Malware/dp/1118825098/)
- [Elaboração do Manual InfoSec: Plano Diretor de Monitoramento de Segurança e Resposta a Incidentes — Por Jeff Bollinger, Brandon Enright e Matthew Valites.](https://www.amazon.com/Crafting-InfoSec-Playbook-Security-Monitoring/dp/1491949406)
- [Forense Digital e Resposta a Incidentes: Técnicas e procedimentos de resposta a incidentes para responder a ameaças cibernéticas modernas — Por Gerard Johansen.](https://www.amazon.com/Digital-Forensics-Incident-Response-techniques/dp/183864900X)
- [Introdução ao DFIR — Por Scott J. Roberts.](https://medium.com/@sroberts/introduction-to-dfir-d35d5de4c180/)
- [Técnicas de Resposta a Incidentes para Ataques de Ransomware — Por Oleg Skulkin.](https://www.amazon.com/Incident-Response-Techniques-Ransomware-Attacks/dp/180324044X)
- [Resposta a Incidentes Orientada por Inteligência — Por Scott J. Roberts, Rebekah Brown.](https://www.amazon.com/Intelligence-Driven-Incident-Response-Outwitting-Adversary-ebook-dp-B074ZRN5T7/dp/B074ZRN5T7)
- [Practical Memory Forensics — Por Svetlana Ostrovskaya e Oleg Skulkin.](https://www.amazon.com/Practical-Memory-Forensics-Jumpstart-effective/dp/1801070334)
- [Incidente Response Overview — Conjunto de materiais sobre o assunto, disponibilizado pelo Joas A Santos.](https://drive.google.com/file/d/1vxVYbIDkSGa3kuHd_bIN6J1dyqdBIJo-/view?usp=share_link)
- [Digital Forensics Discord Server — Comunidade de mais de 8.000 profissionais.](https://discordapp.com/invite/JUqe9Ek)
- [DFIR — Base de Conhecimento.](https://dfir.com.br/)
- [Base de Conhecimento de Artefatos Forenses Digitais](https://github.com/ForensicArtifacts/artifacts-kb)
- [O Futuro da Resposta a Incidentes — Apresentado por Bruce Schneier na OWASP AppSecUSA 2015.](https://www.youtube.com/watch?v=bDcx4UNpKNc)


