# ADR-0001 — Topologia em quatro componentes distribuídos

**Status:** Accepted
**Data:** 2026-04-17
**Decisores:** autor (discente ADS/Sumaré)
**Relacionado:** DAS arc42 §§ 3, 4, 5 · ERS § 1.2

---

## Contexto

O MedTens precisa **adquirir** a tensão AC de um ponto residencial em
Antonina-PR (área de concessão da Copel), **calcular** indicadores
PRODIST (DRP/DRC), **detectar** eventos conforme IEEE 1159 e **persistir**
histórico consultável. As restrições fixadas pelo projeto (ver ERS § 5 e
DAS § 2.1) são: ATmega328P, ESP8266 (ESP01), ESP32 e MySQL compatível com
a stack Home Assistant já existente no servidor residencial (192.168.1.10).

Três topologias foram consideradas:

1. **Monolítica em ESP32** — um único microcontrolador amostra, processa,
   publica e persiste. Menor número de peças; maior risco de
   indisponibilidade (uma falha derruba tudo) e menor separação de
   responsabilidades.
2. **Dois componentes (sensor + servidor)** — ATmega envia direto ao
   servidor via módulo WiFi acoplado. Exige ESP no próprio ponto de
   medição, duplicando *firmware* e agrupando ruído analógico + WiFi.
3. **Quatro componentes em *pipeline*** — cada hardware com uma única
   responsabilidade (aquisição, transporte, agregação, persistência).
   Mais peças, mas cada uma é simples, testável e substituível.

---

## Decisão

Adotar a topologia (3), com quatro componentes em *pipeline*:

```
Sensor (ATmega328P) ── UART ──► Gateway (ESP01) ── MQTT ──► Coletor (ESP32) ── SQL ──► Servidor (MySQL)
```

Cada componente tem uma função única:

| Componente | Hardware | Responsabilidade |
|---|---|---|
| Sensor | Arduino Uno + ZMPT101B | Amostra ADC e calcula RMS |
| Gateway | ESP01 3,3 V | Ponte UART→MQTT com CRC e LWT |
| Coletor | ESP32 | Janela deslizante, DRP/DRC, detecção IEEE 1159 |
| Servidor | MySQL no Mi 8 Lite | Histórico e consulta |

---

## Consequências

### Positivas

- **Testabilidade isolada** — cada *firmware* pode ser validado em bancada
  com *mocks* dos vizinhos (injetar frames UART no gateway, injetar MQTT
  no coletor, `mysql --execute` no servidor).
- **Evolução independente** — trocar o sensor (ex.: ZMPT → transformador
  de potencial calibrado) não afeta gateway/coletor/servidor se o protocolo
  UART for preservado.
- **Isolamento de falhas** — queda do MQTT não derruba aquisição; queda do
  MySQL não derruba publicação.
- **Aderência ao PPI Sumaré** — demonstra aplicação explícita do
  *Single Responsibility Principle* em arquitetura embarcada, útil para
  argumentação na etapa de Metodologia.

### Negativas

- **Mais pontos de falha individuais** — 4 dispositivos, 4 alimentações,
  4 *firmwares* para manter.
- **Latência acumulada** — cada hop adiciona ~100–300 ms.
- **Custo inicial maior** — 3 microcontroladores + servidor versus 1.
- **Timestamp distribuído** — abordado no [ADR-0004](ADR-0004-timestamp-hibrido.md).

### Neutras

- Exige protocolo explícito entre sensor e gateway (ver
  [ADR-0002](ADR-0002-protocolo-uart-binario-crc16.md)).
- Exige *cola assíncrona* entre gateway e coletor (ver
  [ADR-0003](ADR-0003-mqtt-como-barramento.md)).

---

## Alternativas rejeitadas

| Alternativa | Motivo de rejeição |
|---|---|
| Monolítica ESP32 | Maior risco de indisponibilidade; mistura aquisição analógica com rádio WiFi; dificulta calibração isolada |
| Dois componentes (ESP32 direto ao MySQL) | Duplica firmware no ponto de medição; mistura ruído analógico com WiFi; menos didático para o trabalho |

---

## Referências

- Bass, Clements e Kazman (2021) — *Software Architecture in Practice*, cap. 4 (qualidade como direcionador arquitetural)
- Starke (2024) — *arc42 Template*, seção 4 (Solution Strategy)
