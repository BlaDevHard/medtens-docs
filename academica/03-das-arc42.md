# Documento de Arquitetura de Software (DAS) — MedTens

Estruturado conforme o modelo **arc42** (https://arc42.org/), versão
resumida, adaptada ao escopo de um Projeto Integrador.

**Versão:** 0.1 (esboço inicial)
**Data:** 2026-04-17
**Status:** em elaboração

> **Enquadramento no PPI Sumaré.** Artefato técnico de apoio à Metodologia
> do Projeto Integrador (ver `01-plano-tcc.md`, § 3). Não substitui nenhuma
> das cinco etapas obrigatórias do PPI, mas documenta de forma padronizada
> as decisões arquiteturais do artefato avaliado no trabalho.

---

## 1. Introdução e metas

### 1.1 Visão geral

O MedTens é um sistema embarcado distribuído para medir continuamente a
tensão AC em baixa tensão residencial (127 V / 60 Hz, Antonina-PR), apurar
os indicadores DRP e DRC conforme o PRODIST Módulo 8 Rev. 12 da ANEEL e
detectar eventos segundo a IEEE 1159, mantendo histórico em banco de dados
relacional.

### 1.2 Metas de qualidade (quality goals)

| Prio | Meta | Cenário verificador |
|---|---|---|
| 1 | **Precisão da medição** | Erro RMS ≤ ±3 % na faixa 90–150 V contra instrumento true-RMS de referência |
| 2 | **Disponibilidade** | ≥ 99 % em janela de 24 h (operação contínua ≥ 168 h) |
| 3 | **Integridade dos dados** | 0 % de frames com CRC inválido aceitos no *gateway*; lacunas ≤ 5 s por reconexão MQTT |
| 4 | **Reprodutibilidade** | Build determinístico a partir do repositório público com dependências pinadas |
| 5 | **Custo** | Materiais ≤ R$ 500 por ponto de medição |

### 1.3 Stakeholders

| Papel | Expectativa |
|---|---|
| Discente (autor) | Artefato funcional, documentado e reprodutível |
| Orientador(a) Sumaré | Aderência ao manual PPI + rigor metodológico |
| Avaliador(a) acadêmico(a) | Clareza, rastreabilidade requisito↔implementação |
| Usuário final (consumidor BT) | Dashboard legível, alertas de qualidade |
| Comunidade (Zenodo, GitHub) | Reprodutibilidade e citabilidade |

---

## 2. Restrições

### 2.1 Restrições técnicas

- **Microcontroladores fixados:** ATmega328P (Arduino Uno), ESP8266 (ESP01), ESP32
- **Banco de dados:** MySQL (compatível com a stack Home Assistant já existente)
- **Protocolo de aplicação:** MQTT (broker existente em 192.168.1.10)
- **Ambiente de build:** PlatformIO Core em Linux
- **Rede local:** WiFi 2.4 GHz; exposição pública não contemplada na v1

### 2.2 Restrições organizacionais

- **Instituição:** Centro Universitário Sumaré, curso ADS
- **Formato:** Projeto Integrador (PPI) — 5 etapas obrigatórias
- **Idioma:** português do Brasil (pt-BR) em toda a documentação
- **Projeto solo** — sem *pair programming* ou equipe

### 2.3 Restrições convencionais

- **Versionamento:** Conventional Commits + SemVer + *Keep a Changelog*
- **Licenças:** compatíveis com uso aberto e citação acadêmica
- **Uso do sistema:** estritamente informativo (não fiscal, não metrológico)

---

## 3. Contexto e escopo

### 3.1 Contexto de negócio

```
                          ┌──────────────────────┐
                          │   Rede BT 127V/60Hz  │
                          │   (Copel, Antonina)  │
                          └──────────┬───────────┘
                                     │ (observação)
                                     ▼
     ┌───────────────────── MedTens (artefato) ─────────────────────┐
     │                                                              │
     │  sensor → gateway → coletor → servidor ──► dashboards/alertas│
     │                                                              │
     └──────────────────────────────────────────────────────────────┘
                           │
                           ▼
       ┌─────────────────────────────────────┐
       │  Consumidor / Pesquisador / Banca   │
       └─────────────────────────────────────┘
```

### 3.2 Contexto técnico

| Interface | Tipo | Partner |
|---|---|---|
| Entrada AC | física (ZMPT101B) | Rede elétrica residencial |
| UART ATmega↔ESP01 | serial 9600 8N1, frame binário + CRC16 | próprio |
| MQTT ESP01↔ESP32 | TCP/IP, tópicos `medtens/sensor/<id>/…` | broker Mosquitto em 192.168.1.10 |
| SQL ESP32↔MySQL | TCP/IP, conexão nativa MySQL | servidor MySQL residencial |
| NTP (ESP01/ESP32) | UDP | pool.ntp.br |

---

## 4. Estratégia de solução

**Princípios norteadores:**

1. **Separação clara por dispositivo**, cada um com única responsabilidade
   (sensor amostra, gateway transporta, coletor calcula, servidor guarda).
2. **Protocolo binário extensível com CRC** entre ATmega e ESP01 para
   detectar ruído do *level-shift* 5V→3,3V sem perder eventos.
3. **MQTT como cola assíncrona** para o resto do pipeline, permitindo
   reinícios independentes e consumidores adicionais (ex.: Home Assistant).
4. **Timestamping híbrido:** ATmega emite `t_ms` relativo (sem RTC);
   ESP01/ESP32 atribui o *epoch* absoluto via NTP no momento da recepção.
5. **Desenvolvimento incremental em releases SemVer** com ADRs por decisão
   não óbvia.

---

## 5. Visão de blocos (Building Blocks)

### 5.1 Nível 1 — macro

```
┌──────────────────┐ UART  ┌──────────────────┐ MQTT  ┌──────────────────┐ SQL ┌──────────────────┐
│  Sensor (AVR)    │──────►│  Gateway (ESP01) │──────►│ Coletor (ESP32)  │────►│  Servidor MySQL  │
│ ZMPT101B → A0    │ bin   │ parser + pub     │       │ indicadores + ev │     │ histórico + cons │
└──────────────────┘ +CRC  └──────────────────┘       └──────────────────┘     └──────────────────┘
```

### 5.2 Nível 2 — decomposição por bloco

#### 5.2.1 Sensor (ATmega328P)

- `adc`: inicialização e leitura A0
- `rms`: cálculo do valor eficaz em janela de ~2 ciclos
- `timer_ms`: contador de tempo relativo ao boot (Timer0 CTC 1 ms)
- `evt`: detecção e registro de eventos de queda
- `uart_tx`: frame binário `[SOF1=0xAA][SOF2=0x55][LEN][TYPE][PAYLOAD][CRC16 LE]`
- `main`: laço principal 500 ms

#### 5.2.2 Gateway (ESP8266/ESP01)

- `uart_rx`: consumidor de *bytes* e montador de frames
- `crc`: validação CRC16-CCITT
- `seq_checker`: detecção de frames perdidos via `seq`
- `mqtt_pub`: publicação em `medtens/sensor/<id>/vrms` e `.../diag`
- `mqtt_lwt`: *Last Will and Testament*
- `ntp_clock`: sincronização NTP e atribuição de *epoch* absoluto
- `reconnect`: reconexão WiFi/MQTT com *backoff*

#### 5.2.3 Coletor (ESP32)

- `mqtt_sub`: assinatura e desserialização
- `janela`: buffer circular de 10 minutos
- `classifier_prodist`: classifica leitura em adequada/precária/crítica
- `drp_drc`: apuração de DRP e DRC horários
- `event_detector`: *sag*/*swell*/*interruption* conforme IEEE 1159
- `sql_writer`: escrita em `medicoes_brutas` e `indices_qualidade`
- `buffer_persist`: *buffering* local quando MySQL estiver indisponível

#### 5.2.4 Servidor (MySQL)

- Schema inicial:
  - `medicoes_brutas(id, ts_epoch_ms, ts_sensor_ms, seq, vrms, flags)`
  - `indices_qualidade(id, ts_inicio, ts_fim, drp, drc, n_leituras)`
  - `eventos(id, ts_inicio, ts_fim, tipo, severidade, v_min, v_max)`

---

## 6. Visão de runtime (cenários)

### 6.1 Cenário feliz — medição e persistência

1. Sensor amostra 3000 valores ADC, calcula `vrms`.
2. Sensor monta frame `[SOF1][SOF2][LEN=9][TYPE=0x01][seq|vrms|flags][CRC16]`.
3. Gateway recebe, valida CRC, publica em `medtens/sensor/<id>/vrms` com
   payload JSON `{seq, vrms, flags, ts_epoch_ms}`.
4. Coletor assina, insere em `medicoes_brutas`, atualiza janela.
5. A cada 10 min, coletor apura DRP/DRC e grava em `indices_qualidade`.

### 6.2 Cenário de queda de energia

1. Sensor detecta `vrms < 10 V` e emite `EVT QUEDA_INICIO t_ms=…`.
2. Enquanto persiste a queda, publica `QUEDA_EM_CURSO` como *heartbeat*.
3. Ao retorno, emite `QUEDA_FIM t_ms=… dur_ms=…`.
4. Gateway/Coletor calculam `ts_inicio_epoch = ts_atual_epoch − dur_ms` para
   grafar o evento com marco absoluto correto, mesmo que o gateway tenha
   perdido parte da janela (dependendo da própria alimentação).
5. Limitação conhecida (documentar): se o *gateway* estiver alimentado pela
   mesma rede em falta, a queda só será reportada após retorno — o que
   ainda preserva a duração pelo timestamp relativo do sensor.

### 6.3 Cenário de falha MQTT

1. Coletor detecta desconexão do *broker*.
2. Mensagens chegando via `mqtt_sub` são roteadas para *buffer* em SPIFFS.
3. Na reconexão, *flush* do buffer em ordem cronológica.

---

## 7. Visão de implantação

| Nó | Dispositivo | Localização |
|---|---|---|
| Sensor | Arduino Uno + ZMPT101B | Ponto de medição (tomada residencial) |
| Gateway | ESP01 | Mesmo ponto, alimentado em 3,3 V |
| Coletor | ESP32 | Rack residencial |
| Broker MQTT | Mosquitto em Mi 8 Lite | 192.168.1.10 |
| MySQL | Mesmo servidor Mi 8 Lite | 192.168.1.10 |
| Dashboard | Home Assistant (opcional) | 192.168.1.10 |

---

## 8. Conceitos transversais

- **Logging:** cada dispositivo emite linhas prefixadas (`MED`, `EVT`, `DIAG`)
  para facilitar *grep* em análise offline.
- **Segurança:** credenciais em `secrets.h` / `.env` fora do VCS; LAN-only.
- **Internacionalização:** tudo em pt-BR; strings não traduzidas.
- **Testes:** unitários para CRC16 e parser de frame; integração com gerador
  de sinal/fonte CA; validação metrológica contra true-RMS.
- **Documentação:** Markdown versionado + BibTeX ABNT.

---

## 9. Decisões arquiteturais

Ver pasta [`adr/`](adr/) — um arquivo por decisão. Índice atual:

| ADR | Título | Status |
|---|---|---|
| [ADR-0001](adr/ADR-0001-topologia-4-componentes.md) | Topologia em quatro componentes distribuídos | Accepted |
| [ADR-0002](adr/ADR-0002-protocolo-uart-binario-crc16.md) | Protocolo UART binário com CRC16 entre ATmega e ESP01 | Accepted |
| [ADR-0003](adr/ADR-0003-mqtt-como-barramento.md) | MQTT como barramento de integração | Accepted |
| [ADR-0004](adr/ADR-0004-timestamp-hibrido.md) | Timestamp híbrido (relativo no sensor, absoluto no gateway) | Accepted |
| [ADR-0005](adr/ADR-0005-mysql-persistencia.md) | MySQL como armazenamento histórico | Accepted |

---

## 10. Qualidade (quality tree resumida)

- **Confiabilidade** → integridade UART (CRC), reconexão MQTT, *buffering* SQL
- **Precisão** → calibração formal, ensaio contra true-RMS
- **Manutenibilidade** → *clang-format*, ADRs, *small releases*
- **Reprodutibilidade** → dependências pinadas, DOI Zenodo

---

## 11. Riscos e dívidas técnicas

| Risco | Impacto | Mitigação |
|---|---|---|
| Alimentação do gateway cair junto com a rede | Perda de evento em tempo real | Relato do evento após retorno com base no `dur_ms` do sensor |
| Ruído do level-shift 5V→3,3V corromper UART | Medições inválidas | CRC16 no frame, métrica de descartes |
| Drift do cristal ATmega ao longo do tempo | Desvio do `t_ms` entre syncs | Atribuição absoluta no gateway a cada pacote via NTP |
| MySQL inacessível na LAN | Perda de janela | *Buffer* local em SPIFFS no coletor |
| Calibração única envelhecer | Erro crescente | Recalibração periódica registrada no lab-notebook |

---

## 12. Glossário

| Termo | Definição |
|---|---|
| ADR | Architecture Decision Record |
| arc42 | Modelo de documentação arquitetural mantido por G. Starke |
| DAS | Documento de Arquitetura de Software |
| DRP | Duração Relativa da transgressão de tensão Precária |
| DRC | Duração Relativa da transgressão de tensão Crítica |
| MQTT | Message Queuing Telemetry Transport |
| PPI | Projeto Integrador (formato Sumaré) |
| RMS | Root Mean Square (valor eficaz) |
| SPIFFS | SPI Flash File System (sistema de arquivos em flash) |
