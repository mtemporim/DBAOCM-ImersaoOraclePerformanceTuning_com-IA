# Desafio 6 — CBO: Cost-Based Optimizer & Estatísticas

**Módulo:** 6 — O optimizer está tomando decisões corretas?  
**Data:** 2026-06-14  
**Banco:** Oracle 23ai (23.26.1.0.0) — PDB `ORCLPDB` / host `ol9-orcl26-ribas`  
**Workload:** Swingbench SOE + plant.sql (stats desatualizadas + lock + job com bad plan)

---

## Objetivo

Entender como o CBO (Cost-Based Optimizer) toma decisões de plano de execução e o que acontece quando as **estatísticas estão desatualizadas ou sem histograma**. Cobre:

| Ferramenta | O que mostra |
|---|---|
| `V$SYS_TIME_MODEL` | Ponto de entrada — sql execute dominant → problema é num SQL específico |
| `DBA_TAB_STATISTICS` | `num_rows`, `stale_stats`, `stattype_locked` — visão do optimizer sobre a tabela |
| `DBA_TAB_COL_STATISTICS` | `histogram`, `num_buckets` — visibilidade de distribuição por coluna |
| `DBMS_XPLAN ALLSTATS LAST` | E-Rows (estimado) vs A-Rows (real) — revela o cardinality miss |
| `DBMS_STATS.GATHER_TABLE_STATS` | Re-coleta com histograma SIZE AUTO — corrige a visão do optimizer |

---

## Fluxo de diagnóstico utilizado

```
oracle-time-model
    └── snapshot 1 (job falhando): parse 74% — DB time negligível (0,08s/70s)
        └── job recriado com DBMS_SESSION.SLEEP
    └── snapshot 2 (job rodando): sql execute 99,9% + DB CPU 98,2%
        └── DB CPU > 50% → auto-encadeia → oracle-optimizer
                └── DBA_TAB_STATISTICS → stale=YES, locked=ALL, num_rows=10k
                └── V$SQL → dpgffusttabzx · TABLE ACCESS FULL · E-Rows=1110 vs A-Rows=202k
                └── DBA_TAB_COL_STATISTICS → STATUS sem histograma (NONE)
                └── Fix: UNLOCK + GATHER SIZE AUTO → FREQUENCY histogram em STATUS
                └── Após: E-Rows=201k ≈ A-Rows=202k · divergência ~0%
```

---

## Passo 0 — Time Model: dois snapshots

### Snapshot 1 — job falhando (DBMS_LOCK.SLEEP sem privilégio)

| Componente | Δ seg | % DB time |
|---|---:|---:|
| DB time | 0,08 | 100,0% |
| parse time elapsed | 0,06 | 73,6% |
| hard parse elapsed time | 0,05 | 69,7% |
| PL/SQL compilation elapsed time | 0,01 | 7,6% |

**Diagnóstico:** parse alto + PL/SQL compilation = job tentando rodar e falhando. DB time negligível (0,08s em 70s). **Ação corretiva:** recriar job com `DBMS_SESSION.SLEEP` no lugar de `DBMS_LOCK.SLEEP`.

### Snapshot 2 — job rodando corretamente

| Componente | Δ seg | % DB time |
|---|---:|---:|
| DB time | 6,35 | 100,0% |
| **sql execute elapsed time** | **6,34** | **99,9%** |
| DB CPU | 6,23 | 98,2% |
| PL/SQL execution elapsed time | 0,01 | 0,1% |
| parse time elapsed | ~0,00 | ~0% |

**Decisão de auto-encadeamento:** `DB CPU > 50%` do DB time → invocar `oracle-optimizer`. SQL execute dominante com CPU puro = optimizer está varrendo dados em excesso.

---

## Passo 1 — Stats desatualizadas e travadas

```sql
SELECT owner, table_name, num_rows, last_analyzed, stale_stats, stattype_locked
FROM dba_tab_statistics
WHERE owner = 'C##CLAUDE' AND table_name = 'LAB6_ORDERS';
```

| owner | table_name | num_rows | stale_stats | stattype_locked |
|---|---|---:|---|---|
| C##CLAUDE | LAB6_ORDERS | **10.000** | **YES** | **ALL** |

**Stats marcadas como stale, travadas (ALL), e num_rows 101x menor que a realidade (1.010.000).**

---

## Passo 2 — Plano com stats erradas (ANTES)

```sql
-- SQL_ID dpgffusttabzx — query do job LAB6_BAD_PLAN
SELECT /*+ GATHER_PLAN_STATISTICS */ SUM(amount)
FROM c##claude.lab6_orders
WHERE status = 'PENDING'
  AND order_date > SYSDATE - 10;
```

**Plano capturado via `DBMS_XPLAN.DISPLAY_CURSOR ALLSTATS LAST`:**

```
| Id | Operation          | Name        | E-Rows |  A-Rows | Buffers |
|  2 | TABLE ACCESS FULL  | LAB6_ORDERS |  1.110 | 202.000 |   4.941 |
```

**Divergência de cardinality: 182x**
- Optimizer estimou **1.110 linhas** (10.000 × 1/3 status × 1/3 data)
- Processou **202.000 linhas** — 182x mais do que o esperado

---

## Análise da distribuição de STATUS

Antes de coletar stats, verificar a distribuição real da coluna filtrada:

```sql
SELECT status, COUNT(*) AS linhas,
       ROUND(100 * COUNT(*) / SUM(COUNT(*)) OVER(), 0) AS pct_real
FROM c##claude.lab6_orders
GROUP BY status ORDER BY linhas DESC;
```

| STATUS | Linhas | % real | % assumida sem histograma |
|---|---:|---:|---:|
| PENDING | 606.000 | **60%** | 33,3% |
| CANCELLED | 202.000 | 20% | 33,3% |
| SHIPPED | 202.000 | 20% | 33,3% |

**Histograma em STATUS: NONE (1 bucket)** — optimizer trata os 3 valores como uniformes.  
Distribuição real é assimétrica (60/20/20). O CBO subestima PENDING em 27 p.p.

Com 3 valores distintos, Oracle vai gerar **histograma de frequência** (exato).

---

## Fix — Re-coletar estatísticas

```sql
-- 1) Desbloquear
EXEC DBMS_STATS.UNLOCK_TABLE_STATS('C##CLAUDE', 'LAB6_ORDERS');

-- 2) Re-coletar com amostragem automática e histogramas
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname          => 'C##CLAUDE',
        tabname          => 'LAB6_ORDERS',
        estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
        method_opt       => 'FOR ALL COLUMNS SIZE AUTO',
        cascade          => TRUE
    );
END;
```

**Resultado:**

| Métrica | Antes | Depois |
|---|---|---|
| num_rows | 10.000 | **1.010.000** |
| stale_stats | YES | **NO** |
| stattype_locked | ALL | **desbloqueado** |
| Histograma STATUS | NONE (1 bucket) | **FREQUENCY (3 buckets)** |
| Histograma ORDER_DATE | NONE | NONE (uniforme — Oracle não gerou) |

---

## Passo 3 — Plano com stats corretas (DEPOIS)

```
| Id | Operation          | Name        | E-Rows |  A-Rows | Buffers |
|  2 | TABLE ACCESS FULL  | LAB6_ORDERS | 201.000| 202.000 |   4.941 |
```

**Divergência: ~0%** — E-Rows ≈ A-Rows. O optimizer agora enxerga a realidade.

**Insight crítico:** o plano não mudou (TABLE ACCESS FULL). Mas agora está **correto por decisão**, não por acaso:
- PENDING = 60% da tabela → full scan é a operação correta
- Se a query fosse `WHERE status = 'SHIPPED'` (20%), o optimizer poderia escolher index scan

**Bônus:** `cursor_sharing=FORCE` (ativo desde Desafio 5) normalizou os literais para `:SYS_B_0` e `:SYS_B_1`. O histograma funcionou via **bind variable peeking** na primeira execução após re-coleta.

---

## Findings e Recomendações

### Finding 1 — Stats travadas e desatualizadas (CRÍTICO · esforço BAIXO · risco BAIXO)

**Evidência:** `stale_stats=YES` · `stattype_locked=ALL` · `num_rows=10.000` vs realidade de 1.010.000 → divergência de 101x no volume.  
**Causa:** inserção massiva após coleta + `LOCK_TABLE_STATS` impedindo auto-collect.  
**Ação:** `UNLOCK_TABLE_STATS` + `GATHER_TABLE_STATS`. Em produção: revisar locks de stats e garantir que jobs de auto-collect alcançam as tabelas críticas.

### Finding 2 — Cardinality miss de 182x (CRÍTICO em queries com JOIN)

**Evidência:** E-Rows=1.110 vs A-Rows=202.000.  
**Impacto real:** em queries simples com aggregate o plano se mantém. Em queries com JOIN, o optimizer escolhe NESTED LOOP onde HASH JOIN seria melhor, subestima memória para sort, define grau de paralelismo errado.  
**Ação:** re-coletar stats. Considerar histogramas em colunas com distribuição não uniforme.

### Finding 3 — STATUS sem histograma (ALTO · esforço BAIXO)

**Evidência:** `histogram=NONE` · distribuição real 60/20/20 vs assumida 33/33/33.  
**Impacto:** optimizer subestima PENDING em 27 p.p. Para queries filtrando SHIPPED/CANCELLED, poderia fazer full scan quando index scan seria melhor.  
**Fix:** `method_opt => 'FOR ALL COLUMNS SIZE AUTO'` — Oracle gerou FREQUENCY histogram (3 buckets exatos).

### Finding 4 — LOCK_TABLE_STATS em produção (MÉDIO)

**Quando usar:** tabelas de lookup pequenas e estáticas, partições históricas imutáveis.  
**Quando evitar:** tabelas transacionais com crescimento contínuo.  
**Ação:** revisar `DBA_TAB_STATISTICS WHERE stattype_locked IS NOT NULL`.

### Finding 5 — job falhando silenciosamente (OBSERVAR)

**Evidência:** `LAB6_BAD_PLAN` falhou 3x com ORA-01031 (`DBMS_LOCK.SLEEP` sem privilégio) e foi auto-dropped.  
**Sinal no Time Model:** parse + PL/SQL compilation altos com DB time negligível.  
**Fix:** substituir `DBMS_LOCK.SLEEP` por `DBMS_SESSION.SLEEP` (disponível para PUBLIC).

---

## Padrão de skills utilizado

| Skill | Papel | Tipo de execução |
|---|---|---|
| `oracle-time-model` | Ponto de entrada — dois snapshots, segundo revela sql execute 99,9% | Agente principal |
| `oracle-optimizer` | Investigação — stats, histograma, E-Rows vs A-Rows, fix | Agente principal |
| `oracle-sql-diagnosis` | **Não utilizado** — pulado pois o SQL ofensor era conhecido (contexto do lab) | — |
| `subagent-*` | **Não utilizados** — todas as queries retornaram resultados pequenos | — |

---

## Lição Central

O CBO é tão bom quanto as estatísticas que recebe. Um optimizer com stats 101x erradas produz planos errados — e o sinal não aparece em I/O nem em waits óbvios. Aparece em `DBMS_XPLAN ALLSTATS LAST` como `E-Rows << A-Rows`. Essa comparação é o diagnóstico mais rápido de problema de CBO.

Com histograma de frequência em STATUS, o optimizer passou de "cada valor vale 33%" para "PENDING vale 60% exato". Essa precisão é o que garante planos corretos em queries com filtros seletivos ou em JOINs com múltiplas tabelas.

---

## Evidências

| Artefato | Valor |
|---|---|
| Time Model — sql execute elapsed (snapshot 2) | 99,9% do DB time / DB CPU 98,2% |
| SQL_ID ofensor | `dpgffusttabzx` — plan_hash 18767498 |
| E-Rows antes | 1.110 (stats velhas, sem histograma) |
| A-Rows real | 202.000 |
| Divergência | **182x** |
| num_rows antes/depois | 10.000 → 1.010.000 |
| Histograma STATUS | NONE → FREQUENCY (3 buckets) |
| E-Rows após fix | 201.000 — divergência ~0% |
| Relatório HTML | `HTML/Desafio6/optimizer_report.html` |
