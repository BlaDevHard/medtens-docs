# ADR-0003 — MQTT como barramento de integração

**Status:** Accepted
**Data:** 2026-04-17
**Decisores:** autor
**Relacionado:** [ADR-0001](ADR-0001-topologia-4-componentes.md) · DAS arc42 § 4, § 6.3 · ERS RF-011, RF-013, RF-014, RF-020, RNF-012

---

## Contexto

Entre o gateway (ESP01) e o coletor (ESP32) precisa haver um mecanismo
de transporte **assíncrono** que:

- Desacople o produtor (gateway) do consumidor (coletor) — cada um pode
  reiniciar sem derrubar o outro.
- Permita **múltiplos consumidores** futuros (Home Assistant, gadget
  desktop na v0.2.0, dashboards *ad hoc*).
- Funcione em rede LAN residencial com latência típica < 10 ms.
- Tenha *broker* já existente no ambiente (Mosquitto no Mi 8 Lite em
  192.168.1.10, usado pelo Home Assistant).
- Seja suportado nativamente no ecossistema Arduino/ESP
  (bibliotecas maduras: `PubSubClient`, `AsyncMqttClient`).

---

## Decisão

Usar **MQTT 3.1.1** sobre TCP/IP como barramento, reutilizando o *broker*
Mosquitto já instalado no servidor residencial (192.168.1.10:1883).

### Estrutura de tópicos

```
medtens/sensor/<sensor_id>/vrms      ← medições brutas (retain=false, QoS 0)
medtens/sensor/<sensor_id>/diag      ← estatísticas do gateway (retain=true, QoS 0)
medtens/sensor/<sensor_id>/status    ← LWT: "online" / "offline" (retain=true, QoS 1)
medtens/sensor/<sensor_id>/event     ← eventos (sag/swell/queda) (retain=false, QoS 1)
medtens/coletor/<coletor_id>/status  ← LWT do coletor (retain=true, QoS 1)
medtens/coletor/<coletor_id>/indice  ← DRP/DRC apurado (retain=false, QoS 1)
```

### Payloads

JSON compacto, sem *pretty-printing*. Exemplo para `vrms`:

```json
{"seq":1042,"vrms":127.43,"flags":0,"ts":1744930123456}
```

### QoS e retenção

- **QoS 0** para medições contínuas — duplicação é aceitável, perda
  ocasional também (atenuada pelo `seq` e pela janela deslizante).
- **QoS 1** para eventos — não podem ser perdidos; o `seq` elimina
  duplicação.
- **Retain** apenas em status e diagnóstico — novos consumidores já
  ingressam com o estado corrente.

### Last Will and Testament

Cada cliente MQTT configura LWT em seu tópico `status` com payload
`"offline"` e `retain=true`. Ao conectar, publica `"online"`. O coletor
monitora `medtens/sensor/+/status` para saber quem está vivo.

---

## Consequências

### Positivas

- **Desacoplamento temporal** — se o coletor reiniciar, o gateway
  continua publicando; quando o coletor volta, consome do *broker* (com
  *buffer* local do ESP32 em SPIFFS para mensagens sem retain — ver
  DAS § 6.3).
- **Múltiplos consumidores** sem mudança no publicador — o gadget
  desktop v0.2.0 assinará os mesmos tópicos, sem acoplar ao gateway.
- **Observabilidade fácil** — `mosquitto_sub -v -t 'medtens/#'` mostra
  tudo que trafega, útil para depuração em campo.
- **Integração com Home Assistant** — dashboards PQ (via `mqtt sensor`
  platform) praticamente de graça.

### Negativas

- **Dependência de disponibilidade LAN** — se o broker cair, o pipeline
  para. Mitigação: buffer local em SPIFFS no gateway (v0.3.0) e no
  coletor (v0.2.0, já previsto em RF-027).
- **Sem autenticação inicial** — Mosquitto residencial roda sem TLS e
  (hoje) sem usuário/senha. Escopo delimitado a LAN (RNF-051). Exposição
  pública requer TLS + ACL, fora do escopo v1.
- **Overhead de JSON** vs. binário — ~80 bytes/mensagem vs. ~11 no UART.
  Aceitável na LAN WiFi (banda sobrando).

### Neutras

- Topic layout segue convenção `<projeto>/<papel>/<id>/<recurso>` — compatível
  com padrões observados no ecossistema Home Assistant.

---

## Alternativas rejeitadas

| Alternativa | Motivo |
|---|---|
| HTTP REST direto do ESP01 para coletor | Síncrono, acopla ciclos; ESP01 não escala múltiplos consumidores |
| WebSocket | Persistente mas 1:1; não atende múltiplos consumidores sem hub dedicado |
| CoAP | Desenhado para UDP constrained; ganha pouco em LAN WiFi e perde ferramental |
| Kafka / NATS | *Overkill* para residencial; exige infraestrutura (JVM ou Go) no servidor |
| gRPC streaming | Biblioteca pesada para ESP01 (512 KB flash) |

---

## Referências

- OASIS (2014) — *MQTT Version 3.1.1* (padrão)
- Light (2017) — *Mosquitto: server and client implementation of the MQTT protocol*
- Home Assistant docs — *MQTT integration* (padrão de tópicos observado)
