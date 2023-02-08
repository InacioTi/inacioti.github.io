---
title: Introducao ao DFIR
author: José Inácio
date: 2022-12-21
categories: [Papers, Tutorial]
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

Dito isso, abaixo segue algumas ferramentas retiradas originalmente da coleção disponibilizada pelo Meirwah<https://github.com/meirwah/awesome-incident-response>.

Ferramentas para simulação adversária;

- APTSimulator<https://github.com/NextronSystems/APTSimulator>
- Caldera
- Red Team Automation (RTA) — O RTA

Ferramenta de criação de imagem de disco;

- AccessData FTK Imager
- Bitscout — Bitscout
- GetData Forensic Imager
- Guymager

Ferramentas de análise de log;

- APT Hunter — APT-Hunter
- Event Log Explorer
- Observador de Log de Eventos
- Kaspersky CyberTrace
- LogonTracer
- WELA

Ferramentas de análise de memória;

- AVML
- Evolve
- inVtero.net
- LiME
- Memoryze

Coleta de evidências;

- bulk_extractor
- CyLR
- UAC — UAC
- DFIR ORC — DFIR ORC
- Fibratus
- Hoarder

Gerenciamento de incidentes;

- Catalyst
- CORTEX XSOAR
- Resposta Rápida a Incidentes (FIR)
- RTIR

Ferramenta Sandbox/Reverse;

- AMAaaS
- Any Run
- CAPEv2
- Cuckoo
- Ghidra
- Virustotal

Ferramentas “all-in-one”;

- GRR Rapid Response
- IRIS — IRIS
- Sleuth Kit & Autopsy
- TheHive / Cortex
- Velociraptor
- X-Ways Forensics
- Diffy
- Digital Survey and Analysis Appliance (ADIA) — Dispositivo
- Computer Aided Investigative Environment (CAINE)
- Security Onion .
- HELK
- RedHunt-OS

PLaybook;

- GuardSIght Playbook Battle Cards
- IRM
- ThreatHunter-Playbook

Bases de conhecimentos (Livros,videos,comunidade);

- Resposta Aplicada a Incidentes — Livro de Steve Anson.
- Arte da Forense da Memória — Detectando Malware e Ameaças na Memória do Windows, Linux e Mac.
- Elaboração do Manual InfoSec: Plano Diretor de Monitoramento de Segurança e Resposta a Incidentes — Por Jeff Bollinger, Brandon Enright e Matthew Valites.
- Forense Digital e Resposta a Incidentes: Técnicas e procedimentos de resposta a incidentes para responder a ameaças cibernéticas modernas — Por Gerard Johansen.
- Introdução ao DFIR — Por Scott J. Roberts.
- Técnicas de Resposta a Incidentes para Ataques de Ransomware — Por Oleg Skulkin.
- Resposta a Incidentes Orientada por Inteligência — Por Scott J. Roberts, Rebekah Brown.
- Practical Memory Forensics — Por Svetlana Ostrovskaya e Oleg Skulkin.
- Incidente Response Overview — Conjunto de materiais sobre o assunto, disponibilizado pelo Joas A Santos.
- Digital Forensics Discord Server — Comunidade de mais de 8.000 profissionais.
- DFIR — Base de Conhecimento.
- Base de Conhecimento de Artefatos Forenses Digitais
- O Futuro da Resposta a Incidentes — Apresentado por Bruce Schneier na OWASP AppSecUSA 2015.


