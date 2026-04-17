# Calibração do ZMPT101B

**Status**: a executar. Valor atual (`calibration_factor = 0.85`) é uma
estimativa inicial não validada.

## Metodologia proposta

### Materiais

- Multímetro true-RMS de referência (modelo e incerteza a documentar)
- Variac (autotransformador variável) ou cargas que produzam tensões distintas
- Carga não linear (opcional, para verificar comportamento em presença de harmônicos)

### Procedimento

1. Fixar o potenciômetro multivoltas do ZMPT101B em posição inicial (ajustar para que a senoide centralizada fique próxima de 2,5 V no pino de saída quando V_in nominal).
2. Para cada ponto de tensão `V_ref` medida com o multímetro padrão:
   - Coletar N = 100 leituras consecutivas do ATmega (saída do firmware)
   - Registrar média `V_med` e desvio-padrão
3. Ajustar `calibration_factor = V_ref / V_med` por regressão linear
   (espera-se coeficiente angular ≈ 1 se o sensor é linear).

### Faixa mínima de pontos

- 90, 100, 110, 117, 120, 127, 133, 135, 140 V (cobre faixas PRODIST)
- Se possível, estender até 150 V para verificar saturação.

## Dados

Medições brutas serão registradas em `data/calibracao_YYYY-MM-DD.csv` com
colunas: `timestamp, V_ref, V_med, sigma_V, temperatura_ambiente`.

## Análise

Notebook em `analysis/calibracao.ipynb`. Saídas esperadas:

- Gráfico V_med × V_ref com regressão
- Erro residual em cada ponto
- Valor recomendado de `calibration_factor` com incerteza
