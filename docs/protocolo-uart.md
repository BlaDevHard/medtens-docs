# Protocolo UART — ATmega ↔ ESP01

**Status**: rascunho (implementação em `medtens-atmega-sensor` planejada para v0.2.0)

## Camada física

- UART 9600 8N1, sem controle de fluxo
- TX do ATmega (5 V) → divisor resistivo 1 kΩ + 2 kΩ → RX do ESP01 (3,3 V)
- Alimentação do ESP01: regulador 3,3 V dedicado (mínimo 300 mA de pico)

## Formato do frame

```
┌──────┬──────┬─────┬──────┬─────────────┬───────────┐
│ SOF1 │ SOF2 │ LEN │ TYPE │   PAYLOAD   │   CRC16   │
│ 0xAA │ 0x55 │  N  │  id  │   N bytes   │  2 bytes  │
└──────┴──────┴─────┴──────┴─────────────┴───────────┘
```

| Campo | Bytes | Descrição |
|---|---|---|
| `SOF1`, `SOF2` | 2 | Marcador de início fixo `0xAA 0x55` |
| `LEN` | 1 | Tamanho do `PAYLOAD` em bytes (0–255) |
| `TYPE` | 1 | Identificador do tipo de payload |
| `PAYLOAD` | `LEN` | Dados (estrutura depende do `TYPE`) |
| `CRC16` | 2 | CRC-CCITT (poly 0x1021, init 0xFFFF), little-endian, sobre `TYPE` + `PAYLOAD` |

## Tipos de payload

### `TYPE = 0x01` — `MEAS_V1` (9 bytes)

```c
struct __attribute__((packed)) {
    uint32_t seq;    // contador sequencial, detecta perda de frames
    float    vrms;   // tensão RMS calibrada em Volts
    uint8_t  flags;  // bit0 = QUEDA (vrms < 10 V), bits restantes reservados
} meas_v1_t;
```

Little-endian nativo do AVR. Total do frame: 2+1+1+9+2 = **15 bytes**.

## Evolução

Novos tipos (ex: `MEAS_V2` com THD e frequência) recebem novo `TYPE` e struct
próprio — parsers antigos ignoram por desconhecer o `TYPE`. Nunca reutilizar
`TYPE` com layout diferente.

## Pseudocódigo do receptor (ESP01)

```
buffer = []
enquanto bytes disponíveis:
    ler byte b
    buffer.append(b)
    se buffer.len >= 5:
        se buffer[0:2] != [0xAA, 0x55]:
            descartar primeiro byte, continuar
        len = buffer[2]
        se buffer.len < 4 + len + 2: continuar  # aguarda mais
        type = buffer[3]
        payload = buffer[4 : 4+len]
        crc_rx = buffer[4+len] | (buffer[4+len+1] << 8)
        crc_calc = crc16_ccitt([type] + payload)
        se crc_rx == crc_calc:
            processar(type, payload)
        buffer = buffer[4+len+2:]
```
