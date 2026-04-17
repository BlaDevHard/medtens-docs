# Plano de Validação e Ensaios — MedTens

**Versão:** 0.1 (esboço inicial)
**Data:** 2026-04-17
**Status:** em elaboração

> **Enquadramento no PPI Sumaré.** Artefato técnico de apoio à etapa de
> **Análise dos Dados** do Projeto Integrador (ver `01-plano-tcc.md`, § 4).
> Define, *a priori*, como cada requisito da ERS (`02-ers.md`) será
> verificado, os critérios de aceite, o aparato experimental e os
> procedimentos. A execução dos ensaios e a apresentação dos resultados
> constituem a evidência empírica exigida pela etapa de Análise dos Dados.

---

## 1. Objetivo

Fornecer **evidência empírica reproduzível** de que o MedTens, na versão
avaliada no TCC, satisfaz os requisitos da ERS — em especial:

- **RNF-020**: erro absoluto da medição RMS ≤ ±3 % em 90–150 V
- **RNF-011**: 0 % de frames com CRC inválido aceitos no gateway
- **RNF-012**: perda ≤ 5 s por reconexão MQTT
- **RNF-010**: disponibilidade ≥ 99 % em janela de 24 h
- **RF-022 / RF-023**: classificação PRODIST e apuração correta de DRP/DRC
- **RF-024**: detecção correta de eventos conforme IEEE 1159

Documentar, junto com cada resultado, a **incerteza de medição**
(ISO/IEC Guide 98-3, GUM) e a **rastreabilidade metrológica** ao
instrumento de referência.

## 2. Abrangência e limites

**Dentro do escopo:**

- Desempenho metrológico em regime senoidal quase-ideal (rede residencial)
- Correção da *pipeline* sensor → gateway → coletor → servidor
- Correção dos algoritmos de classificação e apuração de indicadores
- Resiliência a falhas de rede (WiFi, MQTT, MySQL) induzidas artificialmente

**Fora do escopo desta versão:**

- Ensaio com harmônicos severos (THD > 10 %) — apenas verificação qualitativa
- Ensaio com desequilíbrio de fases (sistema é monofásico por design)
- Ensaio de imunidade eletromagnética conforme IEC 61000-4-x completo
  (exige laboratório acreditado)
- Uso fiscal ou metrológico — explicitamente fora (ver ERS § 1.2)

## 3. Classificação dos ensaios

Adotamos três classes, hierárquicas:

| Classe | Quando ocorre | Tipo |
|---|---|---|
| **V&V-1 — Unitário** | Durante desenvolvimento, em bancada | Testes automatizados de módulos isolados |
| **V&V-2 — Integração** | A cada release candidate | Pipeline completa em bancada com sinal controlado |
| **V&V-3 — Campo** | Pré-entrega, na residência em Antonina-PR | Operação real ≥ 168 h contínuas |

## 4. Aparato experimental

### 4.1 Instrumento de referência (padrão)

| Item | Requisito | Observação |
|---|---|---|
| Multímetro true-RMS | Incerteza ≤ ±0,5 % na faixa 90–150 V AC | Modelo e certificado a registrar |
| Resolução | ≥ 0,1 V | |
| Crest factor | ≥ 3 | Para não saturar em presença de distorção |
| Certificado de calibração | ≤ 12 meses de validade | Rastreável à Rede Brasileira de Calibração (RBC) sempre que possível |

> Se o instrumento disponível não cumprir os requisitos acima, documentar
> explicitamente a limitação metrológica no relatório e classificar o
> resultado como **indicativo**, não **conclusivo**.

### 4.2 Fontes de tensão

| Ensaio | Fonte |
|---|---|
| Calibração e linearidade | Variac (autotransformador variável) 0–150 V, carga resistiva |
| Injeção de eventos | Gerador de sinal AC programável (se disponível) ou Variac operado manualmente com cronômetro |
| Operação contínua | Rede residencial real (127 V / 60 Hz Copel, Antonina-PR) |

### 4.3 Aparato complementar

- Osciloscópio para validação do sinal de entrada do ADC (pino A0)
- Analisador de qualidade de energia acreditado (acesso eventual) —
  como segundo padrão para **comparação** em campo, não calibração
- Computador desktop rodando o `medtens-gadget` e `mosquitto_sub` para
  observação ao vivo

## 5. Protocolos de ensaio

### 5.1 V&V-1 — Testes unitários

#### 5.1.1 CRC16-CCITT

**Objetivo:** garantir que o cálculo de CRC no ATmega e a validação no
ESP01 produzem o mesmo resultado em ≥ 10 000 frames aleatórios.

**Método:**
- Gerar vetor de teste em Python com `crcmod` (referência) e no ATmega
  via bancada PlatformIO (`pio test`)
- Comparar byte a byte

**Critério de aceite:** 100 % de coincidência em 10 000 casos.

#### 5.1.2 Parser de frame

**Objetivo:** validar que o `uart_rx` do ESP01 aceita frames válidos,
rejeita frames com CRC adulterado, e recupera sincronismo após *garbage*.

**Método:**
- Tabela de casos: frame válido / SOF trocado / LEN errado / CRC errado /
  bytes de ruído antes do SOF
- Injeção via *loopback* serial em PlatformIO nativo

**Critério de aceite:**
- Aceitar 100 % dos válidos
- Rejeitar 100 % dos inválidos (RNF-011)
- Ressincronizar em ≤ 3 frames após *garbage* contínuo

#### 5.1.3 Classificador PRODIST

**Objetivo:** provar que `classify(v)` no coletor (e no gadget)
produz a faixa correta para cada V.

**Método:**
- Tabela de pontos nos limites exatos: 109,99 / 110,00 / 116,99 / 117,00
  / 133,00 / 133,01 / 134,99 / 135,00 / 135,01
- Teste parametrizado em `pytest`

**Critério de aceite:** 100 % dos pontos com classe correta.

### 5.2 V&V-2 — Integração em bancada

#### 5.2.1 Linearidade (precisão metrológica)

**Objetivo:** verificar RNF-020 em condições controladas.

**Método:**

1. Aplicar, via Variac, nove pontos: 90, 100, 110, 117, 120, 127, 133,
   135, 140 V (medidos simultaneamente pelo padrão).
2. Em cada ponto, aguardar 60 s de estabilização; coletar 120 medições
   consecutivas do MedTens (via `medicoes_brutas`).
3. Calcular `V_med`, `σ_V` e `erro = (V_med − V_ref) / V_ref`.
4. Gerar gráfico `V_MedTens × V_ref` com regressão linear.
5. Calcular incerteza combinada conforme GUM.

**Critério de aceite:**

- `|erro|` ≤ 3 % em todos os nove pontos (RNF-020)
- R² ≥ 0,999 (linearidade)
- Coeficiente angular entre 0,97 e 1,03

#### 5.2.2 Protocolo UART sob ruído

**Objetivo:** confirmar que CRC16 rejeita 100 % dos frames corrompidos
mesmo com agressor EMI.

**Método:**
- Medição normal durante 30 min com fonte eletrônica chaveada próxima ao
  *level-shifter* como agressor
- Coletar contadores de `frames_recebidos`, `frames_validos`, `frames_descartados`

**Critério de aceite:**

- `frames_descartados` > 0 (ou ausência comprovada de agressor)
- **nenhum** frame com `vrms` fisicamente impossível (|V| > 150 V ou
  |V| < 50 V fora de evento anunciado) entregue ao coletor

#### 5.2.3 Resiliência MQTT

**Objetivo:** verificar RNF-012.

**Método:**

1. Operação nominal por 5 min
2. `systemctl stop mosquitto` (ou bloqueio de firewall)
3. Aguardar 120 s
4. `systemctl start mosquitto`
5. Medir: `tempo_reconexao_gateway`, `tempo_reconexao_coletor`,
   `medicoes_perdidas` (comparando `seq` antes/depois)

**Critério de aceite:**

- Reconexão ≤ 30 s em ambas as pontas
- Lacuna de medições entregue ao MySQL ≤ 5 s (RNF-012), assumindo que o
  buffer local do coletor em SPIFFS foi preservado

#### 5.2.4 Resiliência MySQL

**Objetivo:** verificar que o coletor bufferiza durante queda de SQL e
faz *flush* ordenado.

**Método:**

1. `systemctl stop mysql` durante 10 min com pipeline ativo
2. Reativar
3. Conferir: (a) *flush* completo; (b) ordem temporal preservada;
   (c) ausência de duplicatas (chave por `seq`)

**Critério de aceite:**

- 100 % das medições emitidas no período de indisponibilidade presentes
  no MySQL após o *flush*
- Ordem monotônica de `ts_epoch_ms`
- Zero duplicatas (UNIQUE em `(sensor_id, seq)`)

#### 5.2.5 Injeção de eventos IEEE 1159

**Objetivo:** verificar RF-024.

**Método:** com Variac operado manualmente, induzir:

| Evento | V objetivo | Duração | Classificação IEEE 1159 esperada |
|---|---|---|---|
| Instantaneous sag | ~80 V | 0,5 a 30 ciclos | `instantaneous sag` |
| Momentary sag | ~80 V | 30 ciclos a 3 s | `momentary sag` |
| Temporary interruption | < 10 V | 3 s a 1 min | `temporary interruption` |
| Momentary swell | ~140 V | 30 ciclos a 3 s | `momentary swell` |

**Critério de aceite:**

- Cada evento induzido gera 1 linha correspondente em `eventos`
- `tipo` e `severidade` corretos conforme IEEE 1159 § 4.4.1–4.4.3
- `v_min`/`v_max` dentro de ±3 % do valor injetado
- `dur_ms` dentro de ±100 ms (resolução do Timer0 + jitter de rede)

### 5.3 V&V-3 — Operação em campo

**Local:** residência em Antonina-PR, alimentação Copel Distribuição.

**Duração mínima:** 168 h contínuas (1 semana).

**Métricas coletadas automaticamente:**

| Métrica | Como obter | Critério |
|---|---|---|
| Disponibilidade pipeline | `n_medicoes_recebidas / n_medicoes_esperadas` | ≥ 99 % (RNF-010) |
| Latência média | `ts_servidor − ts_epoch_ms` por medição | Mediana ≤ 1 s; p95 ≤ 2 s |
| Frames UART descartados | `diag.frames_descartados / diag.frames_recebidos` | ≤ 0,1 % em ambiente nominal |
| Reconexões MQTT | contador no gateway | Documentar, sem critério rígido |
| DRP horário | tabela `indices_qualidade` | Comparar com fatura Copel se disponível |
| DRC horário | idem | idem |

**Observações registradas no lab-notebook:**

- Data/hora de qualquer queda percebida pelo morador
- Correlação com eventos registrados no MedTens
- Anormalidades meteorológicas (tempestade, vento forte)

### 5.4 Verificação cruzada opcional

Se houver acesso a um analisador de qualidade acreditado (emprestado da
Copel, laboratório universitário ou similar), operar o MedTens em
paralelo com ele por ≥ 24 h e comparar:

- Leituras RMS ponto a ponto (dispersão esperada ≤ ±3 %)
- Eventos detectados (razão de verdadeiros positivos ≥ 90 %)

Este ensaio é **altamente desejável** para a defesa do TCC, pois eleva a
evidência de "aceitável" para "rastreável".

## 6. Registro e publicação dos dados

### 6.1 Convenção de nomes

```
data/validacao/
├── v2-01-linearidade_YYYY-MM-DD.csv
├── v2-02-uart-emi_YYYY-MM-DD.csv
├── v2-03-mqtt-resiliencia_YYYY-MM-DD.csv
├── v2-04-mysql-resiliencia_YYYY-MM-DD.csv
├── v2-05-eventos-ieee1159_YYYY-MM-DD.csv
└── v3-01-campo-168h_YYYY-MM-DD-to-YYYY-MM-DD/
    ├── medicoes_brutas.csv
    ├── indices_qualidade.csv
    └── eventos.csv
```

### 6.2 Metadados mínimos em cada CSV

Primeiras linhas do arquivo como comentários `#`:

```
# MedTens versão: <git-sha>
# Firmware sensor: <version>
# Firmware gateway: <version>
# Firmware coletor: <version>
# Instrumento padrão: <modelo>, cert. <número>, validade <YYYY-MM-DD>
# Operador: <nome>
# Temperatura ambiente: <°C>
# Observações: <texto livre>
```

### 6.3 Relatório de validação

Um arquivo `academica/05-relatorio-validacao.md` (a criar após execução)
consolidará:

- Gráficos de linearidade com incerteza
- Tabelas de resultados por ensaio
- Interpretação por requisito (passa / falha / indicativo)
- Discussão de limitações e ameaças à validade

### 6.4 Publicação

Dados brutos e *notebooks* de análise versionados no `medtens-docs`.
*Release* de validação marcada com tag `validacao-vX.Y.Z` e DOI Zenodo
(quando o repositório público for configurado).

## 7. Gestão de riscos e ameaças à validade

| Ameaça | Impacto | Mitigação |
|---|---|---|
| Instrumento de referência com incerteza > ±0,5 % | Valida ±3 % com confiança reduzida | Documentar limitação; classificar resultado como indicativo |
| Variac introduz distorção (THD > 3 %) | Afeta cálculo RMS do sensor | Validar senoide no osciloscópio; usar carga resistiva pura |
| Dispersão térmica do ZMPT101B entre laboratório e campo | Erro sistemático | Registrar temperatura; recalibrar em campo se erro > ±1 % |
| Indisponibilidade do laboratório / analisador acreditado | Impossibilita V-Cruzado | Executar apenas V&V-1 a V&V-3; documentar como ameaça à validade |
| Queda prolongada de energia em campo (> 4 h) | Buffer SPIFFS satura | Documentar perda; ensaio não invalidado se ≤ 2 ocorrências em 168 h |
| Janela de 168 h sem eventos | Impossibilita validação de DRP/DRC > 0 | Aceitar; operação contínua e dados de classificação ainda valem |

## 8. Cronograma de execução

Integrado ao cronograma geral do PPI (`01-plano-tcc.md`, § 6). Dependências:

- V&V-1 pode iniciar assim que firmware v0.1.x estiver congelado
- V&V-2 exige firmware gateway (v0.1.x+) e coletor (v0.1.x+) — bloqueado
  pela implementação
- V&V-3 exige V&V-2 aprovado e semana contígua de operação

## 9. Histórico de revisões

| Versão | Data | Autor | Alterações |
|---|---|---|---|
| 0.1 | 2026-04-17 | autor | Versão inicial (esboço) |
