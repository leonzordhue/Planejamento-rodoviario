<!-- AKE/UFT-1.0 | BUILD: SICRO-20260609 | IC: 1.0 | MÓDULO: metodologia -->
# Metodologia e Fórmulas — SICRO (DNIT)

> Fórmulas oficiais para cálculo de custos referenciais. Fonte: **Manual de Custos de
> Infraestrutura de Transportes (DNIT)** — Módulos 1–5 do curso de Orçamentação de Obras
> Rodoviárias. Parâmetros numéricos em `json/constantes_metodologia.json`.
> Os dados (preços) estão em `csv/` e `json/`. Ver `README.md` para o dicionário de dados.

---

## 1. Estrutura geral do custo

```
Preço de Venda (PV) = Custo Direto (CD) × (1 + BDI/100)

Custo Direto do serviço = Σ insumos por seção:
   A. Equipamentos    B. Mão de obra    C. Materiais
   D. Atividades auxiliares    E. Tempo fixo    F. Momento de transporte
```

O SICRO divulga o **Custo Direto** (campo `custo_unitario_direto` / `custo_unitario`).
BDI, Administração Local, Canteiro e Mobilização são orçados à parte.

---

## 2. Composição de custos

```
CUSTO_SERVIÇO = Σ (coeficiente_consumo × custo_unitário_insumo)
```

- **Composição analítica:** insumo a insumo (tabela `composicoes_insumos`).
- **Composição sintética:** custo unitário já consolidado (tabela `composicoes_sinteticas`).
- **Tipos de composição:** horária, unitária e **mista** (padrão SICRO, serviços cíclicos).
- **Atividades auxiliares** (seção D) são, elas mesmas, composições — relação recursiva.

**Fator de eficiência (tempo):** fe = **0,83** (construção/recuperação) ou **0,75**
(conservação). Ajuste fino via FIT.
**Fator de carga (peso):** 1ª cat. 0,90 · 2ª cat. 0,80 · 3ª cat. 0,70.

---

## 3. Custo horário de equipamentos

```
Chp = Dh + Jh + Mh + Cc + Cmo + Ih      (custo horário PRODUTIVO)
Chi = Cmo + Dh + Jh + Ih                (custo horário IMPRODUTIVO)
```

| Símbolo | Componente | Coluna na base |
|---|---|---|
| Dh | Depreciação horária | `depreciacao` |
| Jh | Juros de oportunidade de capital | `oportunidade_capital` |
| Mh | Manutenção | `manutencao` |
| Cc | Combustíveis/lubrificantes | `operacao` |
| Cmo | Mão de obra de operação | `mao_obra_operacao` |
| Ih | Seguros e impostos | `seguros_impostos` |

**Depreciação (linear):** `Dh = (Va − Vr) / (n × HTA)`
*(Va = valor de aquisição; Vr = valor residual; n = vida útil em anos; HTA = horas trabalhadas/ano)*

**Oportunidade de capital:** `Vm = ((n+1)/(2n)) × Va` ; `Jh = (Vm × i) / HTA`
*(i = taxa de juros anual = **6,0%**, atrelada à Selic)*

**Combustível:** `Cc = P × Fc × Vc` *(P = potência; Fc = coef. consumo; Vc = preço do combustível)*

| Coeficiente Fc | Valor |
|---|---|
| Diesel (equipamentos e caminhões) | 0,18 l/kWh |
| Gasolina | 0,20 l/kWh |
| Álcool | 0,28 l/kWh |
| Elétrico | 0,85 kWh/kWh |

---

## 4. Custo de mão de obra

```
Custo_MO = Salário + Encargos_Totais(R$) + Periculosidade(R$)
Encargos_Totais(R$) = Enc_Sociais_Trab(R$) + Enc_Complementares(R$) + Enc_Adicionais(R$)
```

> Identidades validadas em 290/290 registros (ver `README.md` §6).

### Grupos de encargos sociais

| Grupo | Conteúdo |
|---|---|
| **A** | Básicos: Previdência, FGTS, Salário-Educação, SESI/SESC, SENAI/SEBRAE, INCRA, Seguro Acidente, Seconci, FAE |
| **B** | Recebem incidência de A: RSR, feriados, férias+1/3, auxílio enfermidade/acidente, licença paternidade, 13º, faltas justificadas, reciclagem |
| **C** | Verbas rescisórias: aviso prévio (indenizado/trabalhado), férias indenizadas+1/3, depósito rescisão, indenização adicional |
| **D** | Reincidências: de A sobre B, de A sobre aviso prévio |

**Encargos complementares:** alimentação, EPI, ferramenta, transporte, consulta ocupacional.
**Encargos adicionais:** cesta básica, assistência médica, seguro de vida etc.

### Desoneração da folha (CPRB)
Lei 12.546/2011 e 13.161/2015: opção por **4,5% sobre a receita bruta** em vez de 20%
sobre a folha. Na base, reflete-se no componente **A1 (Previdência): 20% → 10%**, gerando
as tabelas `com_desoneracao` (Previdência 10%) e `sem_desoneracao` (Previdência 20%).

---

## 5. FIC — Fator de Influência de Chuvas

```
FIC = 1 + (fa × fp × fe × nd)
```

| Parâmetro | Significado | Tabela |
|---|---|---|
| `fa` | Natureza da atividade (sensibilidade à chuva) | 0,25 / 0,50 / 1,00 / 1,50 |
| `fp` | Permeabilidade do solo | areia 0,50 → argila 1,00 |
| `fe` | Escoamento superficial (declividade) | ≤1% → 1,00; 1–5% → 0,90; ≥5% → 0,80 |
| `nd` | Intensidade de chuvas (por UF) | AM 0,05334; AC 0,03145; PA 0,04583… |

Valores completos em `constantes_metodologia.json`. **A base AM 01/2026 já traz o FIC
aplicado** por composição (campo `fic`, 9 faixas entre 0,0559 e 0,2822).

---

## 6. FIT — Fator de Interferência de Tráfego

Aplica-se a obras que interditam pista ou exigem medidas de segurança (restauração,
3ª faixa, duplicação em pista contígua, conservação na pista). É função do **volume médio
diário de tráfego** e da presença de centros urbanos (Gráfico 1 do Manual, Vol. 10).
Substituiu a antiga distinção construção (fe=0,83) × restauração (fe=0,75).

---

## 7. Transporte

- **Tempo fixo (seção E):** custo de carga, descarga e manobras.
- **Momento de transporte (seção F):** volume transportado × distância média (DMT),
  considerando 3 tipos de pavimento: rodovia pavimentada (RP), leito natural (LN),
  revestimento primário (P).
- **Preços do SICRO não incluem frete** — pesquisar localmente e aplicar as composições
  de momento de transporte. Distinção CIF/FOB conforme contrato.

---

## 8. BDI — Benefícios e Despesas Indiretas

```
BDI(%) = ( (1 + ΣCD) / (1 − ΣPV) − 1 ) × 100
PV = CD × (1 + BDI/100)
```

- **ΣCD** = parcelas sobre o Custo Direto (Administração Central, Risco, Despesas Financeiras).
- **ΣPV** = parcelas sobre o Preço de Venda (Lucro, Tributos, Seguros/Garantias).

**Administração Central (% sobre CD):**

| Natureza da obra | % |
|---|---|
| Construção rodoviária | 6,0 |
| Restauração rodoviária | 6,0 |
| Conservação rodoviária | 9,0 |
| Construção de OAE | 8,0 |
| Recuperação/reforço/alargamento de OAE | 9,0 |
| Construção ferroviária | 6,0 |
| Obras hidroviárias | 7,0 |

**Despesas Financeiras (mensal):** `DF = (1 + SELIC)^(1/2) − 1`

**Tributos:** PIS 0,65% · COFINS 3,0% (cumulativo) ou 7,6% (não cumulativo) · ISSQN
municipal (usar alíquota do(s) município(s) da obra; SICRO usa referencial — ex. ponderado
BR-280/SC = 3,64%).

> **Administração Local NÃO integra o BDI** (Acórdão 2.622/2013-TCU) — orçar à parte (Vol. 08).
> Lucro, Risco e Seguros variam por porte/obra (matriz de gerenciamento de riscos).

---

## 9. Reajustamento de preços

```
R = V × ( (Ii / I0) − 1 )
```
R = valor do reajuste · V = parcela a preços iniciais · Ii = índice no mês do reajuste ·
I0 = índice no mês-base. (Instrução de Serviço DG/DNIT nº 01/2019.)

---

## 10. Base legal essencial

- **Decreto 7.983/2013** — custo global de referência a partir das composições do SICRO.
- **Lei 14.133/2021** — Nova Lei de Licitações.
- **IN nº 44/2021** — Preços Novos (serviço fora do SICRO/SINAPI).
- **Acórdão 2.622/2013-TCU** — Administração Local fora do BDI.
- **Portaria 434/2017** — transporte fluvial de asfalto.

---

*Consolidado dos Módulos 1–5 do curso "Introdução à Orçamentação de Obras Rodoviárias" (DNIT).
Para o conteúdo didático integral, ver `Modulos 1 - 5.md` na pasta-mãe.*
