# Índices de Qualidade de Fornecimento

**Status**: esboço. Fórmulas a serem validadas contra [PRODIST Módulo 8](https://www.gov.br/aneel/) (ANEEL) e IEEE 1159.

## Indicadores candidatos (fase 1 — só tensão)

### 1. Tensão média e desvio

Sobre janela deslizante de N amostras:

- `V_avg = (1/N) · Σ V_rms,i`
- `σ_V = √( (1/N) · Σ (V_rms,i − V_avg)² )`

### 2. Classificação por faixa (PRODIST 8, TR = 127 V)

| Faixa | Limite inferior | Limite superior |
|---|---|---|
| Adequada | 117 V | 133 V |
| Precária (inferior) | 110 V | 117 V |
| Precária (superior) | 133 V | 135 V |
| Crítica | < 110 V ou > 135 V |

Para cada janela horária, calcular a fração do tempo em cada faixa:

- `DRP` (Duração Relativa da transgressão de tensão **Precária**)
- `DRC` (Duração Relativa da transgressão de tensão **Crítica**)

### 3. Afundamentos e elevações (sags/swells — IEEE 1159)

- **Sag**: V_rms entre 10 % e 90 % do nominal por 0,5 ciclo a 1 min
- **Swell**: V_rms entre 110 % e 180 % do nominal por 0,5 ciclo a 1 min
- **Interrupção**: V_rms < 10 % do nominal

### 4. Índice composto (proposta)

`Q = w1 · (1 − DRP) + w2 · (1 − DRC) + w3 · (1 − n_eventos/max_eventos)`

Pesos `w_i` a calibrar a partir de dados reais.

## Indicadores futuros (fase 2 — demanda mais hardware)

- **THD (Total Harmonic Distortion)**: requer FFT sobre forma de onda; ATmega
  não entrega taxa de amostragem suficiente. Mover cálculo para ESP32.
- **Frequência da rede**: detecção de cruzamento por zero no ATmega.
- **Flicker (Pst, Plt)**: IEC 61000-4-15. Complexo, considerar biblioteca.

## Referências

Ver [bibliografia.bib](bibliografia.bib).
