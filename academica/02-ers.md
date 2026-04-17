# Especificação de Requisitos de Software (ERS) — MedTens

Conforme **ISO/IEC/IEEE 29148:2018** — *Systems and software engineering — Life cycle processes — Requirements engineering*.

**Versão:** 0.1 (esboço inicial)
**Data:** 2026-04-17
**Status:** em elaboração

---

## 1. Introdução

### 1.1 Propósito

Este documento especifica os requisitos do sistema MedTens, um sistema
embarcado distribuído de monitoramento de qualidade de energia elétrica em
baixa tensão residencial. Destina-se a orientadores, avaliadores acadêmicos
e desenvolvedores que contribuam com o projeto.

### 1.2 Escopo

O sistema compreende quatro componentes de software/firmware interconectados:

1. **Sensor** (firmware ATmega328P) — aquisição de tensão via ZMPT101B
2. **Gateway** (firmware ESP8266/ESP01) — ponte UART↔MQTT
3. **Coletor** (firmware ESP32) — cálculo de indicadores e persistência
4. **Servidor** (MySQL + consumidores) — histórico e consulta

**Fora de escopo** (delimitação):
- Medição de corrente, potência, fator de potência, energia ativa/reativa
- Atuação sobre a rede (relés de desligamento, by-pass)
- Uso fiscal ou metrológico (sistema é classificado como **informativo**)
- Suporte a sistemas trifásicos nesta versão

### 1.3 Definições, acrônimos e abreviações

| Termo | Definição |
|---|---|
| ADC | Analog-to-Digital Converter |
| ADR | Architecture Decision Record |
| ANEEL | Agência Nacional de Energia Elétrica |
| DRC | Duração Relativa da transgressão de tensão Crítica |
| DRP | Duração Relativa da transgressão de tensão Precária |
| ERS | Especificação de Requisitos de Software |
| MQTT | Message Queuing Telemetry Transport |
| PRODIST | Procedimentos de Distribuição de Energia Elétrica |
| RMS | Root Mean Square (valor eficaz) |
| THD | Total Harmonic Distortion |
| UART | Universal Asynchronous Receiver-Transmitter |

### 1.4 Referências

- ISO/IEC/IEEE 29148:2018
- ANEEL PRODIST Módulo 8 (2021)
- IEEE Std 1159-2019

### 1.5 Visão geral

- Seção 2: descrição geral, contexto, restrições
- Seção 3: requisitos funcionais
- Seção 4: requisitos não-funcionais
- Seção 5: restrições de projeto
- Seção 6: rastreabilidade

---

## 2. Descrição geral

### 2.1 Perspectiva do produto

Sistema independente, auto-contido, operando em rede local do usuário.
Integra-se opcionalmente a broker MQTT pré-existente (ex: Home Assistant).

### 2.2 Funções principais

- Medir continuamente a tensão AC de um ponto de consumo residencial
- Calcular indicadores de qualidade conforme PRODIST 8 (DRP, DRC)
- Detectar e registrar eventos (afundamentos, elevações, interrupções)
- Persistir histórico acessível para consulta e análise

### 2.3 Classes de usuários

| Classe | Descrição | Frequência de uso |
|---|---|---|
| Morador/consumidor | Consulta dashboards, recebe alertas | Diária |
| Pesquisador/desenvolvedor | Acesso a dados brutos, calibração | Eventual |
| Avaliador acadêmico | Inspeção de código, documentação | Eventual |

### 2.4 Ambiente operacional

- Rede WiFi residencial 2.4 GHz
- Alimentação: 5 V USB (ATmega+ZMPT), 3.3 V regulado (ESP01), fonte dedicada ESP32
- Temperatura: 0–40 °C (ambiente interno residencial)
- Broker MQTT alcançável na LAN

### 2.5 Restrições de projeto

- Custo total de hardware por ponto de medição ≤ R$ 150 (meta)
- Componentes disponíveis no mercado brasileiro
- Sem uso fiscal/metrológico (afirmado explicitamente ao usuário)

### 2.6 Suposições e dependências

- Disponibilidade contínua de WiFi residencial
- Broker MQTT existente e acessível
- Fornecimento elétrico estável para alimentação dos microcontroladores
  (o sistema não monitora a própria falta de energia em si — limitação a documentar)

---

## 3. Requisitos funcionais

Identificadores seguem o padrão `RF-NNN`. Prioridade: **M** (must), **S** (should), **C** (could), **W** (won't — nesta versão).

### 3.1 Sensor (ATmega328P)

| ID | Requisito | Prio |
|---|---|---|
| RF-001 | O sensor deve amostrar a tensão AC via ADC em A0 a uma taxa suficiente para cobrir ao menos 2 ciclos de 60 Hz por janela de cálculo. | M |
| RF-002 | O sensor deve calcular o valor RMS sobre cada janela de amostragem. | M |
| RF-003 | O sensor deve aplicar fator de calibração configurável em tempo de compilação. | M |
| RF-004 | O sensor deve transmitir cada medição via UART em frame binário com CRC16. | M |
| RF-005 | Cada frame deve conter número de sequência incremental para detecção de perdas no receptor. | M |
| RF-006 | O sensor deve sinalizar condição de queda (V_RMS abaixo de limiar configurável) via flag no frame. | M |
| RF-007 | O sensor deve permitir reconfiguração do fator de calibração sem recompilação. | C |

### 3.2 Gateway (ESP01)

| ID | Requisito | Prio |
|---|---|---|
| RF-010 | O gateway deve consumir frames binários da UART, validar CRC16 e descartar frames inválidos. | M |
| RF-011 | O gateway deve publicar medições válidas em tópico MQTT `medtens/sensor/<id>/vrms`. | M |
| RF-012 | O gateway deve contabilizar frames recebidos, aceitos e descartados e publicar diagnóstico periódico em `medtens/sensor/<id>/diag`. | S |
| RF-013 | O gateway deve reconectar automaticamente ao broker em caso de queda. | M |
| RF-014 | O gateway deve publicar mensagem *Last Will and Testament* para sinalizar indisponibilidade. | S |

### 3.3 Coletor (ESP32)

| ID | Requisito | Prio |
|---|---|---|
| RF-020 | O coletor deve assinar os tópicos MQTT das medições. | M |
| RF-021 | O coletor deve manter janela deslizante de medições para cálculo de indicadores. | M |
| RF-022 | O coletor deve classificar cada medição em faixas PRODIST (adequada/precária/crítica). | M |
| RF-023 | O coletor deve calcular DRP e DRC em base horária. | M |
| RF-024 | O coletor deve detectar afundamentos e elevações conforme IEEE 1159. | S |
| RF-025 | O coletor deve persistir medições brutas em tabela `medicoes_brutas`. | M |
| RF-026 | O coletor deve persistir indicadores agregados em tabela `indices_qualidade`. | M |
| RF-027 | O coletor deve bufferizar localmente medições quando MySQL estiver indisponível. | S |

### 3.4 Servidor (MySQL)

| ID | Requisito | Prio |
|---|---|---|
| RF-030 | O schema deve armazenar medições brutas com timestamp em resolução ≤ 1 s. | M |
| RF-031 | O schema deve permitir consulta de indicadores por intervalo de tempo arbitrário. | M |
| RF-032 | O sistema deve permitir export de dados em CSV para análise externa. | S |

---

## 4. Requisitos não-funcionais

### 4.1 Desempenho

| ID | Requisito |
|---|---|
| RNF-001 | Latência entre amostragem e publicação MQTT ≤ 1 s em 95 % dos casos. |
| RNF-002 | Taxa de emissão de medições ≥ 1 Hz. |

### 4.2 Confiabilidade

| ID | Requisito |
|---|---|
| RNF-010 | Disponibilidade do pipeline ≥ 99 % em janela de 24 h (meta; a medir). |
| RNF-011 | Taxa de frames UART com CRC inválido aceitos: 0 % (requisito forte). |
| RNF-012 | Perda máxima de medições aceitável por reconexão MQTT: ≤ 5 s. |

### 4.3 Precisão (metrológica, não fiscal)

| ID | Requisito |
|---|---|
| RNF-020 | Erro absoluto da medição RMS após calibração: ≤ ±3 % na faixa 90–150 V (meta a validar). |
| RNF-021 | O erro e a incerteza devem ser reportados no documento de validação. |

### 4.4 Manutenibilidade

| ID | Requisito |
|---|---|
| RNF-030 | Código deve seguir padrão formatado via `.clang-format`. |
| RNF-031 | Cobertura de testes unitários ≥ 60 % para parsers e cálculos de indicadores. |
| RNF-032 | Toda decisão arquitetural não óbvia deve ser documentada em ADR. |

### 4.5 Reprodutibilidade

| ID | Requisito |
|---|---|
| RNF-040 | Dependências devem ser pinadas em versões exatas. |
| RNF-041 | README deve conter instruções passo-a-passo de build para cada componente. |
| RNF-042 | Cada release deve receber DOI via Zenodo. |

### 4.6 Segurança

| ID | Requisito |
|---|---|
| RNF-050 | Credenciais MQTT/WiFi não devem ser commitadas; uso de arquivo `.env` / `secrets.h` ignorado pelo git. |
| RNF-051 | Comunicação MQTT na LAN; exposição pública requer TLS (fora de escopo v1). |

### 4.7 Portabilidade

| ID | Requisito |
|---|---|
| RNF-060 | Firmware deve compilar em PlatformIO Core no Linux (Ubuntu-based). |

---

## 5. Restrições de projeto

- Microcontroladores fixados: ATmega328P, ESP8266 (ESP01), ESP32
- Banco de dados fixado: MySQL (compatibilidade com stack Home Assistant existente)
- Idioma da documentação: português do Brasil
- Licenças de software de terceiros: compatíveis com uso aberto

## 6. Rastreabilidade

Cada requisito deste documento deve ser rastreável a:

- uma ou mais **histórias de usuário** (XP backlog)
- uma ou mais **decisões arquiteturais** (ADRs) quando aplicável
- um ou mais **testes** (unitário, integração, ensaio de validação)

Matriz de rastreabilidade a ser mantida em `academica/rastreabilidade.md`
(a criar quando o backlog XP for estruturado).

## 7. Histórico de revisões

| Versão | Data | Autor | Alterações |
|---|---|---|---|
| 0.1 | 2026-04-17 | *a preencher* | Versão inicial (esboço) |
