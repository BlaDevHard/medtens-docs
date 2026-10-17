# Sessão 2026-04-18 — Gadget como instrumento inicial de Verificação e Validação (V&V) do sensor

> **Convenção de abreviações:** primeira menção com forma completa + sigla
> entre parênteses; depois só a sigla. Dicionário-fonte em
> `docs/glossario.md`.

## Objetivo

Registrar a linha do tempo e a motivação do `medtens-gadget` como ferramenta
**inicial de verificação da leitura do sensor** (cadeia ZMPT101B → ATmega328P
→ Universal Asynchronous Receiver/Transmitter (UART)), usando o multímetro
**ANENG M118A** como referência prática de bancada.

> Este não é um ensaio metrológico acreditado. É o **primeiro laço de
> confiança** para validar que o sinal sai do sensor até a interface gráfica
> do usuário (Graphical User Interface — GUI) do gadget sem corrupção e com
> leitura dentro da ordem de grandeza esperada da rede elétrica.

---

## 1. Contexto

O MedTens tem quatro componentes em cadeia:

```
ZMPT101B → ATmega328P → UART → ESP01 → MQTT → ESP32 collector → MySQL
```

Antes de construir o caminho completo, precisávamos **evidência independente**
de que a primeira metade da cadeia (ZMPT101B → ATmega → saída UART) produz
valores compatíveis com a tensão real da tomada. Sem isso, qualquer bug nas
camadas seguintes (parser, Message Queuing Telemetry Transport (MQTT), banco)
fica indistinguível de um bug do sensor.

**O gadget desktop em Python/PyQt6 resolve isso como instrumento de
bancada**: lê a saída UART do ATmega diretamente (bypass do ESP01 e do MQTT)
e exibe a tensão em valor eficaz (Root Mean Square — RMS) e eventos de queda
em tempo real. Comparamos visualmente com o multímetro de valor eficaz
verdadeiro (True Root Mean Square — True RMS) conectado ao mesmo ponto de
medição.

---

## 2. Multímetro de referência: ANENG M118A

Instrumento de bancada usado como **referência prática** (não é padrão
metrológico; é um True RMS comercial popular e aceito para triagem).

### Características principais

| Item | Valor |
|---|---|
| Tipo | Multímetro digital de bolso, auto-ranging |
| Display | 6000 contagens, 3 5/6 dígitos |
| Modo AC | True RMS |
| Faixa de tensão em corrente alternada (Alternating Current — AC) | 0,5 V – 600 V |
| Exatidão AC (6 V/60 V/600 V) | ±(0,5% da leitura + 3 dígitos) |
| Ambiente nominal | 23 ±5 °C, umidade relativa < 75% |
| Janela de garantia da exatidão | 1 ano |
| Recursos auxiliares | Non-Contact Voltage (NCV), Data Hold, lanterna, backlight, detecção fase/neutro, desligamento automático |

### O que esses números significam para o nosso uso

Na faixa de 127 V (tensão nominal da rede em Antonina-PR), a faixa ativa é
a de 600 V. Com a exatidão ±(0,5% + 3):

- Erro percentual: 0,5% × 127 = **±0,635 V**
- Erro dos dígitos: 3 × 0,1 V (resolução da faixa) = **±0,3 V**
- **Incerteza total de referência: ≈ ±0,94 V** em torno de 127 V

Isso é **mais que suficiente para triagem**, mas **insuficiente** para
qualquer alegação regulatória (acreditação pela Rede Brasileira de
Calibração (RBC), International Electrotechnical Commission (IEC)
61000-4-30 Classe A exigiria < ±0,1%).

---

## 3. Linha do tempo do gadget

### 2026-04-17 — gênese (necessidade)
- Firmware ATmega v0.1.0 rodando, emitindo código American Standard Code
  for Information Interchange (ASCII) pela UART a 9600 baud.
- Sem forma prática de ver o stream em tempo real: `screen /dev/ttyUSB0`
  dá o texto, mas não comparação visual com o multímetro.
- Decisão: construir um **widget desktop mínimo** para exibir tensão
  eficaz (Voltage Root Mean Square — Vrms), classificação pelos
  Procedimentos de Distribuição de Energia Elétrica no Sistema Elétrico
  Nacional (PRODIST) simplificada e sparkline de 5 min, servindo como
  "osciloscópio pobre" ao lado do ANENG M118A.

### 2026-04-17 — bootstrap
- `pyproject.toml` com dependências: PyQt6, pyserial, pyqtgraph
- `SerialReader` em QThread (não bloqueia UI)
- `GadgetWindow` frameless, sempre-no-topo, arrastável
- Autostart via `~/.config/autostart/medtens-gadget.desktop`
- Tag `v0.1.0` e push para `BlaDevHard/medtens-gadget` (privado)

### 2026-04-18 — análise de viabilidade (hoje)
- Revisão da stack Python vs C++ (ver análise em chat de sessão):
  - Carga do gadget é trivial (20 B/s de UART, 300 floats em janela)
  - Garbage Collector (GC) do Python não é problema nesta classe
  - C++ seria otimização prematura
- Decisão: **manter Python/PyQt6** até v1.0 ou até surgir requisito real
  de binário standalone.
- Documentação desta linha do tempo (este arquivo).

### Ainda nesta janela (a executar)
- Ensaio comparativo **gadget vs ANENG M118A** na mesma tomada, ao longo
  de pelo menos 30 min, registrando:
  - Tensão nominal esperada: 127 V (Antonina-PR, concessionária
    Companhia Paranaense de Energia (Copel))
  - Leitura do M118A (referência visual, anotada manualmente)
  - Leitura do gadget (captura de tela + log opcional)
  - Delta observado
- Se delta > ±3 V, disparar campanha de calibração do ZMPT101B
  (ver `docs/calibracao.md`).

---

## 4. Papel do gadget na arquitetura ao longo do tempo

O gadget é **deliberadamente descartável como caminho de produção**. Seu
valor está em três fases distintas:

| Fase | Papel do gadget | Fonte de dados |
|---|---|---|
| **Agora (v0.1.x)** | Instrumento de bancada para V&V do sensor vs ANENG M118A | UART direta do ATmega |
| **v0.2.x** | Ferramenta de diagnóstico durante bring-up do ESP01 gateway | UART do ATmega (mesma) |
| **v1.0** | Widget informativo no desktop do usuário final | MQTT do broker |
| **Pós-TCC** | Opcional — pode ser aposentado se a interface web do `medtens-server` cobrir o caso de uso | — |

Importante: trocar a camada de dados (UART → MQTT) **não muda a UI**. A
arquitetura já separa `SerialReader` (fonte) de `GadgetWindow` (apresentação),
exatamente para permitir essa migração sem reescrita.

> Sigla "TCC" (Trabalho de Conclusão de Curso) usada acima na linha
> "Pós-TCC" — expansão canônica registrada no glossário.

---

## 5. Limitações do ensaio gadget vs M118A

Para registro honesto (importante no Trabalho de Conclusão de Curso (TCC)):

1. **Referência não é rastreável**: o M118A não tem certificado RBC.
   Só permite dizer "estão de acordo dentro da incerteza combinada dos
   dois instrumentos".
2. **Incerteza combinada**: σ_gadget (ZMPT101B ±1–3%) ⊕ σ_M118A (±0,94 V)
   ≈ ±4 V em torno de 127 V. Qualquer delta menor que isso é ruído.
3. **Condições ambientais não controladas**: tensão da rede flutua, carga
   da casa muda, temperatura do ZMPT101B deriva.
4. **Janela curta**: 30 min não detecta Duração Relativa de Transgressão
   de Tensão Precária (DRP) nem Duração Relativa de Transgressão de
   Tensão Crítica (DRC) — esses exigem janelas de 168 h.

Esse ensaio é **teste de sanidade**, não de conformidade regulatória.

---

## 6. Referências

- Manual ANENG M118A:
  [ManualsLib](https://www.manualslib.com/manual/3321291/Aneng-M118a.html)
- Ficha técnica comercial:
  [electrical center b2c](https://www.electric-b2c.com/products/aneng-m118a-digital-mini-multimeter-tester-auto-mmultimetro-true-rms-tran),
  [Bot'n Roll](https://www.botnroll.com/en/multimeters/3836-true-rms-multimeter-with-automatic-scale-and-ncv-m118a.html)
- Análise de stack do gadget: chat de sessão 2026-04-18
- ZMPT101B e faixas PRODIST: ver `docs/calibracao.md`, `docs/indice-qualidade.md`
- Glossário de siglas: `docs/glossario.md`

## 7. Próxima ação

Retomar os **"próximos passos úteis"** discutidos no chat:
(a) guard de buffer no `serial_reader.py`, (b) benchmark com `tracemalloc` +
`psutil`, (c) Architecture Decision Record (ADR) formal sobre a stack do gadget.
