# Desafio 9 — Cost Reduction: Chaining, HWM & Compressão

**Data:** 2026-06-14
**Banco:** Oracle 23ai (23.26.1.0.0) — PDB ORCLPDB
**Host:** ol9-orcl26-ribas
**Schemas analisados:** C##CLAUDE + SOE
**Workload base:** Swingbench SOE + 3 objetos plantados (LAB9_CHAINED · LAB9_HWM · LAB9_COMPRESS)

---

## Módulo ativado

Este módulo não parte do Time Model — o diagnóstico é orientado por **padrões de armazenamento**, não por onde o DB time está sendo gasto. Os três sintomas de entrada são:

| Sintoma | View diagnóstica | Ação |
|---|---|---|
| `chain_cnt > 0` em DBA_TABLES | `DBA_TABLES` | MOVE TABLE + fix PCTFREE |
| `blocks × 8 KB` >> `num_rows × avg_row_len` | `DBA_TABLES` | SHRINK SPACE |
| Segmento grande com COMPRESSION=DISABLED | `DBA_SEGMENTS + DBA_TABLES` | MOVE ROW STORE COMPRESS ADVANCED |

---

## Passo 1 — Row Chaining / Migration

**Fonte:** `DBA_TABLES.chain_cnt` + `ANALYZE TABLE ... COMPUTE STATISTICS`

| Owner | Tabela | Linhas | Encadeadas | % Chained | Alocado | PCTFREE |
|---|---|---:|---:|---:|---:|---:|
| C##CLAUDE | LAB9_CHAINED | 50.000 | **24.999** | **50,0%** | 103 MB | **0** |

Nenhuma tabela SOE apresentou `chain_cnt > 0` — schema em boa forma para chaining.

**Causa raiz:** `PCTFREE 0` não reserva espaço dentro do bloco para crescimento da linha. O UPDATE posterior (3.500 chars de payload) não coube no bloco original → a linha migrou para outro bloco, deixando um ponteiro no bloco de origem. Cada SELECT que toca essa linha agora exige **2 I/Os** em vez de 1.

**Fix:**

```sql
ALTER TABLE c##claude.lab9_chained MOVE;
-- Rebuild obrigatório de todos os índices após MOVE
ALTER INDEX c##claude.sys_c... REBUILD;
-- Aumentar PCTFREE preventivamente:
ALTER TABLE c##claude.lab9_chained PCTFREE 20;
```

**Nota diagnóstica:** `empty_blocks` em DBA_TABLES **não detecta** chaining — é preciso executar `ANALYZE TABLE ... COMPUTE STATISTICS` para popular `chain_cnt`. O `DBMS_STATS.GATHER_TABLE_STATS` padrão não o faz.

---

## Passo 2 — HWM Alto (blocos alocados vs dado real)

**Fonte:** `DBA_TABLES` — cálculo `num_rows × avg_row_len` vs `blocks × 8 KB`

> **Método correto:** blocos liberados por DELETE ficam **abaixo** do HWM — não aparecem em `empty_blocks` (esse campo só mostra blocos acima do HWM, nunca usados). A métrica correta é comparar o dado real estimado com o espaço alocado total.

| Owner | Tabela | Alocado | Dado est. | % Waste | Linhas |
|---|---|---:|---:|---:|---:|
| SOE | **INVENTORIES** | 176 MB | 12 MB | **93%** | 898.776 |
| C##CLAUDE | **LAB9_HWM** | 116 MB | 19 MB | **83%** | 100.000 |
| SOE | CARD_DETAILS | 31 MB | 19 MB | 39% | 444.469 |
| SOE | LOGON | 39 MB | 24 MB | 38% | 1.418.787 |
| SOE | ORDER_ITEMS | 127 MB | 93 MB | 26% | 2.135.326 |
| SOE | ADDRESSES | 43 MB | 33 MB | 22% | 477.207 |
| SOE | ORDERS | 79 MB | 62 MB | 21% | 878.545 |

**Destaque crítico — SOE.INVENTORIES (93% waste):**
176 MB alocados para apenas 12 MB de dado real. O Swingbench realiza operações massivas de UPDATE em INVENTORIES (`order_qty`) sem DELETEs balanceados — o HWM subiu e nunca foi resetado. Isso tem impacto direto: full scans varrem **176 MB de blocos**, dos quais 93% estão vazios.

**LAB9_HWM:** confirma o cenário plantado — DELETE de 80% das linhas sem SHRINK deixou 116 MB alocados para 19 MB de dado.

**Fix:**

```sql
-- INVENTORIES (SOE) — shrink online
ALTER TABLE soe.inventories ENABLE ROW MOVEMENT;
ALTER TABLE soe.inventories SHRINK SPACE COMPACT;
ALTER TABLE soe.inventories SHRINK SPACE;
ALTER TABLE soe.inventories DISABLE ROW MOVEMENT;

-- LAB9_HWM — mesmo procedimento
ALTER TABLE c##claude.lab9_hwm ENABLE ROW MOVEMENT;
ALTER TABLE c##claude.lab9_hwm SHRINK SPACE COMPACT;
ALTER TABLE c##claude.lab9_hwm SHRINK SPACE;
ALTER TABLE c##claude.lab9_hwm DISABLE ROW MOVEMENT;
```

---

## Passo 3 — Candidatos à Compressão

**Fonte:** `DBA_SEGMENTS` + `DBA_TABLES.compression`

Todos os segmentos analisados têm `COMPRESSION = DISABLED`. Tipo recomendado: **ROW STORE COMPRESS ADVANCED** — suporta DML normal (OLTP), ratio estimado 2–3×.

| # | Owner | Tabela | Alocado | Linhas | Ganho est. (2×) | Esforço |
|---|---|---|---:|---:|---:|---|
| 1 | C##CLAUDE | LAB9_COMPRESS | 362 MB | 1.000.000 | ~181 MB | Baixo |
| 2 | SOE | INVENTORIES | 177 MB | 898.776 | ~89 MB | Baixo\* |
| 3 | SOE | ORDER_ITEMS | 157 MB | 2.135.326 | ~79 MB | Médio |
| 4 | C##CLAUDE | LAB9_HWM | 120 MB | 100.000 | ~60 MB | Baixo\* |
| 5 | SOE | ORDERS | 104 MB | 878.545 | ~52 MB | Médio |
| 6 | C##CLAUDE | LAB9_CHAINED | 104 MB | 50.000 | ~52 MB | Médio\* |
| 7 | SOE | CUSTOMERS | 79 MB | 394.469 | ~40 MB | Baixo |
| 8 | SOE | ADDRESSES | 60 MB | 477.207 | ~30 MB | Baixo |
| 9 | SOE | LOGON | 56 MB | 1.418.787 | ~28 MB | Baixo |
| 10 | SOE | CARD_DETAILS | 40 MB | 444.469 | ~20 MB | Baixo |

\* INVENTORIES e LAB9_HWM: fazer SHRINK **antes** de comprimir (não faz sentido comprimir espaço vazio).
\* LAB9_CHAINED: fazer MOVE para corrigir chaining — o próprio MOVE pode já incluir a compressão.

**LAB9_COMPRESS** tem payload 100% repetido (`RPAD('repeated_payload_AAAA', 300, 'A')`) — ratio esperado 3–4× neste caso específico.

**Fix:**

```sql
-- Compressão de LAB9_COMPRESS (sem chaining prévio — apenas comprimir)
ALTER TABLE c##claude.lab9_compress MOVE ROW STORE COMPRESS ADVANCED;

-- Corrigir chaining e comprimir em um único MOVE
ALTER TABLE c##claude.lab9_chained MOVE ROW STORE COMPRESS ADVANCED;
ALTER INDEX c##claude.sys_c... REBUILD;

-- SOE (OLTP ativo — ADVANCED tolera DML, mínimo overhead)
ALTER TABLE soe.inventories  MOVE ROW STORE COMPRESS ADVANCED;
ALTER TABLE soe.order_items  MOVE ROW STORE COMPRESS ADVANCED;
ALTER TABLE soe.orders       MOVE ROW STORE COMPRESS ADVANCED;
-- Rebuild de índices após cada MOVE
```

---

## Priorização por Impacto

| # | Ação | Objeto | Ganho | Esforço | Risco |
|---|---|---|---|---|---|
| 1 | SHRINK imediato | SOE.INVENTORIES | **~164 MB + menos buffer pressure** | Baixo | Baixo |
| 2 | SHRINK | C##CLAUDE.LAB9_HWM | ~97 MB recuperados | Baixo | Baixo |
| 3 | MOVE + COMPRESS ADVANCED | C##CLAUDE.LAB9_COMPRESS | ~181–270 MB (payload repetido) | Baixo | Baixo |
| 4 | MOVE + COMPRESS ADVANCED | SOE.ORDER_ITEMS + ORDERS | ~130 MB | Médio | Médio |
| 5 | MOVE + fix PCTFREE 20 | C##CLAUDE.LAB9_CHAINED | Elimina 24.999 linhas encadeadas | Médio | Médio |

**Potencial total** de liberação no schema SOE: **~280 MB** com SHRINK + ~270 MB com compressão.

---

## Nota técnica — Detecção correta de HWM

A view `DBA_TABLES.empty_blocks` conta apenas blocos **acima** do HWM (nunca alocados). Blocos liberados por DELETE ficam **abaixo** do HWM e são invisíveis nessa coluna. A métrica correta:

```sql
ROUND(100 * (1 - (num_rows * avg_row_len) / NULLIF(blocks * 8192, 0)), 1) AS pct_waste
```

Threshold operacional: `pct_waste > 20%` com mais de 50 blocos merece avaliação de SHRINK.

---

## Findings & Recomendações

| # | Finding | Impacto | Esforço | Risco |
|---|---|---|---|---|
| F1 | LAB9_CHAINED: 50% de linhas encadeadas (PCTFREE 0) | **Alto** (2 I/Os por linha) | Médio | Médio |
| F2 | SOE.INVENTORIES: 93% waste — 176 MB para 12 MB de dado | **Crítico** (full scans desnecessários) | Baixo | Baixo |
| F3 | LAB9_HWM: 83% waste — DELETE massivo sem SHRINK | Alto | Baixo | Baixo |
| F4 | 10 tabelas COMPRESSION=DISABLED — potencial 2–3× ratio | Médio | Baixo/Médio | Baixo |
| F5 | `empty_blocks` não detecta HWM real — query de waste correta | Metodológico | — | — |

---

## Skills Utilizadas

| Skill | Utilizada | Motivo |
|---|---|---|
| oracle-time-model | Não | Módulo orientado por padrões de storage, não por DB time ao vivo |
| oracle-cost-reduction | Sim | Análise de chaining, HWM e candidatos à compressão |
| subagent-space | Sim | Varredura de DBA_SEGMENTS + DBA_TABLES nos dois schemas |
| oracle-buffer-io | Complementar | INVENTORIES domina buffer cache — correlação com Desafio 7 |
