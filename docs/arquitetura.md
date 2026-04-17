# Arquitetura — MedTens

## Visão geral

Pipeline de 4 estágios, cada um em hardware/repositório separado para
isolamento de responsabilidades e facilidade de manutenção/citação.

## Estágios

### 1. Sensor — ATmega328P

- **Hardware**: Arduino Uno, ZMPT101B conectado em A0
- **Função**: amostragem do ADC, cálculo de V_RMS por janela de ~2 ciclos de 60 Hz
- **Saída**: UART 9600 8N1, frame binário (ver [protocolo-uart.md](protocolo-uart.md))
- **Repo**: `medtens-atmega-sensor`

### 2. Gateway — ESP01 (ESP8266)

- **Função**: consumir UART, validar CRC, republicar em MQTT
- **Justificativa**: ATmega não tem WiFi; ESP01 é barato e resolve só esta ponta
- **Saída**: tópicos MQTT no broker local (Mi 8 Lite, 192.168.1.10)
- **Repo**: `medtens-esp01-gateway`

### 3. Coletor — ESP32

- **Função**: assinar MQTT, calcular índices de qualidade, persistir em MySQL
- **Justificativa**: CPU/RAM suficientes para janela deslizante, FFT futura, etc.
- **Saída**: inserts no MySQL; eventualmente dashboards
- **Repo**: `medtens-esp32-collector`

### 4. Servidor — MySQL

- **Função**: histórico permanente, consultas de análise, export para notebooks
- **Repo**: `medtens-server`

## Por que essa separação?

| Alternativa | Problema |
|---|---|
| Tudo no ESP32 + ZMPT direto | ESP32 tem ADC ruidoso/não linear; ATmega com ref AVcc dá RMS mais limpo |
| ATmega direto no WiFi | Não tem WiFi |
| ESP01 fazendo tudo | RAM e flash muito limitados para índices de qualidade |
| Gravar direto no ESP32 sem broker | Perde histórico se ESP32 reinicia; broker desacopla |

## Fluxo de dados (exemplo: uma medição)

1. ATmega faz 3000 leituras ADC em A0 (~30 ms)
2. Calcula RMS, aplica fator de calibração
3. Monta frame binário `[SOF|LEN|TYPE|seq|vrms|flags|CRC16]` e envia por UART
4. ESP01 lê frame, valida CRC, converte para JSON e publica em `medtens/sensor/1/vrms`
5. ESP32 recebe, atualiza janela deslizante, calcula métricas derivadas
6. A cada N medições ou M segundos, ESP32 insere linha em `medicoes_brutas` e/ou `indices_qualidade`
