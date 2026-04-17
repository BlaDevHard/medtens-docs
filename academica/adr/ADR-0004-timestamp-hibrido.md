# ADR-0004 — Timestamp híbrido: relativo no sensor, absoluto no gateway

**Status:** Accepted
**Data:** 2026-04-17
**Decisores:** autor
**Relacionado:** [ADR-0001](ADR-0001-topologia-4-componentes.md) · [ADR-0002](ADR-0002-protocolo-uart-binario-crc16.md) · DAS arc42 § 4 (princípio 4), § 6.2 · ERS RF-030

---

## Contexto

As medições e eventos precisam ser carimbados com timestamp absoluto
(*epoch ms*) para alimentar as apurações horárias de DRP/DRC (PRODIST 8)
e para correlacionar com ocorrências registradas pelo consumidor
(fotos, chamados à Copel, etc.).

O ATmega328P **não tem RTC nem rede**: sozinho, só sabe medir tempo
relativo ao *boot* via contador de ciclos ou Timer0. Três abordagens
foram consideradas:

1. **NTP no ESP** (gateway e/ou coletor) + timestamp atribuído no momento
   da recepção.
2. **RTC de hardware (DS3231) no ATmega** via I2C — sensor carimba direto.
3. **Timestamp do servidor** — MySQL grava `CURRENT_TIMESTAMP` no INSERT.

Cada uma tem *trade-offs* distintos em precisão, custo, complexidade
e resiliência a falhas de rede.

---

## Decisão

Adotar abordagem **(1) primária + (3) complementar**:

### Primária — NTP no ESP01 e no ESP32

1. O ATmega envia em cada frame binário (ADR-0002) um campo `t_ms`
   *uint32 LE* = tempo desde o boot em milissegundos (Timer0 CTC).
2. O ESP01 (gateway) sincroniza seu relógio com `pool.ntp.br` (`pool.ntp.org`
   como fallback) no *boot* e a cada 6 h.
3. O ESP01 calcula, no momento da publicação MQTT:
   `ts_epoch_ms = now_epoch_ms − (t_ms_now_sensor − t_ms_frame)`
   onde `t_ms_now_sensor` é o `t_ms` do último frame recebido.
4. O ESP32 (coletor) também mantém NTP próprio para timestamp dos
   índices DRP/DRC que ele apura (esses não dependem do sensor).

### Complementar — `CURRENT_TIMESTAMP` no MySQL

Cada tabela tem uma coluna `ts_servidor DATETIME(3) DEFAULT
CURRENT_TIMESTAMP(3)` gravada pelo próprio servidor no INSERT. Serve
como:
- *sanity check* contra `ts_epoch_ms` calculado pelo gateway
- fallback se o NTP do ESP falhar no *boot*
- referência única em consultas SQL ad-hoc

### Tratamento de eventos de queda

Para eventos `QUEDA_FIM` com `dur_ms` reportado pelo sensor, o gateway
grafa `ts_inicio_epoch = ts_evento_atual − dur_ms`. Isso preserva a
duração correta mesmo se o gateway tiver ficado sem energia junto com a
rede (ver DAS § 6.2 cenário 2, e risco documentado na DAS § 11).

---

## Consequências

### Positivas

- **Custo zero em hardware** — reutiliza WiFi/NTP já presentes nos ESPs.
- **Precisão adequada** — NTP via LAN tem erro típico < 10 ms; PRODIST 8
  opera em base horária, 1000× menos exigente.
- **Dupla verificação** — `ts_epoch_ms` (cliente) vs. `ts_servidor` (banco)
  permite detectar drift do relógio do ESP sem esforço extra.
- **Resiliência a `boot` sem NTP** — se o NTP não sincronizar, o MySQL
  ainda carimba tudo com `ts_servidor`. O coletor marca essas linhas com
  flag `ntp_synced=0` para análise posterior.
- **Não requer firmware novo no ATmega** — só acrescenta `t_ms` ao frame
  binário, que já está previsto em ADR-0002.
- **Responde à pergunta "MedTens pode usar o time do sistema?"** —
  sim, através desta arquitetura.

### Negativas

- **Perda de precisão durante queda longa do gateway** — se o gateway
  estiver sem energia junto com a rede, eventos anteriores ao retorno
  ficam sem timestamp absoluto preciso (depende da fidelidade do Timer0
  do ATmega, que mantém contagem enquanto tiver alimentação própria — o
  que não é o caso típico). Mitigação: reportar `dur_ms` e `ts_fim` e
  calcular `ts_inicio` por subtração.
- **Dependência de WiFi para sincronização inicial** — sem internet por
  muito tempo, o ESP perde o *epoch*. Mitigação: usar `ts_servidor` do
  MySQL como fonte de verdade quando `ntp_synced=0`.
- **Drift do cristal do ATmega** — em 24 h, um cristal de ±30 ppm
  acumula ~2,6 s. Aceitável para PRODIST horário; não seria para
  *time-of-day metering*.

### Neutras

- Abordagem (2) (DS3231) fica documentada como *fallback off-grid*
  para uma futura variante v2 do MedTens que precise funcionar sem WiFi.

---

## Alternativas rejeitadas

| Alternativa | Motivo |
|---|---|
| **DS3231 RTC no ATmega** | Custo (~R$ 15) + 2 pinos I2C + biblioteca; desproporcional dado que o ESP01 já tem NTP de graça |
| **Só timestamp do servidor MySQL** | Mede quando o INSERT chegou, não quando a tensão foi amostrada — pode ter atraso de segundos em caso de buffer MQTT |
| **NTP direto no ATmega** | ATmega não tem rede |
| **GPS com PPS** | *Overkill* absoluto; precisão microssegundo irrelevante aqui |

---

## Referências

- Mills (2010) — *Computer Network Time Synchronization: The Network Time Protocol on Earth and in Space*
- ANEEL (2021) — PRODIST Módulo 8, Rev. 12 (janela horária para DRP/DRC)
- IEEE Std 1159-2019 — § 5 (requisitos de timestamp para eventos de qualidade)
