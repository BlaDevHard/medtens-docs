# Architecture Decision Records — MedTens

ADRs documentam decisões arquiteturais não óbvias, no formato proposto
por Michael Nygard (2011): **Contexto → Decisão → Consequências**. Cada
decisão recebe um número sequencial e um status.

## Índice

| ADR | Título | Status |
|---|---|---|
| [ADR-0001](ADR-0001-topologia-4-componentes.md) | Topologia em quatro componentes distribuídos | Accepted |
| [ADR-0002](ADR-0002-protocolo-uart-binario-crc16.md) | Protocolo UART binário com CRC16 entre ATmega e ESP01 | Accepted |
| [ADR-0003](ADR-0003-mqtt-como-barramento.md) | MQTT como barramento de integração | Accepted |
| [ADR-0004](ADR-0004-timestamp-hibrido.md) | Timestamp híbrido (relativo no sensor, absoluto no gateway) | Accepted |
| [ADR-0005](ADR-0005-mysql-persistencia.md) | MySQL como armazenamento histórico | Accepted |

## Estados possíveis

- **Proposed** — em discussão
- **Accepted** — decisão vigente
- **Deprecated** — ainda se aplica ao código existente mas desencorajada
- **Superseded** — substituída por outro ADR (referência obrigatória ao sucessor)

## Como criar um novo ADR

1. Próximo número sequencial (ex.: `ADR-0006-slug-curto.md`)
2. Nunca reescrever um ADR aceito — criar novo ADR que o supersede
3. Referenciar ADRs relacionados na seção de cabeçalho
4. Incluir ao menos uma alternativa rejeitada e seu motivo

## Referência

- Nygard, M. T. (2011). *Documenting Architecture Decisions*. Disponível em:
  https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
