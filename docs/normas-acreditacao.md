# Normas e vias de acreditação aplicáveis ao MedTens

> Este documento mapeia o **caminho formal de acreditação** para um sistema
> de medição de qualidade da energia elétrica no Brasil. Serve como
> referência única para o Trabalho de Conclusão de Curso (TCC) e para a
> decisão sobre o **uso pretendido** do MedTens
> (ver `academica/01-plano-tcc.md` e `memory/medtens_sessao_atual.md`).
>
> **Conclusão antecipada**: acreditação plena exige classe A da International
> Electrotechnical Commission (IEC) 61000-4-30, certificação da Agência
> Nacional de Energia Elétrica (ANEEL), calibração em laboratório acreditado
> pela Coordenação Geral de Acreditação (CGCRE) do Instituto Nacional de
> Metrologia, Qualidade e Tecnologia (INMETRO), e laboratório ensaiador
> acreditado na norma ABNT NBR ISO/IEC 17025. Incompatível com orçamento e
> escopo de TCC usando sensor ZMPT101B.

---

## 1. Mapa das normas relevantes

```
┌──────────────────────────────────────────────────────────────────┐
│ Camada regulatória (ANEEL) — O QUE precisa ser medido             │
│   PRODIST Módulo 8 Rev.12 (REN 956/2021, ajustes REN 1148/2026)  │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ Camada técnica (IEC/ABNT) — COMO medir                            │
│   IEC 61000-4-30  → método de medição (classe A ou S)            │
│   IEC 62586-1     → produto: segurança, EMC, ambiental           │
│   IEC 62586-2     → ensaios funcionais e incertezas              │
│   ABNT NBR IEC 61000-4-30 (adoção brasileira da IEC)             │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ Camada metrológica — POR QUEM precisa ser calibrado/ensaiado      │
│   ABNT NBR ISO/IEC 17025 → laboratórios de calibração/ensaio      │
│   CGCRE/INMETRO          → órgão acreditador no Brasil           │
│   ILAC MRA               → reconhecimento internacional           │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. Camada regulatória — ANEEL / PRODIST Módulo 8

**Procedimentos de Distribuição de Energia Elétrica (PRODIST), Módulo 8 —
Qualidade da Energia Elétrica (QEE)**, revisão 12, aprovada pela Resolução
Normativa (REN) 956/2021 da ANEEL e atualizada pela REN 1148/2026.

Estabelece:

- Indicadores obrigatórios de QEE: Duração Relativa de Transgressão de
  Tensão Precária (DRP), Duração Relativa de Transgressão de Tensão Crítica
  (DRC), distorção harmônica total (DHT), desequilíbrio, flutuação (flicker
  Pst/Plt), frequência.
- Faixas de tensão adequada/precária/crítica para cada tensão nominal
  (127 V, 220 V, 380 V, etc.).
- **Janela mínima de medição: 168 horas (7 dias)** para apuração de DRP/DRC.
- A partir de **2023**: certificação obrigatória do processo de medição,
  coleta de dados, apuração dos indicadores e compensações relacionadas à
  tensão em regime permanente.
- Referência obrigatória aos **métodos de medição classe A ou classe S** da
  IEC 61000-4-30 (incorporada via ABNT NBR IEC 61000-4-30).

**Quando cada classe é exigida:**

| Caso de uso | Classe exigida |
|---|---|
| Medição solicitada pelo consumidor (rotina) | Classe S |
| Ação judicial, indenização, compensação | **Classe A** |
| Rede básica (alta tensão) | **Classe A** |

Sem enquadramento em classe A, **não há valor probatório** para acionamento
da concessionária com pedido de indenização. Esse é o ponto central da
questão levantada em 2026-04-17.

---

## 3. Camada técnica — IEC 61000-4-30, IEC 62586-1 e IEC 62586-2

### 3.1. IEC 61000-4-30 (edição 2025)

Norma-mãe dos métodos de medição. Define, para cada parâmetro de QEE:

- Métodos de medição, agregação temporal (3 s, 10 min, 2 h)
- Requisitos de exatidão
- Formato de saída dos resultados

**Classe A** — exatidão máxima, resultados repetíveis e comparáveis entre
instrumentos diferentes:
- **Incerteza de medição de tensão: ±0,1%** da tensão de entrada
- Harmônicas até a 50ª ordem, inter-harmônicas
- Desequilíbrio de tensão, variações rápidas
- Exigência de sincronização temporal de alta precisão (geralmente Global
  Positioning System (GPS) ou Network Time Protocol (NTP) com disciplina
  rigorosa)

**Classe S** — métodos simplificados para aplicações estatísticas de rotina.
Requisitos de exatidão e sincronização mais brandos.

### 3.2. IEC 62586-1 — produto (PQI, Power Quality Instrument)

Define requisitos de **produto** para o instrumento físico:
- Segurança elétrica
- Compatibilidade Eletromagnética (EMC)
- Requisitos climáticos e mecânicos (temperatura, umidade, vibração)

### 3.3. IEC 62586-2 — ensaios funcionais

Define **como ensaiar** um instrumento para verificar conformidade com a
classe A ou S da IEC 61000-4-30. É a norma pela qual um laboratório
acreditado vai submeter o protótipo a ensaio.

### 3.4. ABNT NBR IEC 61000-4-30

Adoção nacional (ABNT) idêntica à IEC, com carimbo brasileiro. É a versão
citada pelo PRODIST e exigida em licitações.

---

## 4. Camada metrológica — ABNT NBR ISO/IEC 17025 e CGCRE/INMETRO

### 4.1. ABNT NBR ISO/IEC 17025

Norma de **competência técnica de laboratórios de calibração e ensaio**.
Exige:
- Sistema da qualidade documentado
- Rastreabilidade metrológica até padrões nacionais/internacionais
- Estimativa formal de incerteza
- Validação de métodos
- Pessoal tecnicamente qualificado

### 4.2. CGCRE/INMETRO

A Coordenação Geral de Acreditação (CGCRE) do INMETRO é o **órgão
acreditador brasileiro**. Acredita laboratórios segundo a ABNT NBR ISO/IEC
17025. A Divisão de Acreditação de Laboratórios (Dicla) conduz o processo.

CGCRE é signatária do International Laboratory Accreditation Cooperation
Mutual Recognition Arrangement (ILAC MRA) — relatórios de laboratórios
acreditados pela CGCRE são aceitos internacionalmente.

### 4.3. Rede Brasileira de Calibração (RBC)

Rede de laboratórios acreditados pela CGCRE para **calibração**
(diferente de ensaio). Uma calibração de medidor de tensão com valor legal
precisa vir de laboratório RBC com certificado rastreável.

---

## 5. O que seria necessário para o MedTens obter acreditação plena

Checklist realista:

1. **Trocar o sensor**. O ZMPT101B é um transformador de potencial de baixo
   custo com exatidão de ±1–3%. Classe A exige ±0,1% — duas ordens de
   grandeza de distância. Substituir por transdutor de tensão de precisão
   (LEM LV 25-P ou similar) com sua própria calibração rastreável.
2. **Projeto eletrônico qualificado em EMC**. Blindagem, filtros, layout de
   Printed Circuit Board (PCB) conforme IEC 62586-1. Protótipo em placa de
   desenvolvimento não passa.
3. **Sincronização temporal rigorosa**. Classe A geralmente exige GPS ou NTP
   com stratum baixo. ESP32 com NTP padrão fica marginal.
4. **Ensaiar o protótipo** em laboratório ABNT NBR ISO/IEC 17025 acreditado
   segundo IEC 62586-2. Custo típico: dezenas de milhares de reais por ciclo.
5. **Calibrar cada unidade produzida** em laboratório RBC. Certificado com
   validade de 1 ano.
6. **Processo de fabricação com rastreabilidade** de lote, firmware
   assinado, procedimento de campo documentado.
7. **Submissão à ANEEL** para reconhecimento como medidor apto às medições
   solicitadas (classe S) ou probatórias (classe A) conforme PRODIST Mód. 8.

**Tempo estimado**: 18 a 36 meses. **Custo**: incompatível com TCC.

---

## 6. O que o MedTens atual **pode** legitimamente alegar

Considerando as limitações (ZMPT101B, protótipo, sem acreditação), o MedTens
pode honestamente se posicionar como:

- **Sistema informativo de triagem de QEE para usuário final**
- **Indutor de medição oficial**: quando indicadores próprios apontam
  violação sustentada, o usuário aciona a concessionária (Companhia
  Paranaense de Energia — Copel) solicitando medição com instrumento
  acreditado, **com base em PRODIST Mód. 8 item 10.3** (direito do
  consumidor à medição gratuita).
- **Plataforma acadêmica experimental**: não substitui o medidor oficial da
  concessionária; fornece histórico pessoal e educa o consumidor.

Essa é a **Opção 1** discutida em 2026-04-17 ("manter informativo +
explicitar uso pretendido").

---

## 7. Via intermediária — conformidade parcial sem acreditação

Se o interesse for **aproximar-se do rigor da classe S** sem pretender
certificação formal (útil como diferencial do TCC):

- Declarar conformidade **parcial** aos parâmetros de agregação temporal da
  IEC 61000-4-30 classe S (3 s, 10 min).
- Documentar incerteza combinada do sensor + Analog-to-Digital Converter
  (ADC) + cadeia digital.
- Aplicar as faixas PRODIST Mód. 8 para 127 V (adequada/precária/crítica).
- Fazer ensaio **comparativo** contra medidor classe A (emprestado ou
  contratado pontualmente) por 168 h, reportando coeficiente de correlação
  e viés.

Isso dá valor acadêmico sem reivindicar acreditação.

---

## 8. Referências

### Regulamentação brasileira
- ANEEL, PRODIST Módulo 8 — [página oficial](http://app.aneel.gov.br/modulo-8)
- ANEEL, REN 956/2021, anexo VIII — [PRODIST Mód. 8 Rev.12 PDF](https://www2.aneel.gov.br/cedoc/aren2021956_prodist_modulo_8_v13.pdf)
- IMS Eng., "Revisão 12 do Módulo 8" — [ims.ind.br](https://ims.ind.br/blog/prodist-revisao-12-do-modulo-8/)
- KRA2 Energia, "Instrumentos de medição PRODIST Mód. 8" — [PDF](https://kra2energia.com/wp-content/uploads/2024/02/Instrumentos-de-Medicao-QEE.pdf)
- Enel RJ, especificação técnica de medidor portátil PQ conforme PRODIST — [PDF](https://www.eneldistribuicao.com.br/rj/documentos/MAT-NTBR-SGD-21-0020-INBR%20-%20Medidor%20Port%C3%A1til%20de%20Grandezas%20El%C3%A9tricas%20(PQ)%20conforme%20o%20m%C3%B3dulo%208%20(10.3%20e%2010.4)%20do%20PRODIST%20%E2%80%93%20ANEEL.pdf)

### Normas técnicas internacionais (IEC)
- IEC 61000-4-30:2025 — [webstore IEC](https://webstore.iec.ch/en/publication/71611)
- IEC 62586-2:2017 — [webstore IEC](https://webstore.iec.ch/en/publication/29675)
- Fluke, "What does IEC 61000-4-30 Class A mean to me?" — [fluke.com](https://www.fluke.com/en-gb/learn/blog/power-quality/what-does-the-iec-61000-4-30-class-a-standard-mean-to-me)
- Elspec, "Knowing IEC 61000-4-30 Class A" — [elspec-ltd.com](https://www.elspec-ltd.com/knowing-iec-61000-4-30-class-a/)
- Embrasul, "IEC 61000-4-30 — learn all about the standard" — [embrasul.com.br](https://www.embrasul.com.br/en/post/iec61000-4-30-learn-all-about-the-standard)
- Power Quality Blog, "Revised IEC and IEEE Standards for PQ Measurements" — [powerquality.blog](https://powerquality.blog/2021/07/30/revised-iec-and-ieee-standards-for-pq-measurements-for-users-of-pq-monitoring-equipment/)

### Normas técnicas nacionais (ABNT)
- ABNT NBR IEC 61000-4-30 — [Target Normas (catálogo)](https://www.normas.com.br/visualizar/abnt-nbr-nm/30581/abnt-nbriec61000-4-30-compatibilidade-eletromagnetica-parte-4-30-tecnicas-de-medicao-e-ensaio-metodos-de-medicao-de-qualidade-da-energia)

### Acreditação metrológica
- INMETRO/CGCRE, acreditação de laboratórios ABNT NBR ISO/IEC 17025 — [portal INMETRO](http://ftp.inmetro.gov.br/Credenciamento/acre_lab.asp)
- CMS Científica, "ISO 17025 e os laboratórios de calibração e ensaio" — [cmscientifica.com.br](https://cmscientifica.com.br/iso-17025-e-os-laboratorios-de-calibracao-e-ensaio/)
- Konrad, "ISO/IEC 17025 — o que é, para que serve" — [konrad.com.br](https://konrad.com.br/iso-iec-17025/)

---

## 9. Retomada

Este documento fecha a pergunta "quais normas se aplicam à acreditação" e
sustenta a decisão entre as três opções de rumo em aberto
(ver `memory/medtens_sessao_atual.md`). Próxima ação pendente no chat:
retomar os **próximos passos úteis** do gadget (guard de buffer, benchmark
com tracemalloc/psutil, Architecture Decision Record formal da stack Python).
