# Projeto Integrador (PPI) — Plano de Pesquisa

**Título provisório:** MedTens — Sistema embarcado distribuído para monitoramento e avaliação contínua da qualidade do fornecimento de energia elétrica em baixa tensão residencial

**Instituição:** Centro Universitário Sumaré
**Curso:** Análise e Desenvolvimento de Sistemas (ADS)
**Formato:** Projeto Integrador (PPI), conforme *Manual de Trabalho Científico para os Projetos Integradores* da Sumaré
**Orientador(a):** *a definir*
**Discente:** *a preencher*
**Local do ponto de medição:** Antonina–PR (área de concessão da Copel Distribuição)
**Data:** 2026-04-17 (v0.2 — pré-projeto, formatado como PPI)

> Este documento segue a estrutura obrigatória do PPI Sumaré (5 etapas:
> Introdução, Referencial Teórico, Metodologia, Análise dos Dados, Conclusão).
> Citações e referências seguem **ABNT NBR 6023 e NBR 10520**, conforme o
> Manual Acadêmico Sumaré.

---

## 1. Introdução

O fornecimento de energia elétrica em baixa tensão no Brasil é regulado pela
Agência Nacional de Energia Elétrica (ANEEL) através dos Procedimentos de
Distribuição (PRODIST), em especial o **Módulo 8 — Qualidade do Fornecimento
de Energia Elétrica**, cuja Revisão 12 foi aprovada pela Resolução Normativa
ANEEL nº 956/2021 e atualizada pela Resolução Normativa ANEEL nº 1148, de 27
de janeiro de 2026 (ANEEL, 2021; ANEEL, 2026). O módulo estabelece limites de
tensão em regime permanente (adequada, precária, crítica) e obriga as
distribuidoras a compensar consumidores quando os indicadores de duração
relativa das transgressões — DRP (precária) e DRC (crítica) — ultrapassam
limites regulados. A apuração oficial exige medição contínua por período
mínimo de sete dias (1008 leituras de dez minutos).

Apesar da existência desse arcabouço regulatório, o consumidor final raramente
dispõe de instrumentação própria para verificar a conformidade do serviço que
recebe, dependendo da auto-declaração da concessionária. No contexto deste
trabalho, o ponto de medição situa-se em Antonina-PR, na área de concessão da
**Companhia Paranaense de Energia (Copel Distribuição)**, holding integrada
responsável pela distribuição de energia elétrica em baixa tensão em todo o
estado do Paraná. Esse cenário de assimetria informacional torna-se mais
crítico a partir da **Lei nº 15.269, de 24 de novembro de 2025** (BRASIL,
2025), que abriu o Ambiente de Contratação Livre (ACL) para todos os
consumidores — incluindo os de baixa tensão — e criou novas exigências de
transparência e mensuração no lado do consumidor.

Sistemas comerciais de medição de qualidade de energia (*power quality meters*
classe A ou S) apresentam custo de aquisição elevado — tipicamente entre
R$ 3.000 e R$ 30.000 — inviável para uso doméstico ou em pequenos comércios.
Trabalhos acadêmicos recentes demonstram a viabilidade de soluções embarcadas
de baixo custo para propósitos correlatos, como o medidor inteligente baseado
em ESP32 + ZMPT101B + SCT-013 proposto por Camargo (2025) na UNESP-Sorocaba,
o *dashboard* móvel sobre ESP32 + SCT-013 apresentado por Ribeiro (2025) no
IFAM, e o dispositivo PQEm baseado em ADE9000 + Raspberry Pi descrito por
Moustafa *et al.* (2025) na *IEEE Access*.

### 1.1 Problematização

Segundo Gil (2017) e Marconi e Lakatos (2022), uma boa problematização exige
pergunta de pesquisa **precisa**, **empírica**, **passível de solução** e
**delimitada**, acompanhada de hipótese testável.

**Pergunta de pesquisa:**

> É viável construir um artefato embarcado distribuído de baixo custo
> (custo de materiais inferior a R$ 500 por ponto de medição), baseado nos
> microcontroladores ATmega328P, ESP8266 e ESP32, capaz de medir continuamente
> a tensão AC residencial em 127 V / 60 Hz em Antonina–PR, calcular os
> indicadores DRP e DRC conforme o PRODIST Módulo 8 Rev. 12 e detectar eventos
> de afundamento, elevação e interrupção segundo a IEEE Std 1159-2019, com
> erro relativo da medição RMS de tensão inferior a ±3 % em relação a
> instrumento true-RMS de referência, sustentando operação contínua por pelo
> menos 168 horas (sete dias, período mínimo de apuração oficial)?

A pergunta é **precisa** (define grandeza, faixa, erro máximo e duração),
**empírica** (mensurável em laboratório e em ambiente real), **passível de
solução** (componentes disponíveis no mercado brasileiro) e **delimitada**
(monofásico, baixa tensão, uso não-fiscal, um ponto de medição em Antonina–PR).

**Hipótese:**

> É possível construir o referido artefato atendendo simultaneamente aos
> critérios de custo, precisão e continuidade operacional, **desde que** o
> sensor ZMPT101B seja calibrado contra instrumento true-RMS de referência,
> o transporte UART entre ATmega e ESP8266 utilize frame binário com CRC16
> para garantir integridade, e o coletor ESP32 implemente janela deslizante
> de 10 minutos para apuração dos indicadores de qualidade e *buffering*
> local em caso de indisponibilidade do servidor.

### 1.2 Justificativa

Seguindo a estrutura do Manual Sumaré, a justificativa abrange quatro aspectos:

**Aspecto científico.** O trabalho contribui para a literatura brasileira
sobre instrumentação embarcada de qualidade de energia, complementando
trabalhos como Camargo (2025) — que trata de medição de consumo e não
incorpora indicadores do PRODIST 8 — e Ribeiro (2025) — que enfatiza
visualização móvel. O MedTens se diferencia por focar **nos indicadores
regulatórios brasileiros** (DRP, DRC) e seguir norma internacional (IEEE
1159) para classificação de eventos, aproximando-se metodologicamente do
PQEm descrito por Moustafa *et al.* (2025), mas com uso de
microcontroladores amplamente disponíveis no mercado nacional.

**Aspecto social.** Fornece ao consumidor final em baixa tensão — em
particular no contexto pós Lei 15.269/2025, em que o ACL se estende ao
consumidor residencial — uma ferramenta aberta e de baixo custo para
acompanhar a qualidade do serviço contratado, reduzindo a assimetria
informacional entre consumidor e concessionária (neste caso, a Copel
Distribuição). O artefato torna-se passível de uso por residências e
pequenos comércios que hoje não têm acesso a medidores classe A.

**Aspecto institucional.** Integra e demonstra competências desenvolvidas
ao longo do curso de Análise e Desenvolvimento de Sistemas (ADS) do Centro
Universitário Sumaré — engenharia de requisitos, arquitetura de software,
sistemas embarcados, redes de computadores, banco de dados e análise de
dados — em um único artefato reprodutível, alinhado à natureza integradora
do PPI prevista no manual institucional.

**Aspecto pessoal.** O discente reside em Antonina–PR e dispõe de
ambiente residencial para instalação, operação e observação contínua do
artefato, permitindo coleta real de dados ao longo de meses. A escolha do
tema combina afinidade com sistemas embarcados, interesse em regulação do
setor elétrico e disponibilidade da infraestrutura doméstica necessária
para a fase de operação contínua.

### 1.3 Objetivos

Os verbos empregados são extraídos da tabela de verbos autorizados do
Manual Acadêmico Sumaré (analisar, comparar, investigar, identificar,
demonstrar, interpretar, relacionar, descrever, relatar, concluir,
determinar, registrar, definir, medir, especificar, enumerar, exemplificar,
formular, planejar, elaborar, organizar, propor, distinguir).

#### 1.3.1 Objetivo geral

**Desenvolver** e **avaliar** um sistema embarcado distribuído de baixo
custo para monitoramento contínuo da tensão AC residencial em Antonina–PR,
com cálculo automatizado dos indicadores de qualidade DRP e DRC conforme o
PRODIST Módulo 8, detecção de eventos segundo a IEEE 1159 e persistência
histórica em banco de dados relacional MySQL.

#### 1.3.2 Objetivos específicos

1. **Especificar** os requisitos funcionais e não-funcionais do sistema
   conforme a norma ISO/IEC/IEEE 29148:2018.
2. **Elaborar** a arquitetura do sistema utilizando o modelo arc42,
   **registrando** cada decisão arquitetural não óbvia em *Architecture
   Decision Records* (ADR).
3. **Implementar** o firmware embarcado no ATmega328P para aquisição de
   sinal via sensor ZMPT101B e cálculo da tensão RMS com registro de
   eventos de queda de energia.
4. **Implementar** o firmware de *gateway* no ESP8266 (ESP01) para
   transporte UART em frame binário com CRC16 e republicação em *broker*
   MQTT.
5. **Implementar** o coletor em ESP32 para cálculo dos indicadores DRP e
   DRC em janela deslizante, detecção de afundamentos e elevações segundo
   IEEE 1159, e persistência em MySQL.
6. **Planejar** e **executar** protocolo de calibração do sensor ZMPT101B
   contra instrumento true-RMS de referência.
7. **Medir** a precisão, a latência e a disponibilidade do sistema em
   ensaios controlados e em operação contínua mínima de 168 horas.
8. **Analisar** os dados coletados e **comparar** os indicadores apurados
   pelo MedTens com os limites regulados pelo PRODIST Módulo 8.

---

## 2. Referencial Teórico

A revisão da literatura se organiza em quatro eixos:

### 2.1 Regulação do setor elétrico brasileiro

- Lei nº 9.074/1995 — abertura inicial do setor.
- Decreto nº 5.163/2004 — consolidação de ACR e ACL.
- **Lei nº 15.269/2025** — abertura do ACL para baixa tensão.
- **REN ANEEL nº 956/2021** e **REN nº 1148/2026** — PRODIST Módulo 8 Rev. 12.

### 2.2 Qualidade da energia elétrica

- PRODIST Módulo 8 Rev. 12 (ANEEL, 2021): faixas de tensão, DRP, DRC,
  procedimento de medição (7 dias / 1008 leituras), direito de compensação.
- IEEE Std 1159-2019: classificação de distúrbios (*sag*, *swell*, *interruption*,
  *harmonic distortion*, *voltage imbalance*).
- IEC 61000-4-15 (*flicker*) e IEC 61000-4-30 (métodos de medição).
- Camargo (2025), Ribeiro (2025), Moustafa *et al.* (2025) — trabalhos
  correlatos.

### 2.3 Sistemas embarcados e arquitetura distribuída

- Arquitetura AVR (ATmega328P): ADC, amostragem, cálculo RMS digital.
- ESP8266 e ESP32: pilha WiFi, MQTT, persistência.
- Padrão *publisher/subscriber*; desacoplamento temporal.

### 2.4 Engenharia de software e metodologia de pesquisa

- Engenharia de requisitos: ISO/IEC/IEEE 29148:2018.
- Arquitetura: arc42, ADRs (Nygard, 2011), visões 4+1 (Kruchten, 1995).
- Engenharia ágil: Extreme Programming (Beck; Andres, 2004).
- Metodologia científica: Gil (2017), Marconi e Lakatos (2022), Lozada e
  Grassi-Kassisse (2019), Creswell (2010).
- Design Science Research: Hevner *et al.* (2004), Dresch, Lacerda e
  Antunes Jr. (2015).

---

## 3. Metodologia

### 3.1 Classificação da pesquisa

Seguindo a taxonomia apresentada por Gil (2017) e referenciada no Manual
Sumaré, o trabalho classifica-se como:

| Critério | Classificação |
|---|---|
| **Quanto à natureza** | Aplicada (desenvolvimento de artefato) |
| **Quanto aos objetivos** | Explicativa e aplicada |
| **Quanto à abordagem** | Predominantemente quantitativa (grandezas elétricas, tempos, erros) com componente qualitativo na análise de eventos |
| **Quanto aos procedimentos técnicos** | Experimental (calibração contra instrumento de referência) + pesquisa-ação (operação contínua do artefato em ambiente residencial real) |

### 3.2 Enquadramento metodológico complementar — Design Science Research (DSR)

Como o artefato (sistema embarcado distribuído) é o produto central da
pesquisa, adota-se, de forma complementar à classificação acima, a Design
Science Research (HEVNER *et al.*, 2004; DRESCH; LACERDA; ANTUNES JR.,
2015). A DSR é **compatível** com a estrutura do PPI Sumaré: as cinco etapas
obrigatórias do PPI (Introdução, Referencial Teórico, Metodologia, Análise,
Conclusão) acomodam as seis etapas da DSR conforme o mapeamento abaixo.

| Etapa DSR | Etapa PPI correspondente |
|---|---|
| Identificação do problema | Introdução (§ 1) |
| Definição de objetivos de solução | Introdução — Objetivos (§ 1.3) |
| Projeto e desenvolvimento | Metodologia (§ 3.3) + ERS + DAS |
| Demonstração | Metodologia — operação contínua (§ 3.4) |
| Avaliação | Análise dos Dados (§ 4) |
| Comunicação | Esta monografia + repositórios públicos com DOI Zenodo |

> **Nota ao orientador:** caso a DSR não seja reconhecida pela banca, o
> trabalho pode ser conduzido e descrito usando exclusivamente o vocabulário
> do Manual Sumaré (pesquisa experimental + pesquisa-ação), sem perda de
> rigor metodológico.

### 3.3 Procedimentos de desenvolvimento do artefato

Adotam-se práticas de Extreme Programming (BECK; ANDRES, 2004) aplicáveis
ao contexto solo-acadêmico:

- *Small releases* — cada incremento funcional gera *release* versionada em
  SemVer, com tag anotada.
- *Simple design* — cada decisão arquitetural não óbvia registrada em ADR.
- *Continuous integration* — compilação via PlatformIO a cada *push*.
- *Coding standards* — `.clang-format` e `.editorconfig` versionados.
- *Testing* — testes unitários de *parsers* (CRC16, decodificador de frame
  binário) e testes de integração *hardware-in-the-loop*.
- *Refactoring* contínuo, apoiado por *Conventional Commits*.

Práticas não aplicáveis por se tratar de projeto solo: *pair programming*,
*whole team*, *on-site customer*.

### 3.4 Procedimentos de coleta de dados

A coleta ocorre em três momentos complementares:

1. **Calibração** (ensaio controlado): sensor ZMPT101B submetido a tensões
   de referência entre 90 V e 150 V, comparado contra multímetro true-RMS de
   referência. Obtém-se fator de calibração e envelope de erro.
2. **Ensaio de integração**: geração artificial de frames UART com CRC
   inválidos e interrupções de MQTT para verificar robustez do *pipeline*.
3. **Operação contínua**: instalação do artefato em ponto de medição
   residencial em Antonina–PR, operando por período mínimo de 168 horas
   (7 dias, compatível com o período mínimo exigido pelo PRODIST Módulo 8),
   gerando registros de medições brutas, eventos de queda e indicadores
   agregados em MySQL.

### 3.5 Instrumentos e artefatos

- **Hardware:** Arduino Uno (ATmega328P) + ZMPT101B, ESP8266 (ESP01),
  ESP32, *broker* MQTT em servidor residencial.
- **Software:** PlatformIO, Git, GitHub, MySQL, MQTT *broker* Mosquitto.
- **Gerência:** Conventional Commits, SemVer, *Keep a Changelog*.
- **Rastreabilidade:** caderno de laboratório digital em
  `docs/lab-notebook/AAAA-MM-DD.md` para cada sessão significativa.
- **Reprodutibilidade:** dependências pinadas (`platformio.ini`); DOI
  Zenodo por *release*.

---

## 4. Análise dos Dados *(a ser redigida após a fase de operação contínua)*

Esta seção receberá, ao final da operação, os seguintes conteúdos:

- Erro medido da tensão RMS em relação ao instrumento true-RMS.
- Distribuição das medições nas faixas PRODIST (adequada / precária /
  crítica).
- Valores apurados de DRP e DRC para a janela observada.
- Lista de eventos classificados segundo IEEE 1159 (*sag*, *swell*,
  *interruption*).
- Discussão sobre conformidade (ou não) com os limites regulados na área
  de concessão da Copel Distribuição.
- Comparação qualitativa com resultados reportados por Camargo (2025),
  Ribeiro (2025) e Moustafa *et al.* (2025).

Métodos estatísticos previstos: análise descritiva, histogramas e
*box-plots* das medições; cálculo de intervalos de confiança para o erro de
calibração.

---

## 5. Conclusão *(a ser redigida ao fim do PPI)*

Retomada dos objetivos geral e específicos, apresentação das respostas à
pergunta de pesquisa, discussão sobre a confirmação ou refutação da
hipótese, limitações identificadas e proposição de trabalhos futuros.

---

## 6. Cronograma

Granularidade mensal; ajustada após definição do calendário acadêmico
Sumaré.

| Fase | Duração | Entregáveis |
|---|---|---|
| F1 — Pré-projeto (PPI) | 1 mês | Este documento, aprovação pelo orientador |
| F2 — Especificação | 1 mês | ERS finalizada, DAS arc42, ADRs iniciais |
| F3 — Implementação sensor | 1 mês | Firmware ATmega v1.0 + testes unitários |
| F4 — Implementação gateway e coletor | 2 meses | ESP01 *gateway*, ESP32 coletor, *schema* MySQL |
| F5 — Calibração e validação | 1 mês | Dados de calibração, relatório de ensaios |
| F6 — Operação contínua | 2 meses | Coleta real em Antonina-PR, análise estatística |
| F7 — Redação da monografia | 2 meses | PPI escrito conforme Manual Sumaré |
| F8 — Defesa | — | Apresentação |

---

## 7. Resultados esperados

- Artefato funcional (hardware + firmware + software + banco) operando
  continuamente por pelo menos 168 horas em Antonina–PR.
- Demonstração de precisão ≤ ±3 % na faixa 90–150 V relativamente a
  instrumento true-RMS de referência.
- Relatório de indicadores DRP/DRC e eventos conforme PRODIST Módulo 8 e
  IEEE 1159.
- Repositórios públicos com DOI citável (Zenodo) e documentação completa.

---

## 8. Referências

Formatação conforme **ABNT NBR 6023** e **ABNT NBR 10520**, nos termos do
Manual Acadêmico Sumaré. Lista completa e em BibTeX em
[`../docs/bibliografia.bib`](../docs/bibliografia.bib).

- AGÊNCIA NACIONAL DE ENERGIA ELÉTRICA (ANEEL). **PRODIST — Módulo 8: Qualidade do Fornecimento de Energia Elétrica**. Revisão 12. Aprovado pela Resolução Normativa ANEEL nº 956/2021. Brasília: ANEEL, 2021.
- AGÊNCIA NACIONAL DE ENERGIA ELÉTRICA (ANEEL). **Resolução Normativa ANEEL nº 1148, de 27 de janeiro de 2026**. Altera os Anexos VI e VIII da REN 956/2021. Brasília: ANEEL, 2026.
- BECK, K.; ANDRES, C. **Extreme Programming Explained: Embrace Change**. 2. ed. Boston: Addison-Wesley, 2004.
- BRASIL. **Lei nº 15.269, de 24 de novembro de 2025**. Moderniza o marco regulatório do setor elétrico. Brasília: Diário Oficial da União, 2025.
- CAMARGO, M. L. **Sistema embarcado para medição de energia e monitoramento online**. 2025. 59 f. Trabalho de Conclusão de Curso (Bacharelado em Engenharia de Controle e Automação) — Universidade Estadual Paulista (UNESP), Sorocaba, 2025.
- CRESWELL, J. W. **Projeto de pesquisa: métodos qualitativo, quantitativo e misto**. 3. ed. Porto Alegre: Artmed, 2010.
- DRESCH, A.; LACERDA, D. P.; ANTUNES JR., J. A. V. **Design Science Research: método de pesquisa para avanço da ciência e tecnologia**. Porto Alegre: Bookman, 2015.
- GIL, A. C. **Como elaborar projetos de pesquisa**. 6. ed. São Paulo: Atlas, 2017.
- HEVNER, A. R. *et al.* Design Science in Information Systems Research. **MIS Quarterly**, v. 28, n. 1, p. 75-105, 2004.
- IEEE. **IEEE Std 1159-2019 — Recommended Practice for Monitoring Electric Power Quality**. New York: IEEE, 2019.
- ISO/IEC/IEEE. **ISO/IEC/IEEE 29148:2018 — Systems and software engineering — Life cycle processes — Requirements engineering**. Geneva: ISO, 2018.
- KRUCHTEN, P. The 4+1 View Model of Architecture. **IEEE Software**, v. 12, n. 6, p. 42-50, nov. 1995.
- LOZADA, G.; GRASSI-KASSISSE, M. H. **Metodologia Científica**. Porto Alegre: SAGAH, 2019.
- MARCONI, M. A.; LAKATOS, E. M. **Fundamentos de Metodologia Científica**. 9. ed. São Paulo: Atlas, 2022.
- MOUSTAFA, R. *et al.* A Smart Web-Based Power Quality and Energy Monitoring System With Enhanced Features. **IEEE Access**, v. 13, p. 88458-88471, 2025. DOI: 10.1109/ACCESS.2025.3571623.
- NYGARD, M. **Documenting Architecture Decisions**. 2011. Disponível em: https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions.
- RIBEIRO, M. V. R. **Medidor de energia elétrica com monitoramento inteligente utilizando dashboard mobile**. 2025. 66 f. Trabalho de Conclusão de Curso (Bacharelado em Engenharia de Controle e Automação) — Instituto Federal de Educação, Ciência e Tecnologia do Amazonas (IFAM), Manaus, 2025.
