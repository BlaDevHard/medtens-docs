# ADR-0002 — Protocolo UART binário com CRC16 entre ATmega e ESP01

**Status:** Accepted
**Data:** 2026-04-17
**Decisores:** autor
**Relacionado:** [ADR-0001](ADR-0001-topologia-4-componentes.md) · DAS arc42 § 3.2, § 5.2 · ERS RF-004, RF-005, RF-010, RNF-011

---

## Contexto

O sensor (ATmega328P @ 5 V) e o gateway (ESP01 @ 3,3 V) precisam se
comunicar por UART em 9600 8N1. Entre os dois há um *level-shifter*
5 V→3,3 V (divisor resistivo ou MOSFET transistor) que introduz risco de
erro de *bit* por ruído, *glitches* na transição de alimentação e
incompatibilidade de *slew rate*.

A versão 0.1.0 do firmware transmite linhas de texto ASCII como:

```
Tensao AC: 127.43 V
QUEDA NA REDE
```

O gadget desktop (medtens-gadget v0.1.0) já consome esse formato, o que é
aceitável para **monitoramento local de um humano** mas não para
**ingestão automatizada** em pipeline:

- Sem número de sequência → perda silenciosa de medições
- Sem checksum → aceita `Tensao AC: 999.99 V` corrompido como válido
- Parser frágil (regex sensível a espaços e pontuação)
- Tamanho variável → não cabe em *ring buffer* de tamanho fixo

Para o pipeline gateway→coletor→servidor, isso inviabiliza o requisito
não-funcional **RNF-011 (0 % de frames com CRC inválido aceitos)**.

---

## Decisão

Adotar um **protocolo binário com delimitadores, sequência e CRC16** para
a comunicação UART do sensor para o gateway:

```
+------+------+-----+------+-----------+-----------+
| SOF1 | SOF2 | LEN | TYPE |  PAYLOAD  |  CRC16 LE |
| 0xAA | 0x55 |  N  |  T   |  N bytes  |  2 bytes  |
+------+------+-----+------+-----------+-----------+
```

- **SOF1/SOF2** (Start Of Frame): `0xAA 0x55` — sincronização; o parser
  descarta bytes até encontrar esse par.
- **LEN**: tamanho do payload (0–255 bytes).
- **TYPE**: identificador do tipo de mensagem.
  - `0x01` = `MEASUREMENT` (`seq:uint16 LE | vrms_x100:uint16 LE | flags:uint8`)
  - `0x02` = `EVENT_OUTAGE_BEGIN`
  - `0x03` = `EVENT_OUTAGE_END` (`dur_ms:uint32 LE`)
  - `0xFF` = `DIAG`
- **PAYLOAD**: bytes conforme o tipo.
- **CRC16**: CRC-16/CCITT-FALSE (poly `0x1021`, init `0xFFFF`, *little
  endian*) calculado sobre `LEN | TYPE | PAYLOAD`.

O gateway valida CRC antes de republicar; frames inválidos são contados
em `medtens/sensor/<id>/diag` mas **não são publicados** como medição.

A **v0.1.0 do firmware atual (texto ASCII)** é mantida como compatível
para depuração via serial monitor; a **v0.2.0** introduzirá o formato
binário em paralelo, e a **v0.3.0** removerá o texto.

---

## Consequências

### Positivas

- **Integridade verificável** — CRC16-CCITT detecta todos os erros de
  até 16 bits e a maioria dos erros em *burst*, atendendo RNF-011.
- **Detecção de lacunas** — `seq` permite calcular perda percentual para
  diagnóstico.
- **Tamanho fixo por tipo** — `MEASUREMENT` tem 5 bytes de payload +
  6 bytes de *overhead* = 11 bytes = ~11 ms a 9600 baud, cabe
  confortavelmente na cadência de 500 ms do sensor.
- **Extensibilidade** — novos tipos adicionam-se sem quebrar *parsers*
  existentes.

### Negativas

- **Complexidade extra no firmware ATmega** — máquina de estados de
  transmissão vs. `uart_puts()` atual.
- **Perda da legibilidade direta** no *serial monitor* — necessita
  decodificador (pode ser um script Python de 30 linhas).
- **Quebra o gadget desktop atual** — o parser do `medtens-gadget` v0.1.0
  (regex `Tensao AC: X V`) deixa de funcionar quando migrarmos para
  binário. Solução: adicionar *decoder* binário no gadget na v0.2.0.

### Neutras

- CRC16-CCITT é o mesmo polinômio usado em XMODEM, BACnet e HDLC — há
  implementações de referência em C para AVR em ~30 linhas.

---

## Alternativas rejeitadas

| Alternativa | Motivo |
|---|---|
| Manter texto ASCII + checksum em hex | Resolve integridade mas não o tamanho variável nem a regex frágil |
| SLIP + binário sem CRC | Delimitação ok, mas ruído do level-shift precisa de detecção, não só enquadramento |
| Modbus RTU | *Overkill*; desenhado para ambiente industrial multi-mestre. O MedTens é ponto-a-ponto |
| Protocol Buffers | Requer biblioteca pesada no AVR; desproporcional para 5 bytes de payload |

---

## Referências

- Koopman (2002) — *32-Bit Cyclic Redundancy Codes for Internet Applications* (análise de CRC-16-CCITT)
- Tanenbaum e Wetherall (2011) — *Computer Networks*, cap. 3 (enquadramento e detecção de erros)
