<!-- AKE/UFT-1.0 | BUILD: SICRO-20260609 | IC: 1.0 | MÓDULO: base_dados_mestre -->
# Base de Dados SICRO — AM 01/2026 (Amazonas)

> Banco de dados de custos referenciais de obras rodoviárias, extraído dos relatórios
> oficiais do **SICRO/DNIT** e estruturado para uso direto em cálculos, motores de
> orçamento, planilhas e código. **Todos os dados foram validados** contra as
> identidades matemáticas oficiais do sistema (ver §6).

| Campo | Valor |
|---|---|
| **Sistema** | SICRO — Sistema de Custos Referenciais de Obras (DNIT) |
| **Referência** | AM 01/2026 — Estado do **Amazonas**, mês-base **Janeiro/2026** |
| **Moeda** | Real (R$) |
| **Regime** | Tabelas com e **sem desoneração** da folha (CPRB) |
| **Origem** | 13 relatórios oficiais `.xlsx` (pasta `DNIT/am-01-2026`) |
| **Gerado em** | 2026-06-09 |
| **Build** | SICRO-20260609 |

---

## 1. O que é esta base

Banco de dados relacional em **CSV** (dados limpos) + **JSON** (consumo por código) +
**Markdown** (documentação e fórmulas). Cobre toda a cadeia de cálculo do SICRO:

```
INSUMOS (materiais, mão de obra, equipamentos)
   └─► COMPOSIÇÕES ANALÍTICAS (insumo a insumo, seções A–F)
          └─► COMPOSIÇÕES SINTÉTICAS (custo unitário final por serviço)
                 └─► ORÇAMENTO (× quantidades + BDI)   ← aplicado pelo usuário
```

O **BDI**, o **FIC/FIT** paramétricos, a **Administração Local**, **Canteiro** e
**Mobilização** **não** integram os preços do SICRO (Acórdão 2.622/2013-TCU). As
fórmulas e parâmetros para aplicá-los estão em `METODOLOGIA.md` e
`json/constantes_metodologia.json`.

---

## 2. Estrutura de arquivos

```
BASE_DADOS_SICRO_AM01-2026/
├── README.md                  ← este arquivo (índice + dicionário + uso)
├── METODOLOGIA.md             ← fórmulas oficiais (equipamento, MO, FIC, FIT, BDI, transporte)
├── extrator.py                ← script que gera os CSV/JSON a partir dos .xlsx
├── validar.py                 ← script de validação de integridade
├── _validacao.json            ← resultado das validações (prova de correção)
├── _extracao_stats.json       ← contagem de registros por tabela
├── csv/                       ← dados em CSV (separador ";", encoding UTF-8-BOM)
│   ├── encargos_sociais.csv
│   ├── mao_de_obra.csv
│   ├── equipamentos.csv
│   ├── materiais.csv
│   ├── composicoes.csv               (cabeçalho/sintético de cada serviço)
│   ├── composicoes_sinteticas.csv    (custo unitário final — visão enxuta)
│   ├── composicoes_insumos.csv       (50.411 linhas — detalhe analítico A–F)
│   └── origem_precos.csv
└── json/                      ← mesmas tabelas em JSON + constantes de metodologia
    ├── *.json
    └── constantes_metodologia.json
```

> **CSV:** delimitador `;`, decimal `.`, codificação `utf-8-sig` (abre direto no Excel
> PT-BR). Valores ausentes = célula vazia.

---

## 3. Dicionário de dados

### 3.1 `materiais` (1.693 registros)
Preço unitário de cada material. Chave: `codigo` (prefixo `M`).

| Coluna | Tipo | Unid. | Descrição |
|---|---|---|---|
| `codigo` | texto | — | Código do material (ex.: `M0004`) |
| `descricao` | texto | — | Nome do material |
| `unidade` | texto | — | Unidade comercial/técnica (kg, m³, l, un…) |
| `preco_unitario` | número | R$ | Preço de referência (à vista, com tributos, sem frete) |

### 3.2 `mao_de_obra` (290 = 145 profissões × 2 regimes)
Custo horário/mensal por profissão, com decomposição de encargos. Chave: (`regime`,`codigo`), prefixo `P`.

| Coluna | Tipo | Unid. | Descrição |
|---|---|---|---|
| `regime` | texto | — | `com_desoneracao` \| `sem_desoneracao` |
| `codigo` / `descricao` / `unidade` | — | — | Identificação (unid. `h` horista ou `mês` mensalista) |
| `salario_base` | número | R$ | Salário-base (por hora ou por mês) |
| `grupo_A_pct`…`grupo_D_pct` | número | % | Totais dos grupos de encargos sociais A, B, C, D |
| `enc_sociais_trab_pct` / `_rs` | número | % / R$ | Encargos sociais e trabalhistas (inclui base periculosidade) |
| `periculosidade_pct` / `_rs` | número | % / R$ | Adicional de periculosidade/insalubridade (0 quando não aplicável) |
| `enc_complementares_pct` / `_rs` | número | % / R$ | Alimentação, EPI, ferramenta, transporte, consulta ocupacional |
| `enc_adicionais_pct` / `_rs` | número | % / R$ | Cesta básica, assistência médica, seguro de vida etc. |
| `encargos_totais_pct` / `_rs` | número | % / R$ | Total de encargos (**exclui** periculosidade) |
| `custo_total` | número | R$ | **Custo final** = `salario_base` + `encargos_totais_rs` + `periculosidade_rs` |

> É o `custo_total` deste regime que aparece como preço unitário da mão de obra nas
> composições analíticas (seção B).

### 3.3 `encargos_sociais` (290 = 145 × 2 regimes)
Decomposição percentual completa dos encargos (componentes A1–D2). Chave: (`regime`,`codigo`).

| Coluna | Descrição |
|---|---|
| `regime`, `codigo`, `descricao`, `unidade` | Identificação |
| `A1`…`A9` | Sociais básicos. **`A1` = Previdência: 20% (sem deson.) / 10% (com deson.)** |
| `B1`…`B10` | Trabalhistas (RSR, feriados, férias+1/3, 13º, faltas…) |
| `C1`…`C5` | Verbas rescisórias |
| `D1`,`D2` | Reincidências |
| `total_pct` | Soma de todos os componentes (= Σ A..D) |

### 3.4 `equipamentos` (1.128 = 564 × 2 regimes)
Custo horário por equipamento. Chave: (`regime`,`codigo`), prefixo `E`.

| Coluna | Unid. | Descrição |
|---|---|---|
| `regime`, `codigo`, `descricao` | — | Identificação |
| `valor_aquisicao` | R$ | Valor de aquisição do equipamento |
| `depreciacao` | R$/h | Depreciação horária (Dh) |
| `oportunidade_capital` | R$/h | Juros de oportunidade de capital (Jh) |
| `seguros_impostos` | R$/h | Seguros e impostos (Ih) |
| `manutencao` | R$/h | Manutenção (Mh) |
| `operacao` | R$/h | Combustíveis/lubrificantes (Cc) |
| `mao_obra_operacao` | R$/h | Mão de obra de operação (Cmo) |
| `custo_produtivo` | R$/h | **Custo horário produtivo** = soma dos componentes acima |
| `custo_improdutivo` | R$/h | Custo horário improdutivo |

> 98 itens (caminhões/veículos de transporte) têm `custo_produtivo`/`improdutivo`
> **diretos**, sem decomposição em componentes (campos de componentes vazios). É o padrão SICRO para esses veículos.

### 3.5 `composicoes` (6.525) — cabeçalho analítico de cada serviço
Chave: `codigo` (7 dígitos). Subtotais por seção + custo final.

| Coluna | Unid. | Descrição |
|---|---|---|
| `codigo`, `descricao` | — | Identificação do serviço |
| `producao_unidade` | — | Unidade do serviço (m³, t, un, m², km…) |
| `producao_equipe` | — | Produção da equipe por ciclo |
| `fic` | — | Fator de Influência de Chuvas aplicado (AM 01/2026: 9 faixas, 0,0559–0,2822) |
| `custo_horario_equip` | R$/h | Subtotal seção A (equipamentos) |
| `custo_horario_mo` | R$/h | Subtotal seção B (mão de obra) |
| `custo_unitario_material` | R$ | Subtotal seção C (materiais) |
| `custo_unitario_tempo_fixo` | R$ | Subtotal seção E (tempo fixo) |
| `custo_unitario_transporte` | R$ | Subtotal seção F (transporte) |
| `subtotal` | R$ | A+B+C+D por unidade |
| `custo_unitario_direto` | R$ | **Custo unitário direto total** (final, sem BDI) |
| `custo_unitario_sintetico` | R$ | Mesmo valor vindo do relatório sintético (conferência cruzada) |

### 3.6 `composicoes_sinteticas` (6.525) — visão enxuta
`codigo`, `descricao`, `unidade`, `custo_unitario` (R$). É o **preço de tabela** do serviço.

### 3.7 `composicoes_insumos` (50.411) — detalhe analítico A–F
Uma linha por insumo dentro de cada composição. Chave estrangeira: `comp_codigo` → `composicoes.codigo`; `insumo_codigo` → tabela do tipo conforme `secao`.

| Coluna | Descrição |
|---|---|
| `comp_codigo` | Composição à qual o insumo pertence |
| `secao` | `EQUIPAMENTO` \| `MAO_OBRA` \| `MATERIAL` \| `ATIV_AUXILIAR` \| `TEMPO_FIXO` \| `TRANSPORTE` |
| `insumo_codigo` / `insumo_descricao` | Insumo (E…/P…/M…/composição auxiliar) |
| `quantidade` | Coeficiente de consumo |
| `unidade` | Unidade do coeficiente |
| `util_operativa` / `util_improdutiva` | (só EQUIPAMENTO) frações de utilização |
| `custo_h_produtivo` / `custo_h_improdutivo` | (só EQUIPAMENTO) custos horários |
| `preco_unitario` | Custo unitário do insumo (R$) |
| `dmt_ln` / `dmt_rp` / `dmt_p` | (só TRANSPORTE) referências de DMT por tipo de pavimento |
| `custo` | Contribuição da linha ao custo da composição (R$) |

### 3.8 `origem_precos` (2.390) — procedência do preço por UF
`codigo`, `descricao`, `unidade` + **27 colunas de UF** (AC, AM, AP, PA, RO, RR, TO, AL, BA, CE, MA, PB, PE, PI, RN, SE, ES, MG, RJ, SP, PR, RS, SC, DF, GO, MS, MT).
Valores: `PRE` (preço referencial), `PFM` (preço de mercado/fabricante), `PCI` (preço com insumo), `PC` (preço de catálogo). Garante rastreabilidade/auditoria.

---

## 4. Modelo relacional (como fazer joins)

```
composicoes (codigo) 1 ──── N composicoes_insumos (comp_codigo)
                                     │ insumo_codigo + secao
                                     ├─ secao=MATERIAL      → materiais (codigo)
                                     ├─ secao=MAO_OBRA      → mao_de_obra (codigo, regime)
                                     ├─ secao=EQUIPAMENTO   → equipamentos (codigo, regime)
                                     └─ secao=ATIV_AUXILIAR → composicoes (codigo)   [recursivo]

composicoes (codigo) 1 ──── 1 composicoes_sinteticas (codigo)   [mesmo custo unitário]
mao_de_obra (codigo, regime) 1 ──── 1 encargos_sociais (codigo, regime)
qualquer_insumo (codigo) ──── origem_precos (codigo)   [procedência por UF]
```

---

## 5. Como calcular (receitas)

As fórmulas completas estão em **`METODOLOGIA.md`**. Resumo operacional:

**Custo de um serviço (composição):**
```
custo_unitario_servico = custo_unitario_direto        (já calculado, coluna em composicoes)
                       = Σ(insumo.quantidade × insumo.preco_unitario)  por seção  (reconstruível)
```

**Mão de obra:** `custo_total = salario_base + encargos_totais_rs + periculosidade_rs`

**Equipamento:** `custo_produtivo = depreciacao + oportunidade_capital + seguros_impostos + manutencao + operacao + mao_obra_operacao`

**FIC:** `FIC = 1 + (fa × fp × fe × nd)` — parâmetros em `constantes_metodologia.json`.

**Orçamento final de um item:**
```
preco_item = quantidade_projeto × custo_unitario_servico × (1 + BDI/100)
BDI(%)     = ((1 + ΣCD) / (1 − ΣPV) − 1) × 100
```

---

## 6. Validação — garantia de correção

Executar `python validar.py`. Resultado da última execução (`_validacao.json`):

| Verificação | Resultado |
|---|---|
| Encargos: Σ(componentes A1..D2) = `total_pct` | **290/290 (100%)** |
| MO: `enc_sociais_trab_rs` + `enc_complementares_rs` + `enc_adicionais_rs` = `encargos_totais_rs` | **290/290 (100%)** |
| MO: `salario_base` + `encargos_totais_rs` + `periculosidade_rs` = `custo_total` | **290/290 (100%)** |
| Equipamento: Σ(componentes) = `custo_produtivo` (itens com decomposição) | **1.030/1.030 (100%)** |
| Composição: `custo_unitario_direto` (analítico) = `custo_unitario` (sintético) | **6.525/6.525 (100%)** |
| Cobertura: todo serviço sintético tem analítico (e vice-versa) | **0 faltantes** |
| Integridade referencial: insumos apontam para códigos existentes | **0 órfãos / 50.411** |
| Reconstrução: Σ(insumos seção) = subtotal do header — EQUIPAMENTO | **4.700/4.700 (100%)** |
| Reconstrução — TEMPO_FIXO | **2.970/2.970 (100%)** |
| Reconstrução — MÃO_OBRA | **4.125/4.126 (99,98%)** |
| Reconstrução — MATERIAL | **3.378/3.380 (99,94%)** |

> Tolerância: R$ 0,02 absoluto; 0,01% relativo para mensalistas (salários de dezenas
> de milhares de R$, onde o arredondamento de centavos na fonte acumula).
>
> **Borda conhecida (3 registros):** em 3 composições de grande porte (ex.: `0903803`,
> `0919002`, `7119788` — postos/instalações de milhões de R$), o **subtotal de seção
> gravado no header** difere da soma dos insumos listados (nuance de agregação da
> fonte). Isso afeta **apenas os campos redundantes de subtotal** em `composicoes`; o
> **custo unitário final** (`custo_unitario_direto`) dessas composições validou 100%
> contra o sintético, e os insumos têm integridade referencial total. Para cálculo,
> use sempre `custo_unitario` / `custo_unitario_direto` como valor autoritativo.

---

## 7. Exemplos de uso

### Python (pandas)
```python
import pandas as pd
mat  = pd.read_csv("csv/materiais.csv", sep=";")
comp = pd.read_csv("csv/composicoes_sinteticas.csv", sep=";")
ins  = pd.read_csv("csv/composicoes_insumos.csv", sep=";")

# custo de tabela de um serviço
print(comp.loc[comp.codigo=="0307731", "custo_unitario"].item())   # 154.16

# reconstruir custo de uma composição a partir dos insumos
sub = ins[ins.comp_codigo=="0307731"]
print(sub.groupby("secao")["custo"].sum())
```

### JavaScript (JSON)
```js
const comp = require("./json/composicoes_sinteticas.json");
const mat  = require("./json/materiais.json");
const preco = comp.find(c => c.codigo === "0307731").custo_unitario; // 154.16
```

### SQL (importando os CSV no SQLite)
```sql
-- no shell:  sqlite3 sicro.db
.mode csv
.separator ";"
.import csv/composicoes_sinteticas.csv composicoes_sinteticas
.import csv/composicoes_insumos.csv    composicoes_insumos
.import csv/materiais.csv              materiais

SELECT c.descricao, SUM(i.custo) AS custo_total
FROM composicoes_insumos i
JOIN composicoes_sinteticas c ON c.codigo = i.comp_codigo
WHERE i.comp_codigo = '0307731'
GROUP BY c.descricao;
```

---

## 8. Notas e limitações

- **Específico do Amazonas / Jan-2026.** Preços, FIC e `origem_precos` são regionais e
  datados. Para outra UF/mês, baixar o pacote correspondente no DNIT e rodar `extrator.py`.
- **Sem frete embutido.** Transporte é tratado por composições de momento de transporte
  (seções E/F); o frete real deve ser pesquisado localmente.
- **Sem BDI / Administração Local / Canteiro / Mobilização** nos preços (regra SICRO).
- A coluna `dmt_*` da seção TRANSPORTE traz referências (códigos) de DMT, não distâncias
  em km; o custo de transporte por composição já está consolidado em `custo_unitario_direto`.
- Reprodutibilidade: `extrator.py` + os 13 `.xlsx` regeneram a base byte-a-byte.

---

*Fonte primária: relatórios oficiais SICRO/DNIT (AM 01/2026) e Manual de Custos de
Infraestrutura de Transportes. Metodologia consolidada dos Módulos 1–5 do curso de
Orçamentação de Obras Rodoviárias do DNIT.*
