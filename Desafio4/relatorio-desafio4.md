# Desafio 4 — SQL Problem Identification & Execution Plans

**Módulo:** 4 — Encontrar qual SQL está com problema e por que o plano está ruim  
**Data:** 2026-06-13  
**Banco:** Oracle 23ai (23.26.1.0.0) — PDB `ORCLPDB` / host `ol9-orcl26-ribas`  
**Workload:** Swingbench SOE (New Order, Browse Products) + plant.sql (full scan plantado)

---

## Objetivo

Após identificar *onde* o tempo é gasto (Módulo 1) e *o que aconteceu na janela* (Módulo 3), o Módulo 4 desce um nível: **qual SQL específico está com problema e por que o plano é ruim?**

| Ferramenta | O que mostra |
|---|---|
| `V$SQL` | Top SQLs por elapsed, execuções, gets/exec — identifica o candidato |
| `DBMS_XPLAN.DISPLAY_CURSOR` | Plano **real** do cursor (não hipotético) com estatísticas de execução |
| SQL Trace 10046 + tkprof | Detalhamento por fase: parse / execute / fetch / waits por evento |

---

## O que o plant.sql plantou

```sql
-- Tabela com 500.000 linhas e SEM índice no campo de filtro
CREATE TABLE c##claude.lab4_orders (
    order_id    NUMBER,
    customer_id NUMBER,   -- campo filtrado, sem índice
    ...
);
INSERT /*+ APPEND */ ... CONNECT BY level <= 500000;
EXEC DBMS_STATS.GATHER_TABLE_STATS('C##CLAUDE','LAB4_ORDERS');

-- Job que martela full scan a cada 0,5s por 5 minutos (~600 execuções)
-- SELECT COUNT(*) FROM lab4_orders WHERE customer_id = :n
```

---

## Metodologia — Passo 1: Panorama V$SQL

A primeira ação é sempre rodar o diagnóstico **sobre todo o V$SQL**, não ir direto ao SQL suspeito:

```sql
SELECT sql_id, module, executions,
       ROUND(elapsed_time/NULLIF(executions,0)/1e6,3) AS elapsed_per_exec_s,
       ROUND(buffer_gets/NULLIF(executions,0))         AS gets_per_exec,
       SUBSTR(sql_text,1,60) AS texto
FROM v$sql
WHERE executions > 10
  AND elapsed_time > 5e5
  AND NVL(module,'-') NOT IN ('DBMS_SCHEDULER','dbms_stats: gather table stats')
ORDER BY elapsed_time DESC
FETCH FIRST 10 ROWS ONLY;
```

---

## Resultados — Panorama V$SQL (Swingbench + lab)

| SQL_ID | Módulo | Exec | Elapsed/exec | Gets/exec | Tipo | Texto |
|---|---|---:|---:|---:|---|---|
| `bvcnjjrdz9aab` | JDBC Thin Client | 8,6M | ~0ms | 12 | SQL interno | `select products.product_id, product_name...` |
| `3gnwwgbk8h6au` | JDBC Thin Client | 6,9M | ~0ms | 23 | SQL interno | `select products.PRODUCT_ID...` |
| **`34mt4skacwwwd`** | JDBC Thin Client | 72k | **21ms** | **1.040** | SQL interno | `WITH need_to_process AS (SELECT order_id...` |
| `09pzy8x10gjkg` | JDBC Thin Client | 1M | ~0ms | 15 | INSERT interno | `insert into order_items...` |
| `a6hdpzrqqhc7d` | JDBC Thin Client | 538k | ~1ms | 17 | INSERT interno | `insert into orders...` |
| `f3navkanm4s6k` | JDBC Thin Client | 1,6M | ~0ms | 3 | SQL interno | `SELECT CUSTOMER_ID, CUST_FIRST_NAME...` |

**SQL plantado (identificado separadamente por gets/exec):**

| SQL_ID | Módulo | Exec | Elapsed/exec | Gets/exec | Texto |
|---|---|---:|---:|---:|---|
| **`gujuujxvwzv48`** | DBMS_SCHEDULER | 310 | **156ms** | **16.093** | `SELECT COUNT(*) FROM LAB4_ORDERS WHERE CUSTOMER_ID = :B1` |

> **Interpretação:** A maioria dos SQLs Swingbench é eficiente (gets/exec baixo). O `34mt4skacwwwd` chama atenção com 1.040 gets/exec e 21ms/exec. O SQL plantado `gujuujxvwzv48` é o pior — 16.093 gets/exec por ter zero índices.

---

## Metodologia — Passo 2: Plano Real via DBMS_XPLAN

### Plano — `34mt4skacwwwd` (Swingbench — New Order flow)

```
Plan hash value: 235854103

-----------------------------------------------------------------------
| Id  | Operation                            | Name          | E-Rows |
-----------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |               |        |
|   1 |  NESTED LOOPS OUTER                  |               |     37 |
|   2 |   NESTED LOOPS                       |               |      9 |
|   3 |    NESTED LOOPS                      |               |      9 |
|   4 |     VIEW                             |               |      9 |
|*  5 |      COUNT STOPKEY                   |               |        |
|*  6 |       TABLE ACCESS FULL              | ORDERS        |     10 |
|   7 |     TABLE ACCESS BY INDEX ROWID      | ORDERS        |      1 |
|*  8 |      INDEX UNIQUE SCAN               | ORDER_PK      |      1 |
|   9 |    TABLE ACCESS BY INDEX ROWID       | CUSTOMERS     |      1 |
|* 10 |     INDEX UNIQUE SCAN                | CUSTOMERS_PK  |      1 |
|  11 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDER_ITEMS   |      4 |
|* 12 |    INDEX RANGE SCAN                  | ITEM_ORDER_IX |      4 |
-----------------------------------------------------------------------

Predicate Information:
   5 - filter(ROWNUM<10)
   6 - filter("ORDER_STATUS"<=4)

Note: this is an adaptive plan
```

**Análise:** O `TABLE ACCESS FULL` em ORDERS (operação 6) é causado pelo filtro `WHERE order_status <= 4 AND ROWNUM < 10`. O optimizer não consegue usar índice em `order_status` quando combinado com ROWNUM — precisa escanear para encontrar qualquer 10 linhas elegíveis. Resultado: **1.040 consistent gets por execução** para retornar 9 pedidos.

---

### Plano — `gujuujxvwzv48` (Lab plantado — ANTES do índice)

```
Plan hash value: 156782776

---------------------------------------------------
| Id  | Operation          | Name        | E-Rows |
---------------------------------------------------
|   0 | SELECT STATEMENT   |             |        |
|   1 |  SORT AGGREGATE    |             |      1 |
|*  2 |   TABLE ACCESS FULL| LAB4_ORDERS |      5 |
---------------------------------------------------

   2 - filter("CUSTOMER_ID"=:B1)
```

**16.093 consistent gets** para retornar ~5 linhas — varre 500.000 linhas a cada execução.

---

## Fix aplicado — criação do índice

```sql
CREATE INDEX c##claude.idx_lab4_orders_custid
ON c##claude.lab4_orders(customer_id);
```

Após o índice, o cursor `gujuujxvwzv48` foi **automaticamente invalidado** (`invalidations=1`) e re-parseado na próxima execução.

### Plano — `gujuujxvwzv48` (DEPOIS do índice)

```
Plan hash value: 59964129

-------------------------------------------------------------
| Id  | Operation         | Name                   | E-Rows |
-------------------------------------------------------------
|   0 | SELECT STATEMENT  |                        |        |
|   1 |  SORT AGGREGATE   |                        |      1 |
|*  2 |   INDEX RANGE SCAN| IDX_LAB4_ORDERS_CUSTID |      5 |
-------------------------------------------------------------

   2 - access("CUSTOMER_ID"=:B1)
```

**3 consistent gets** — redução de 99,98%.

---

## Comparativo before/after

| Métrica | Sem índice | Com índice | Melhoria |
|---|---:|---:|---:|
| Operação | TABLE ACCESS FULL | INDEX RANGE SCAN | — |
| Consistent gets/exec | **16.093** | **3** | **99,98%** |
| Linhas examinadas/exec | ~500.000 | ~5 | 100.000x |
| Disk reads/exec | 0 | 0 | — |

---

## Findings e Recomendações

### Finding 1 — Full Scan em LAB4_ORDERS (impacto CRÍTICO · esforço BAIXO · risco BAIXO)

16.093 gets/exec em 310 execuções. Fix: criar índice em `customer_id`. Redução de 99,98%.

### Finding 2 — Full Scan em ORDERS (impacto MÉDIO · esforço MÉDIO · risco BAIXO)

SQL `34mt4skacwwwd` do fluxo New Order faz TABLE ACCESS FULL em ORDERS com filtro `order_status <= 4 AND ROWNUM < 10`. O ROWNUM impede uso de índice em `order_status`. Alternativa: criar índice em `(order_status, order_id)` e reescrever a query para evitar ROWNUM.

### Finding 3 — Disk reads = 0 ≠ ausência de problema

Ambos os full scans correram inteiramente em buffer cache. O problema se manifesta em CPU e latches, não em I/O. Em produção com buffer cache menor, adicionaria I/O físico.

### Finding 4 — Limitação tkprof em Multitenant

Jobs de `C##` executam no CDB$ROOT. `DBMS_MONITOR` via PDB não alcança essas sessões para gerar trace file. Workaround: ativar trace via conexão ao CDB (`/ AS SYSDBA`).

---

## Lição central

`V$SQL` é o **primeiro lugar** a olhar — sem parar nada, sem trace, sem AWR. Ordenar por `elapsed_time` revela o consumo total; dividir por `executions` revela o custo unitário. `DBMS_XPLAN.DISPLAY_CURSOR` confirma qual operação causa o consumo — plano real, não hipotético.

---

## Evidências

| Artefato | Localização |
|---|---|
| Script de plant | `modulo-4-sql-diagnosis/plant.sql` |
| Script de cleanup | `modulo-4-sql-diagnosis/cleanup.sql` |
| SQL_ID Swingbench ofensor | `34mt4skacwwwd` — TABLE ACCESS FULL em ORDERS |
| SQL_ID lab (antes/depois) | `gujuujxvwzv48` — 16.093 → 3 gets/exec |
| Relatório HTML | `HTML/Desafio4/sql_diagnosis_report.html` |
