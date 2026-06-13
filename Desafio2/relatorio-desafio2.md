# Desafio 2 — Alert Log & Trace Files (Evento 10046)

**Módulo:** 2 — Leitura de evidências Oracle  
**Data:** 2026-06-13  
**Banco:** Oracle 23ai (23.26.1.0.0) — PDB `ORCLPDB` / host `ol9-orcl26-ribas`  
**Workload:** Swingbench SOE (ativo durante toda a coleta)  
**Arquivos analisados:** `alert_orcl.log` · `orcl_ora_2667099.trc`  
**Janela do alert log:** 14:00 → 15:13 (2026-06-13)  
**Janela do trace:** 15:11:12 → 15:13:41 (~149 segundos)

---

## Objetivo

Extrair sinal diagnóstico de dois tipos de arquivo que o Oracle gera continuamente:

| Arquivo | Conteúdo | Tamanho |
|---|---|---|
| `alert_<SID>.log` | ORA- errors, log switches, checkpoints, mudanças estruturais | Centenas de MB |
| `<sid>_ora_<pid>.trc` | SQL Trace 10046: cada SQL, waits por evento, bind values, planos | Dezenas de MB |

Ambos são **grandes demais para leitura direta** — a leitura foi delegada ao `subagent-reader`, que devolveu resumos estruturados em JSON.

---

## Metodologia

### 1. Ativação do SQL Trace (plant.sql)

O plant ativou o evento **10046** na sessão SOE mais ativa via `DBMS_MONITOR.SESSION_TRACE_ENABLE`:

```sql
DBMS_MONITOR.SESSION_TRACE_ENABLE(
    session_id   => 14,     -- SID da sessão SOE mais ativa
    serial_num   => 22734,
    waits        => TRUE,   -- captura todos os wait events
    binds        => TRUE,   -- captura valores dos bind variables
    plan_stat    => 'ALL_EXECUTIONS'  -- plano em cada execução
);
```

O trace foi gerado em:
```
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_2667099.trc
```

### 2. Cópia dos arquivos para reports/

Os arquivos foram copiados da VM para `DocsImersao/reports/` via SCP:

```powershell
scp oracle@10.11.0.201:/u01/app/oracle/diag/rdbms/orcl/orcl/trace/alert_orcl.log     DocsImersao\reports\
scp oracle@10.11.0.201:/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_2667099.trc DocsImersao\reports\
```

### 3. Delegação ao subagent-reader

Dois agentes especializados foram despachados **em paralelo** — um para cada arquivo. O `subagent-reader` lê os arquivos em contexto isolado e devolve apenas JSON estruturado, nunca conteúdo bruto.

---

## Resultados — Alert Log

**Janela analisada:** 2026-06-13T14:00:01 → 15:13:18 (73 minutos)

### ORA- Codes

| Código | Ocorrências na janela | Total no log | Origem |
|---|---:|---:|---|
| **ORA-08181** | **833** | 20.713 | Workers ORCLPDB (w000–w00h) — SCN inválido |

> ORA-08181 repetido em massa indica que processos background tentam validar SCNs que não existem — padrão associado a atividade de Change Data Capture, LogMiner ou Flashback com séries de SCN interrompidas.

### Eventos Críticos

| Evento | Ocorrências | Significado |
|---|---:|---|
| **Log switch** | **10** em 73 min | Redo logs muito pequenos — troca a cada ~7,3 min |
| Thread 1 cannot allocate new log | 10 | LGWR não consegue abrir novo grupo antes de precisar |
| ATTENTION: redo log writes slow | 1 (14:08:40) | I/O do LGWR acima do threshold interno do Oracle |
| Checkpoint not complete | 0 | — sem ocorrências (positivo) |

### Crescimento de datafiles

| Arquivo | Início | Fim | Crescimento |
|---|---|---|---|
| undotbs01.dbf | 102.400 KB | ~389.120 KB | +~280 MB em 73 min |
| users01.dbf | 73.728 KB | ~753.152 KB | +~665 MB em 73 min |

---

## Resultados — Trace File 10046

**Sessão:** SOE / SID=14 / SERIAL#=22734 / JDBC Thin Client (ip: 172.23.5.10)  
**Duração:** 149 segundos (15:11:12 → 15:13:41)  
**Total elapsed:** 44,11s · **Total CPU:** 31,12s

### Top 5 SQLs por elapsed total

| # | SQL_ID | Execuções | Elapsed total (s) | CPU (s) | Wait (s) | Wait dominante |
|---|---|---:|---:|---:|---:|---|
| 1 | `3gnwwgbk8h6au` | 16.230 | 12,12 | 10,15 | 11,84 | SQL\*Net msg from client |
| 2 | `bvcnjjrdz9aab` | 20.710 | 11,24 | 6,73 | 30,83 | SQL\*Net msg from client |
| 3 | `09pzy8x10gjkg` | 2.480 | 3,86 | 2,40 | 1,87 | SQL\*Net msg from client |
| 4 | `34mt4skacwwwd` | 159 | 3,05 | 2,57 | 0,11 | SQL\*Net msg from client |
| 5 | `a6hdpzrqqhc7d` | 1.277 | 2,42 | 1,45 | 0,96 | SQL\*Net msg from client |

> SQLs 1 e 2 são variantes de busca em `PRODUCTS`/`INVENTORIES` com bind variables — elapsed unitário < 1ms, muito eficientes. SQL 4 é CTE sobre `ORDERS`, ~19ms/execução — candidato futuro a tuning.

### Waits agregados

| Evento | Total (s) | Ocorrências | Avg (ms) |
|---|---:|---:|---:|
| SQL\*Net message from client | 62,68 | 83.615 | 0,75 |
| **log file sync** | **7,54** | **4.258** | **1,77** |
| SQL\*Net message to client | 1,50 | 83.615 | 0,02 |
| cursor: pin S | 0,09 | 60 | 1,5 |
| buffer busy waits | 0,01 | 74 | 0,13 |
| log file switch (pvt strand) | 0,01 | 1 | — |

> `SQL*Net message from client` domina mas é wait **ocioso** (round-trip da aplicação) — não é gargalo no banco. `log file sync` com 1,77ms/commit é aceitável hoje, mas alinhado ao sinal do alert log (redo logs pequenos, LGWR pressionado).

---

## Findings e Recomendações

### Finding 1 — Redo logs subdimensionados (impacto ALTO, risco MÉDIO, esforço BAIXO)

**Evidência:** 10 log switches em 73 minutos = a cada ~7,3 min. Oracle recomenda ≥ 20 min por switch para reduzir checkpoint pressure.  
**Sintoma correlato:** ATTENTION de redo log writes lentos às 14:08 e 10 eventos de "Private strand flush not complete".

**Ação recomendada:**
```sql
-- Verificar tamanho atual dos redo log groups
SELECT l.group#, l.members, l.bytes/1024/1024 AS mb, l.status
FROM v$log l ORDER BY l.group#;

-- Adicionar novos grupos maiores (ex: 500MB cada) e dropar os antigos
ALTER DATABASE ADD LOGFILE GROUP 21 SIZE 500M;
ALTER DATABASE ADD LOGFILE GROUP 22 SIZE 500M;
-- Após switches: ALTER DATABASE DROP LOGFILE GROUP <grupo_antigo>;
```

### Finding 2 — ORA-08181 em massa (impacto MÉDIO, risco BAIXO, esforço MÉDIO)

**Evidência:** 833 ocorrências em 73 min (20.713 no log total). Gerado por workers ORCLPDB ao validar SCNs.  
**Causa provável:** atividade de Change Data Capture, LogMiner ou Flashback com lacunas de SCN.

**Ação recomendada:** investigar se há jobs de CDC/replicação ativos. Verificar `DBA_CAPTURE`, `DBA_APPLY`.

### Finding 3 — log file sync moderado (impacto BAIXO, risco BAIXO, esforço BAIXO)

**Evidência:** 7,54s / 4.258 commits = 1,77ms/commit na sessão tracada.  
**Contexto:** aceitável para VM de lab. Em produção, threshold de alerta é > 5ms/commit.

**Ação:** monitorar. Se crescer após redimensionar redo logs, investigar I/O do storage para os redo log files.

### Finding 4 — Workload saudável no trace (positivo)

- Sem `db file sequential read` ou `db file scattered read` — 100% em buffer cache
- Parse time negligível — bind variables sendo usadas corretamente
- Sem ORA- errors no trace

---

## Padrão de delegação aplicado

```
Agente principal                     Subagent-reader (×2 em paralelo)
      │                                       │
      │── prompt: "leia alert_orcl.log" ─────►│── Read/Grep no arquivo grande
      │── prompt: "leia 2667099.trc"   ─────►│── Grep por PARSING/FETCH/EXEC/WAIT
      │                                       │
      │◄── JSON estruturado (≤ 2KB) ─────────│
      │◄── JSON estruturado (≤ 2KB) ─────────│
      │
      └── sintetiza e gera relatório
```

> Princípio pedagógico: o agente principal **nunca lê diretamente** arquivos grandes. A delegação ao `subagent-reader` protege o contexto e escala para arquivos de centenas de MB.

---

## Próximos Passos

| Pergunta | Próximo Módulo |
|---|---|
| O que aconteceu neste período segundo snapshots AWR? | Módulo 3 — AWR / ASH / ADDM |
| Qual SQL está consumindo mais CPU (além do Swingbench)? | Módulo 4 — SQL Diagnosis |
| O redo log sync está afetando o Shared Pool? | Módulo 5 — Shared Pool |

**Recomendação:** executar o `cleanup.sql` agora para desativar o trace 10046 e evitar que o arquivo cresça indefinidamente.

```sql
-- cleanup.sql — Módulo 2
DECLARE
    v_sid NUMBER; v_serial NUMBER;
BEGIN
    FOR r IN (SELECT sid, serial# FROM c##claude.lab2_trace_session) LOOP
        BEGIN DBMS_MONITOR.SESSION_TRACE_DISABLE(r.sid, r.serial#);
        EXCEPTION WHEN OTHERS THEN NULL; END;
    END LOOP;
END;
/
DROP TABLE c##claude.lab2_trace_session PURGE;
```

---

## Evidências

| Artefato | Localização |
|---|---|
| Script de plant | `modulo-2-alert-trace/plant.sql` |
| Script de cleanup | `modulo-2-alert-trace/cleanup.sql` |
| Alert log (cópia) | `DocsImersao/reports/alert_orcl.log` |
| Trace file (cópia) | `DocsImersao/reports/orcl_ora_2667099.trc` |
| Relatório HTML | `HTML/Desafio2/alert_trace_report.html` |
