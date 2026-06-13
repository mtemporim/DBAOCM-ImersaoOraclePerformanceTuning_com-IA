# Desafio 3 — AWR, ASH & ADDM

**Módulo:** 3 — O que aconteceu nesta janela de tempo?  
**Data:** 2026-06-13  
**Banco:** Oracle 23ai (23.26.1.0.0) — PDB `ORCLPDB` / host `ol9-orcl26-ribas`  
**Snapshots AWR:** 2717 → 2718 (15:20:39 → 15:29:00, ~8,4 min de janela)  
**Snap AWR efetivo (carga):** 2717 → 2718 (15:27:20 → 15:29:00, ~1,7 min de carga real)  
**Workload:** Swingbench SOE + plant.sql (lock contention + buffer busy waits)

---

## Objetivo

Responder "o que aconteceu nesta janela?" usando os repositórios históricos do Oracle:

| Relatório | Fonte | Granularidade |
|---|---|---|
| **AWR** | `DBA_HIST_*` (snapshots) | Visão completa do intervalo: time model, top SQL, top waits, I/O |
| **ASH** | `V$ACTIVE_SESSION_HISTORY` | Sessão a sessão, segundo a segundo — ideal para picos curtos |
| **ADDM** | Advisor sobre `DBA_HIST_*` | Findings ranqueados por impacto com recomendações prontas |

---

## O que o plant.sql plantou

```sql
-- 1 job: segura lock em LAB3_HOT.id=1 por 4 minutos
DBMS_SCHEDULER.create_job('LAB3_LOCK_HOLDER', ...
    UPDATE c##claude.lab3_hot SET contador=contador+1 WHERE id=1;
    DBMS_SESSION.SLEEP(240);  -- trava por 4 minutos sem commitar
    COMMIT;

-- 4 jobs: tentam atualizar a mesma linha (id=1) → enq: TX
DBMS_SCHEDULER.create_job('LAB3_LOCK_WAITER_1..4', ...
    UPDATE c##claude.lab3_hot SET contador=contador+1 WHERE id=1; COMMIT;

-- 5 jobs: updates aleatórias na tabela hot → buffer busy waits
DBMS_SCHEDULER.create_job('LAB3_HOT_BURN_1..5', ...
    UPDATE c##claude.lab3_hot SET contador=DBMS_RANDOM.VALUE(2,101); COMMIT;
```

**Efeito esperado no AWR/ASH:**
- `enq: TX - row lock contention` nos top wait events
- `buffer busy waits` nos top segments
- Blocking session identificável via ASH

---

## Metodologia

### Preparação dos snapshots

```sql
-- Snapshot antes do plant
v_snap := DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();  -- snap 2717

-- [plant.sql executado]
-- [60 segundos aguardados]

-- Snapshot após
v_snap := DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();  -- snap 2718
```

> **Nota DBID:** Conectado via PDB, `V$DATABASE.DBID` retorna o DBID do PDB (1753532951). Os snapshots AWR estão sob o DBID do CDB (4231836195). É necessário usar o DBID do CDB nas funções `AWR_REPORT_TEXT` e `ASH_REPORT_TEXT`.

### Geração dos relatórios

```sql
-- AWR — spoolado para DocsImersao/reports/
SELECT output FROM TABLE(DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_TEXT(
    l_dbid => 4231836195, l_inst_num => 1, l_bid => 2717, l_eid => 2718));

-- ASH — query direta em V$ACTIVE_SESSION_HISTORY
SELECT * FROM TABLE(DBMS_WORKLOAD_REPOSITORY.ASH_REPORT_TEXT(
    l_dbid => 4231836195, l_inst_num => 1,
    l_btime => begin_interval_time_snap2717,
    l_etime => end_interval_time_snap2718));
```

Ambos spoolados para `DocsImersao/reports/`. Leitura do AWR delegada ao `subagent-reader`.

---

## Resultados — AWR (snap 2717→2718)

**DB Time:** 12,29 minutos em 1,67 min de clock → **~7,4 sessões simultâneas ativas**

### Instance Efficiency

| Métrica | Valor | Status |
|---|---:|---|
| Buffer hit | 99,9% | ✅ Excelente |
| Soft parse | 94,8% | ⚠️ Abaixo do ideal (> 99%) |
| Execute-to-parse | 99,6% | ✅ Ótimo |
| Latch hit | ~99,9% | ✅ Excelente |

### Top 5 Timed Events

| Evento | Waits | Total (s) | % DB Time |
|---|---:|---:|---:|
| **DB CPU** | — | 331,0 | **44,9%** |
| **enq: TX - row lock contention** | 4.496 | 318,7 | **43,2%** |
| library cache: mutex X | 300.450 | 43,3 | 5,9% |
| row cache mutex | 85.049 | 18,4 | 2,5% |
| **buffer busy waits** | 150.751 | 14,7 | **2,0%** |

### Top SQLs por Elapsed

| SQL_ID | Elapsed (s) | Execuções | % DB Time | Texto |
|---|---:|---:|---:|---|
| `8jr1ksnt95jdp` | 418,2 | — | 56,7% | DBMS_SCHEDULER wrapper (lock holder) |
| `4dtquj1jxty5s` | 335,6 | 1 | 45,5% | `UPDATE LAB3_HOT ... WHERE ID = 1` |
| `dx5hpz3487wb9` | 180,8 | 463.630 | 24,5% | `UPDATE LAB3_HOT ... WHERE ID = TRUNC(...)` |
| `638cgtr1kuxh6` | 335,6 | — | 45,5% | DBMS_SCHEDULER PL/SQL wrapper |

### Segmentos hot

| Objeto | Row Lock Waits | Buffer Busy Waits | Logical Reads |
|---|---:|---:|---:|
| `C##CLAUDE.LAB3_HOT` (TABLE) | **100%** | **100%** | 17,3% |
| `SYS_C008748` (INDEX de LAB3_HOT) | — | — | 16,5% |

---

## Resultados — ASH (15:20:39 → 15:29:00)

**Amostras:** 1.683 · **Avg Active Sessions:** 3,36 · **Avg Active/CPU:** 0,21

### Top User Events

| Evento | % Atividade | Avg Active Sessions |
|---|---:|---:|
| CPU + Wait for CPU | 64,41% | 2,16 |
| **enq: TX - row lock contention** | **19,31%** | **0,65** |
| log file sync | 10,70% | 0,36 |
| library cache: mutex X | 2,08% | 0,07 |
| row cache mutex | 1,43% | 0,05 |

### Top Sessions em enq: TX

| SID | Serial# | % Atividade | Programa |
|---|---|---:|---|
| 14 | 550 | 4,75% | oracle (J001) |
| 253 | 29916 | 4,75% | oracle (J002) |
| 376 | 49551 | 4,75% | oracle (J003) |
| 501 | 31105 | 4,75% | oracle (J004) |

> Quatro jobs `LAB3_LOCK_WAITER_*` aguardando exatamente como plantado.

### Blocking Session

| Bloqueador | % Atividade causada | Evento gerado |
|---|---:|---|
| SID 1349 (LAB3_LOCK_HOLDER) | **19,01%** | enq: TX - row lock contention |

### Top Objeto

| Objeto | % Atividade | Evento |
|---|---:|---|
| `C##CLAUDE.LAB3_HOT` (TABLE) | **20,14%** | enq: TX - row lock contention |

### Timeline de atividade

| Slot | Evento dominante | % |
|---|---|---:|
| 15:20–15:22 | CPU + log file sync (só Swingbench) | ~30% |
| 15:23–15:26 | Baixíssima atividade (3 amostras/min) | — |
| **15:27** | **enq: TX aparece (plant ativo)** | **5,05%** |
| **15:28** | **enq: TX domina** | **14,26%** |

---

## Findings e Recomendações

### Finding 1 — Row lock contention em hot row (impacto CRÍTICO — 43,2% DB Time)

**Evidência AWR:** `enq: TX - row lock contention` = 43,2% DB Time (318,7s em 1,7 min de janela).  
**Evidência ASH:** SID 1349 (`LAB3_LOCK_HOLDER`) bloqueando 4 sessões por 19,01% do tempo total.  
**Objeto:** `C##CLAUDE.LAB3_HOT` — 100% dos row lock waits concentrados nesta tabela.

**Causa raiz:** uma transação atualiza `id=1` e dorme 240s sem commitar. Quatro outras transações tentam atualizar o mesmo `id=1` → enfileiramento em `enq: TX`.

**Em produção:** padrão típico de aplicações que fazem `UPDATE counter WHERE id = 1` como "sequence caseiro" — o pior anti-pattern de concorrência Oracle.

**Ação recomendada:**
- Substituir pelo uso de `SEQUENCE` nativo Oracle (NEXTVAL é lock-free)
- Ou usar `SELECT ... FOR UPDATE SKIP LOCKED` se filas forem o padrão
- Em emergência: identificar e matar o bloqueador via `ALTER SYSTEM KILL SESSION '1349,28066'`

### Finding 2 — Buffer busy waits em hot block (impacto ALTO — 2,0% DB Time)

**Evidência AWR:** 150.751 waits / 14,7s / avg 97µs — 100% concentrado em `LAB3_HOT`.  
**Causa raiz:** 5 sessões fazem `UPDATE` em linhas aleatórias da mesma tabela pequena (100 linhas = poucos blocos) → múltiplas sessões disputam os mesmos blocos de cache.

**Em produção:** tabelas com poucos blocos e alta concorrência de escrita — candidatas à particionamento, hash clustering ou design revisado.

### Finding 3 — library cache: mutex X + row cache mutex (impacto ALTO — 8,4% DB Time)

**Evidência AWR:** mutex X = 5,9% / 300k waits (avg 144µs); row cache = 2,5% / 85k waits.  
**Causa:** parse storm derivado do alto volume de sessões (Swingbench + 10 jobs do plant) tentando parsear cursores simultaneamente.

**Ação:** verificar `session_cached_cursors` (aumentar de 50 para 200), `cursor_sharing = FORCE` se aplicável.

### Finding 4 — log file sync persistente (impacto MÉDIO — 10,7% no ASH)

**Evidência ASH:** 10,70% de atividade em `log file sync` — sinal consistente com o Finding #1 do Desafio 2 (redo logs subdimensionados).  
**Correlação:** os 5 jobs `LAB3_HOT_BURN_*` fazem COMMIT a cada UPDATE → alta frequência de commits → pressão no LGWR.

---

## Diferença entre AWR e ASH neste lab

| Métrica | AWR (snap 2717→2718) | ASH (15:20→15:29) |
|---|---|---|
| Janela real | 1,7 min (só snap 2718) | 8,4 min (todo o período) |
| enq: TX | **43,2% DB time** | **19,31% atividade** |
| CPU | 44,9% | 64,41% |
| Por que difere? | Snap 2717 capturou estado limpo; snap 2718 = só a janela do plant | ASH dilui com os ~7 min anteriores sem plant |

> **Lição:** Para picos curtos, o ASH captura melhor o sinal (você vê a evolução minuto a minuto). O AWR captura a janela entre snapshots — se o snap cobre exatamente o pico, o sinal é amplificado (como aqui).

---

## Resultados — ADDM (snap 2717→2718)

O ADDM foi executado via `DBMS_ADDM.ANALYZE_DB` + `DBMS_ADDM.GET_REPORT`. A criação falhou inicialmente por um auto-binding do SQLcl (string literals em parâmetros `IN OUT VARCHAR2` são substituídos por `TO_CHAR(BIND_VAR)`, que não pode ser destino de designação). Solução: declarar variável PL/SQL intermediária.

### Metodologia ADDM

```sql
DECLARE
    v_task VARCHAR2(30) := 'lab3_addm';
    v_bid  NUMBER       := 2717;
    v_eid  NUMBER       := 2718;
BEGIN
    BEGIN DBMS_ADDM.DELETE(v_task); EXCEPTION WHEN OTHERS THEN NULL; END;
    DBMS_ADDM.ANALYZE_DB(v_task, v_bid, v_eid);
END;
/
-- Relatório spoolado para DocsImersao/reports/addm_lab3.txt
```

> **Nota:** `DBMS_ADDM` usa o DBID do CDB (4231836195) via `dba_hist_snapshot` — mesma restrição do AWR.

### Findings ADDM (ranqueados por impacto)

| # | Finding | % Atividade | Avg Active Sessions | Recomendação |
|---|---|---:|---:|---|
| **1** | Top-SQL Statements | **89,06%** | **6,57** | SQL Tuning Advisor + investigar bloqueio |
| **2** | Row Lock Waits em LAB3_HOT | **43,23%** | **3,19** | Application Analysis (eliminar anti-pattern) |
| **3** | Shared Pool Latches | **8,46%** | 0,62 | Nenhuma (informational) |

#### Finding 1 — Top-SQL (89,06% atividade)

| SQL_ID | Impacto ADDM | Execuções | Elapsed | Waits | Recomendação |
|---|---:|---:|---:|---|---|
| `4dtquj1jxty5s` | **43,75%** | 1 | 335s | 100% enq:TX | Investigar bloqueio — hot row anti-pattern |
| `dx5hpz3487wb9` | **23,44%** | 463.630 | avg 0,39ms | buffer busy | SQL Tuning Advisor |
| `8jr1ksnt95jdp` | **14,06%** | — | — | — | DBMS_SCHEDULER wrapper (não aplicável) |

#### Finding 2 — Row Lock Waits (43,23% atividade)

- **Objeto:** `C##CLAUDE.LAB3_HOT` (object# 1695456)
- **Bloqueador:** SID 1349, SERIAL# 28066 — responsável por **93% do benefício possível**
- **SQL bloqueado:** `4dtquj1jxty5s` (`UPDATE LAB3_HOT ... WHERE ID = 1`)
- **Recomendação ADDM:** Application Analysis — redesenhar a lógica de counter

#### Finding 3 — Shared Pool Latches (8,46% atividade)

- `library cache: mutex X` = 5% DB time
- `row cache mutex` = 2% DB time
- Sem recomendações disponíveis (informational only)

---

## Correlação AWR + ASH + ADDM

| Finding | AWR | ASH | ADDM |
|---|---|---|---|
| Row lock (enq: TX) | 43,2% DB Time | 19,31% atividade | 43,23% atividade (Finding 2) |
| Bloqueador | `8jr1ksnt95jdp` (PL/SQL wrapper) | SID 1349 (19,01%) | SID 1349 SERIAL# 28066 (93% benefício) |
| Hot SQL | `4dtquj1jxty5s` 45,5% | — | 43,75% impacto (Finding 1) |
| Buffer busy | 2,0% DB Time | — | dentro do Top-SQL (Finding 1) |
| Library cache mutex | 5,9% DB Time | 2,08% | 5% DB Time (Finding 3) |

> Os três relatórios convergem para o mesmo diagnóstico: o anti-pattern de "sequence caseiro" em `LAB3_HOT.id=1` é a causa raiz de 43%+ do DB Time.

---

## Padrão de delegação aplicado

```
Agente principal
  │
  ├── AWR: SPOOL → DocsImersao/reports/awr_2717_2718.txt (2.471 linhas)
  │         └── subagent-reader lê e devolve JSON (~2KB)
  │
  ├── ASH: resultado direto do tool (output da query < limite)
  │         └── agente principal usa diretamente
  │
  └── ADDM: SPOOL → DocsImersao/reports/addm_lab3.txt
            └── conteúdo devolvido inline pelo MCP (< 300 linhas)
                → agente principal usa diretamente
```

---

## Próximos Passos

| Pergunta | Próximo Módulo |
|---|---|
| Qual SQL específico dentro do Swingbench consome mais CPU? | Módulo 4 — SQL Diagnosis |
| O library cache mutex é sintoma de shared pool subdimensionado? | Módulo 5 — Shared Pool |
| O buffer busy wait some com particionamento? | Módulo 7 — Buffer Cache & I/O |

---

## Evidências

| Artefato | Localização |
|---|---|
| AWR report | `DocsImersao/reports/awr_2717_2718.txt` |
| ASH report | `DocsImersao/reports/ash_2717_2718.txt` |
| ADDM report | `DocsImersao/reports/addm_lab3.txt` |
| Script de plant | `modulo-3-awr-ash-addm/plant.sql` |
| Relatório HTML | `HTML/Desafio3/awr_ash_report.html` |
