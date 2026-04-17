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

- [`docs/arquitetura.md`](docs/arquitetura.md) — diagrama detalhado e fluxo de dados
- [`docs/protocolo-uart.md`](docs/protocolo-uart.md) — especificação do frame binário ATmega↔ESP01
- [`docs/indice-qualidade.md`](docs/indice-qualidade.md) — definição matemática dos índices
- [`docs/calibracao.md`](docs/calibracao.md) — metodologia de calibração do ZMPT101B
- [`docs/bibliografia.bib`](docs/bibliografia.bib) — referências acadêmicas
- [`docs/lab-notebook/`](docs/lab-notebook/) — caderno de laboratório (um arquivo por sessão de trabalho)

## Motivação acadêmica

Este projeto visa publicação futura (TCC / artigo). Princípios adotados:

1. **Reprodutibilidade** — versões pinadas, instruções passo-a-passo de build
2. **Rastreabilidade** — cada release com tag SemVer e entrada em CHANGELOG
3. **Citabilidade** — Zenodo DOI por release (a configurar quando o repo for publicado)
4. **Transparência** — dados brutos de medição e scripts de análise versionados

## Licença

A definir para todos os componentes.

## Citação

`CITATION.cff` será adicionado quando a autoria for formalizada.
