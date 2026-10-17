# MedTens — Sistema de Monitoramento de Qualidade de Energia

Documentação central do projeto **MedTens**: monitoramento contínuo da tensão
AC da concessionária, com cálculo de índices de qualidade de fornecimento e
histórico em banco de dados para análise.

## Arquitetura

```
┌────────────────┐  UART 9600   ┌─────────────────┐   MQTT    ┌──────────────────┐   SQL   ┌─────────┐
│   ATmega328P   │ ───────────► │   ESP01 (WiFi)  │ ────────► │  ESP32 (coletor) │ ──────► │  MySQL  │
│  ZMPT101B → A0 │   binário    │    gateway      │  broker   │  índices + agreg │         │ histórico│
└────────────────┘   + CRC16    └─────────────────┘           └──────────────────┘         └─────────┘
       (sensor)                        (gateway)                    (coletor/análise)
```

## Componentes (repositórios)

| Papel | Repositório | Linguagem / Stack | Status |
|---|---|---|---|
| Sensor | [medtens-atmega-sensor](https://github.com/BlaDevHard/medtens-atmega-sensor) | C puro + PlatformIO (AVR) | v0.1.0 |
| Gateway | medtens-esp01-gateway *(a criar)* | C++ Arduino (ESP8266) | — |
| Coletor | medtens-esp32-collector *(a criar)* | C++ Arduino (ESP32) | — |
| Servidor | medtens-server *(a criar)* | MySQL + (TBD: PHP/Python) | — |
| Documentação | **este repositório** | Markdown | em construção |

## Conteúdo deste repositório

### Documentos acadêmicos (PPI Sumaré)

- [`academica/01-plano-tcc.md`](academica/01-plano-tcc.md) — plano do Projeto
  Integrador conforme o *Manual de Trabalho Científico para os Projetos
  Integradores* da Sumaré (5 etapas, problematização P.E.P.D. + hipótese,
  justificativa em 4 aspectos, objetivos com verbos do manual)
- [`academica/02-ers.md`](academica/02-ers.md) — Especificação de Requisitos
  de Software (ISO/IEC/IEEE 29148:2018) — artefato técnico de apoio ao PPI

### Documentos técnicos

- [`docs/arquitetura.md`](docs/arquitetura.md) — diagrama detalhado e fluxo de dados
- [`docs/protocolo-uart.md`](docs/protocolo-uart.md) — especificação do frame binário ATmega↔ESP01
- [`docs/indice-qualidade.md`](docs/indice-qualidade.md) — definição matemática dos índices
- [`docs/calibracao.md`](docs/calibracao.md) — metodologia de calibração do ZMPT101B
- [`docs/bibliografia.bib`](docs/bibliografia.bib) — referências em BibTeX (ABNT NBR 6023)
- [`docs/glossario.md`](docs/glossario.md) — glossário de siglas e expressões do projeto
- [`docs/normas-acreditacao.md`](docs/normas-acreditacao.md) — normas e vias de acreditação aplicáveis
- [`docs/lab-notebook/`](docs/lab-notebook/) — caderno de laboratório (um arquivo por sessão)

## Enquadramento acadêmico

O MedTens é desenvolvido como **Projeto Integrador (PPI)** do curso de
**Análise e Desenvolvimento de Sistemas (ADS)** do **Centro Universitário
Sumaré**, seguindo o manual institucional e a norma **ABNT** para citações
e referências. O ponto de medição encontra-se em **Antonina-PR**, na área
de concessão da **Copel Distribuição**.

**Marcos regulatórios aplicados:** PRODIST Módulo 8 Rev. 12 (REN ANEEL
956/2021 e REN 1148/2026), Lei 15.269/2025, IEEE Std 1159-2019,
IEC 61000-4-30.

**Metodologia:** pesquisa aplicada, experimental + pesquisa-ação,
complementada por *Design Science Research* (Hevner *et al.*, 2004;
Dresch, Lacerda e Antunes Jr., 2015). Desenvolvimento via práticas
solo-adaptadas de *Extreme Programming* (Beck; Andres, 2004).

### Princípios de reprodutibilidade

1. **Versões pinadas** — dependências fixas em `platformio.ini`
2. **Rastreabilidade** — cada *release* com tag SemVer e entrada em CHANGELOG
3. **Citabilidade** — Zenodo DOI por *release* (a configurar quando publicado)
4. **Transparência** — dados brutos e scripts de análise versionados

## Licença

A definir para todos os componentes.

## Citação

`CITATION.cff` será adicionado quando a autoria for formalizada.
