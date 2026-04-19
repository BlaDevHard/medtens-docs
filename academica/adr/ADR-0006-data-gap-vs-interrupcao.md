# ADR-0006 — Distinção entre interrupção medida e *data gap*

**Status:** Proposed
**Data:** 2026-04-18
**Decisores:** autor
**Relacionado:**
[ADR-0001](ADR-0001-topologia-4-componentes.md) ·
[ADR-0003](ADR-0003-mqtt-como-barramento.md) ·
[ADR-0004](ADR-0004-timestamp-hibrido.md) ·
[ADR-0005](ADR-0005-mysql-persistencia.md) ·
ERS RF-027, RNF-010 (a revisar) ·
Plano de Validação § V&V-3

---

## Contexto

O firmware ATmega328P de `medtens-atmega-sensor` v0.1.0 registra "queda"
sempre que `Vrms < 10 V`, e o gadget `medtens-gadget` mostra "sem dados"
quando a UART fica silenciosa por mais de 5 s. As duas condições são
tratadas no mesmo canal visual, mas **são fenômenos distintos** para
efeito regulatório.

Ao avançar para a cadeia completa (Arduino → ESP01 → ESP32 → Redmi
MySQL), a confusão entre "não chegou dado" e "o fornecimento caiu"
invalida os indicadores apurados:

- **Se o ESP01 perder Wi-Fi por 30 s**, o coletor pode contar como VMT
  inexistente (falso positivo);
- **Se o Arduino sofrer reset por watchdog**, nenhum dado chega — pode
  ser registrado como interrupção (falso positivo);
- **Se a rede AC cair e o sensor estiver em bateria**, o sensor **continua
  medindo** Vrms ≈ 0 — interrupção verdadeira, mas **só é confirmável
  se houver um sinal digital paralelo** atestando que o medidor está
  funcionando e apenas a rede caiu.

As normas de qualidade de energia tratam disto explicitamente.

### PRODIST Módulo 8 — Revisão 12 (ANEEL)

Define limiar fixo de **10% da tensão nominal** (127 V → 12,7 V) e três
categorias de duração:

| Fenômeno | Duração | Indicador regulatório |
|---|---|---|
| VMT — Variação Momentânea de Tensão | ≥ 1 ciclo e ≤ 3 s | FMIK / DMIK |
| VTT — Variação Temporária de Tensão | > 3 s e ≤ 3 min | FMIK / DMIK |
| ILD — Interrupção de Longa Duração | > 3 min | DEC / FEC / DIC / FIC |

Todas exigem **tensão medida abaixo do limiar**. Tempo em que o medidor
esteve inoperante (bateria, reset, falha própria) **não conta** como
interrupção do fornecimento.

### IEEE Std 1159-2019 — Tabela 4

Categorias análogas (momentary, temporary, sustained interruption),
todas com critério de **amplitude + duração medidas**. A norma nomeia
"data gap" qualquer intervalo em que o medidor não forneceu amostras e
exige **relato separado** dos indicadores de qualidade.

### IEC 61000-4-30:2025

- **§5.5** define que interrupção começa quando a tensão eficaz de
  meio-ciclo (Ueff(1/2)) cai abaixo do limiar em todos os canais
  simultaneamente e termina quando sobe acima do limiar + histerese.
- **§4.7** (*Flagged data*) exige: quando o medidor perde amostras por
  motivo interno, **todo o intervalo afetado recebe flag** e é
  **excluído do cálculo dos indicadores**. A flag viaja com o dado
  até a apuração.

---

## Decisão

O MedTens adota três estados mutuamente exclusivos para cada instante
do ensaio, classificados por **cada nó sensor** independentemente:

| Estado | Critério de detecção | Persistência |
|---|---|---|
| **Tensão válida** | `Vrms ≥ 0,1 · U_nom` **E** dado íntegro (CRC válido, sequência contígua) | `medicoes_brutas` com `flags = 0` |
| **Interrupção medida** | `Vrms < 0,1 · U_nom` **E** `ac_detect = 0` (opto confirma ausência da rede) **E** medidor operante (sequência CRC OK) | `eventos.tipo ∈ {instantaneous, momentary, temporary, interruption}` conforme duração |
| **Data gap** | Ausência de heartbeat do nó por > `T_heartbeat` (5 s padrão) **OU** perda de frames CRC consecutiva acima do limiar (> 3 frames) | Nova tabela `data_gaps` (ver § Schema abaixo) |

### Campos obrigatórios no payload MQTT por nó

```json
{
  "sensor_id": "antonina-01",
  "ts": "2026-04-18T18:23:45.123Z",
  "seq": 12345,
  "vrms": 127.4,
  "ac_detect": 1,
  "on_battery": false,
  "uptime_ms": 3600000,
  "flags": 0
}
```

- `ac_detect`: GPIO isolado por optoacoplador (4N25/PC817) indicando
  presença da rede AC. Fonte de verdade **independente** da medição
  analógica. `1` = rede presente; `0` = ausente.
- `on_battery`: `true` quando o nó detecta que está sendo alimentado
  pelo power bank (circuito separado sensoreando a entrada de carga).
- `flags`: campo de 8 bits para bandeiras IEC §4.7 (bit 0 = dado
  estimado, bit 1 = pós-reset, bit 2 = fora de sincronia NTP, etc.).

Com `vrms`, `ac_detect` e `on_battery` juntos, qualquer camada à frente
consegue classificar corretamente.

### Heartbeat por camada

Cada camada publica em `medtens/<camada>/<id>/status` a cada 5 s. A
**ausência do heartbeat** é o sinal de data gap daquela camada. A
**presença do heartbeat sem medição** é uma condição excepcional (bug
do firmware) e também abre `data_gap` com log de alerta.

### Schema — complemento ao ADR-0005

Acrescentar a tabela `data_gaps` e um campo `origem` em `eventos`:

```sql
CREATE TABLE data_gaps (
    id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    sensor_id     VARCHAR(32)   NOT NULL,
    camada        ENUM('sensor','gateway','coletor','servidor') NOT NULL,
    ts_inicio     DATETIME(3)   NOT NULL,
    ts_fim        DATETIME(3)   NULL,
    dur_ms        INT UNSIGNED  NULL,
    motivo        VARCHAR(128)  NULL,   -- texto livre ou código
    INDEX idx_sensor_periodo (sensor_id, ts_inicio, ts_fim)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

ALTER TABLE eventos
    ADD COLUMN confirmado_por_ac_detect TINYINT(1) NOT NULL DEFAULT 0
        AFTER dur_ms;
```

`confirmado_por_ac_detect = 1` significa que o sinal digital AC
confirmou a queda no mesmo intervalo. Eventos sem essa confirmação
entram no banco mas devem ser **considerados suspeitos** pela análise
dos dados.

### Cálculo de DRP/DRC com flagging

Regras de apuração no ESP32 coletor (etapa E4) e/ou em *view* SQL no
Redmi:

1. Para cada janela de 10 min do PRODIST:
   1.1. Somar duração de `data_gaps` na janela.
   1.2. Se `(duração_data_gap / 600_s) > 0,01` → **marcar janela
        inválida**; não entra no cálculo de DRP/DRC.
   1.3. Caso contrário, denominador da janela = `600 s − duração_data_gap`;
        numerador = soma de duração de eventos de queda **medidos**
        (com ou sem confirmação AC, conforme política de V&V-3).
2. DRP e DRC apurados somente sobre janelas válidas.
3. Relatório final lista separadamente:
   - **Cobertura de medição** (fração de tempo com `estado=tensão_válida`)
   - **Tempo de interrupção medida** (soma de durações dos eventos)
   - **Tempo em data gap** (soma de durações dos gaps)

### Revisão do RNF-010

Redação proposta para substituir o RNF-010 atual na ERS:

> **RNF-010 (revisado)**. Em cada janela de 24 h das 168 h do ensaio de
> V&V-3, a **cobertura de medição** (amostras válidas + interrupções
> medidas confirmadas pelo sinal digital AC) deve ser ≥ 99%. **Data
> gaps** (§4.7 IEC 61000-4-30) **não contam** como amostra válida nem
> como interrupção; janelas com data gap acumulado > 1% do intervalo
> são **invalidadas** e o ensaio é estendido até totalizar 7 janelas
> de 24 h válidas.

---

## Consequências

### Positivas

- **Indicadores apurados defensáveis perante banca** — DRP e DRC
  calculados só sobre janelas válidas, sem inflar ou esconder falhas
  próprias do medidor.
- **Conformidade com §4.7 da IEC 61000-4-30** — adequa o projeto a uma
  norma internacional ainda que não atinja classe A.
- **Narrativa honesta para a banca** — a seção "Limitações" do
  Slide 19 do PPI ganha conteúdo concreto: "MedTens diferencia
  interrupção de data gap, diferença que medidores classe S custam
  R$ 15 000 para fazer".
- **Diagnóstico operacional** — saber que o sensor teve 40 min de data
  gap numa noite é útil para saneamento de hardware, impossível no
  modelo anterior onde tudo virava "queda".

### Negativas

- **Aumenta complexidade do firmware ESP01 (gateway)** — precisa
  amostrar GPIO do opto, calcular uptime, montar payload estruturado
  (JSON ou equivalente binário).
- **Aumenta complexidade do coletor ESP32** — passa a manter máquina
  de estados por nó com três estados e timers de heartbeat.
- **Nova tabela e nova coluna no MySQL** — migração do schema
  (versionar em `medtens-server/migrations/002_data_gaps.sql`).
- **Hardware adicional exigido** — optoacoplador AC-detect passa de
  "melhoria futura" para requisito funcional; sem ele o campo
  `ac_detect` é sempre 0 e eventos nunca são confirmados.

### Neutras

- Custo de material do opto (~R$ 2–5) é desprezível frente ao limite
  de R$ 500 por ponto (hipótese da pergunta de pesquisa).
- Logicamente compatível com a proposta de **always-on com
  contabilização de downtime** (decisão de 2026-04-18) — aqui apenas se
  define como contabilizar *o que é o quê*.

---

## Alternativas rejeitadas

| Alternativa | Motivo da rejeição |
|---|---|
| Manter "queda" como rótulo único para tudo (ausência de dado + Vrms baixo) | Invalida a apuração de DRP/DRC perante IEC §4.7 e PRODIST Mód. 8; induz banca a questionar validade do método. |
| Inferir data gap apenas por timeout do último `vrms` recebido (sem heartbeat separado) | Impossível diferenciar "firmware travou" de "sensor desligou por bateria"; perde-se granularidade para diagnóstico. |
| Registrar data gap só nos *logs* (não no banco) | Dashboards e relatórios perdem a informação; banca pode acusar omissão seletiva. |
| Usar apenas `ac_detect` (ignorar `vrms`) para detectar interrupção | Perde a informação de amplitude (não diferencia sag profundo de interrupção); contraria IEEE 1159. |

---

## Referências

- ANEEL. **PRODIST Módulo 8 — Revisão 12**. REN nº 956/2021 (alterada
  pela REN nº 1148/2026).
- IEEE. **Std 1159-2019** — *Monitoring Electric Power Quality*, Tabela 4.
- IEC. **61000-4-30:2025** — *Electromagnetic compatibility (EMC) —
  Part 4-30: Testing and measurement techniques — Power quality
  measurement methods*, §4.7 (Flagged data) e §5.5 (Voltage
  interruptions).
- Bollen, M. H. J. (2000). **Understanding Power Quality Problems —
  Voltage Sags and Interruptions**. IEEE Press.
