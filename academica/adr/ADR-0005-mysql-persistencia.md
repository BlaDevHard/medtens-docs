# ADR-0005 — MySQL como armazenamento histórico

**Status:** Accepted
**Data:** 2026-04-17
**Decisores:** autor
**Relacionado:** [ADR-0001](ADR-0001-topologia-4-componentes.md) · [ADR-0003](ADR-0003-mqtt-como-barramento.md) · [ADR-0004](ADR-0004-timestamp-hibrido.md) · DAS arc42 § 5.2.4, § 7 · ERS RF-025, RF-026, RF-030, RF-031, RF-032

---

## Contexto

O MedTens precisa armazenar:

- **Medições brutas** de tensão RMS (≥ 1 Hz, múltiplos dias).
- **Índices apurados** (DRP/DRC horários).
- **Eventos** (sag/swell/interruption IEEE 1159).

Volume estimado: 1 amostra/s × 86 400 s/dia ≈ 86 400 linhas/dia na
tabela `medicoes_brutas`. Em um ano: ~31 milhões de linhas. Cada linha
ocupa ~50 bytes → ~1,5 GB/ano.

Requisitos funcionais (ERS § 3.4):
- Consulta por intervalo de tempo arbitrário (RF-031)
- Export CSV (RF-032)
- Resolução de timestamp ≤ 1 s (RF-030)

---

## Decisão

Usar **MySQL 8.0+** já instalado no servidor residencial Mi 8 Lite
(192.168.1.10), reutilizado pela stack Home Assistant.

### Schema inicial

```sql
CREATE TABLE medicoes_brutas (
    id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    sensor_id     VARCHAR(32)   NOT NULL,
    ts_epoch_ms   BIGINT        NOT NULL,
    ts_sensor_ms  BIGINT        NULL,
    ts_servidor   DATETIME(3)   NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    seq           INT UNSIGNED  NOT NULL,
    vrms          DECIMAL(6,2)  NOT NULL,
    flags         TINYINT UNSIGNED NOT NULL DEFAULT 0,
    ntp_synced    TINYINT(1)    NOT NULL DEFAULT 0,
    INDEX idx_sensor_ts (sensor_id, ts_epoch_ms),
    INDEX idx_ts_servidor (ts_servidor)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE indices_qualidade (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    sensor_id   VARCHAR(32)   NOT NULL,
    ts_inicio   DATETIME      NOT NULL,
    ts_fim      DATETIME      NOT NULL,
    drp         DECIMAL(5,2)  NOT NULL,
    drc         DECIMAL(5,2)  NOT NULL,
    n_leituras  INT UNSIGNED  NOT NULL,
    INDEX idx_sensor_periodo (sensor_id, ts_inicio, ts_fim)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE eventos (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    sensor_id   VARCHAR(32)   NOT NULL,
    ts_inicio   DATETIME(3)   NOT NULL,
    ts_fim      DATETIME(3)   NULL,
    tipo        ENUM('sag','swell','interruption','instantaneous','momentary','temporary') NOT NULL,
    severidade  ENUM('leve','moderado','severo','critico') NULL,
    v_min       DECIMAL(6,2)  NULL,
    v_max       DECIMAL(6,2)  NULL,
    dur_ms      INT UNSIGNED  NULL,
    INDEX idx_sensor_ts (sensor_id, ts_inicio)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Política de retenção

- `medicoes_brutas`: 90 dias, agregação para média horária após esse
  prazo em uma tabela `medicoes_horarias` (implementação futura).
- `indices_qualidade`: retenção indefinida (volume desprezível: ~8 700
  linhas/ano).
- `eventos`: retenção indefinida.

### Credenciais e segurança

- Usuário MySQL dedicado `medtens_writer` com apenas `INSERT` em
  `medicoes_brutas`, `indices_qualidade`, `eventos`.
- Usuário `medtens_reader` com `SELECT` para dashboards e export.
- Credenciais em `secrets.h` / `.env` (RNF-050), **não commitadas**.
- Conexão sem TLS (LAN-only) na v1 (RNF-051).

---

## Consequências

### Positivas

- **Reaproveitamento** — MySQL já está instalado, administrado e
  versionado junto com o Home Assistant; sem custo operacional novo.
- **Ferramental maduro** — `mysql` CLI, `mysqldump`, dashboards
  Grafana/Metabase, integração Home Assistant via `recorder: db_url`.
- **Transações** — InnoDB garante durabilidade mesmo em queda do ESP32
  mid-INSERT.
- **Export CSV trivial** — `SELECT ... INTO OUTFILE` ou
  `mysql -e "..." > arquivo.csv`.
- **Consultas com JOIN** — facilita correlação medição × evento ×
  índice no documento de Análise dos Dados do PPI.

### Negativas

- **Dependência de rede LAN** — MySQL indisponível → coletor precisa
  bufferizar em SPIFFS (RF-027, já previsto).
- **Ponto único de falha** — servidor Mi 8 Lite desliga → dados perdidos
  enquanto o SPIFFS não satura (capacidade típica 1–3 MB ≈ ~20 k
  medições ≈ 6 h a 1 Hz).
- **Schema migra no tempo** — versionar em `medtens-server/migrations/`
  com Flyway ou arquivos `.sql` numerados.

### Neutras

- Schema acima é o estado inicial; evolução documentada em ADRs
  futuros quando surgirem mudanças não triviais (nova coluna, nova
  tabela, mudança de tipo).

---

## Alternativas rejeitadas

| Alternativa | Motivo |
|---|---|
| **SQLite em arquivo** | Não suporta múltiplos escritores; o coletor fica acoplado ao servidor por NFS ou precisa ser o próprio servidor |
| **InfluxDB** | Adequado para time-series mas exige administração separada; Home Assistant já tem MySQL configurado |
| **Postgres** | Tecnicamente superior mas não está instalado no servidor; adicionaria componente novo |
| **Arquivo CSV append-only** | Sem transação, sem índice, sem consulta por intervalo eficiente |
| **TimescaleDB** | Dependeria de Postgres (ver acima) |

---

## Referências

- Oracle (2024) — *MySQL 8.0 Reference Manual* (InnoDB, DATETIME fractional seconds)
- Home Assistant docs — *Recorder integration* (padrão de uso MySQL)
- ANSI/ISO SQL:2016 — para escopo de tipos temporais
