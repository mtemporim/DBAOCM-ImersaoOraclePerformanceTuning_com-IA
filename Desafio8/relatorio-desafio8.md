# Desafio 8 — PGA, Temp Tablespace & Buffer Cache sob Carga

**Data:** 2026-06-14
**Banco:** Oracle 23ai (23.26.1.0.0) — PDB ORCLPDB
**Host:** ol9-orcl26-ribas
**Workload:** Swingbench SOE + 3 jobs LAB8_SPILL (GROUP BY + ORDER BY em 4M linhas, payload 200 chars)

---

## Cadeia de Skills

```text
oracle-time-model → [sql execute 99,9% | DB CPU 98,7%] → oracle-pga-space
```

DB CPU > 50% indicaria `oracle-sql-diagnosis` pelo auto-chain padrão.
Os jobs LAB8_SPILL com GROUP BY + ORDER BY massivo direcionaram a investigação
para PGA e temp tablespace antes de prosseguir.

---

## Passo 0 — Time Model (entrada obrigatória)

**Janela:** 60 segundos com 3 jobs LAB8_SPILL ativos (GROUP BY em 4M linhas)

| Componente | Δ segundos | % DB time |
| --- | ---: | ---: |
| **DB time** | **155,4s** | 100,0% |
| **DB CPU** | **153,3s** | **98,7%** |
| sql execute elapsed | 155,3s | 99,9% |
| I/O waits (DB time − DB CPU) | 2,1s | 1,4% |
| parse time | ~0s | ~0% |

**Interpretação:** sql execute = 99,9% com quase zero I/O waits. Os jobs fazem
agregação massiva intensamente em CPU. A investigação de PGA determinará se operam
em modo optimal ou geram spill — o Time Model por si não distingue os dois casos.

---

## Passo 1 — PGA: Parâmetros e Uso Atual

**Fonte:** `V$PARAMETER`, `V$PGASTAT`

| Parâmetro | Nosso ambiente |
| --- | ---: |
| pga_aggregate_target | **12.830 MB (~12,5 GB)** |
| pga_aggregate_limit | 25.660 MB (2× target) |
| workarea_size_policy | AUTO |
| global memory bound | **1.024 MB por workarea** |
| memory_target / sga_target | 0 (gerenciamento manual) |

| Métrica (V$PGASTAT) | Valor |
| --- | ---: |
| total PGA inuse | 184 MB |
| total PGA allocated | 260 MB |
| maximum PGA allocated | 346 MB |
| total freeable PGA memory | 68 MB |

---

## Passo 2 — Workareas: dois cenários

O mesmo workload produz resultados radicalmente diferentes conforme o `pga_aggregate_target` disponível.

### Cenário A — PGA farta (nosso ambiente: ~12,5 GB)

**Fonte:** `V$SQL_WORKAREA_HISTOGRAM` — acumulado desde startup

| Modo | Execuções | % |
| --- | ---: | ---: |
| **optimal** | **30.987** | **100%** |
| 1-pass | 0 | 0% |
| multi-pass | 0 | 0% |

Workareas ativas no pico: 3× GROUP BY (HASH) usando ~54.841 KB cada (~53 MB/job),
passes=0, temp spill=nenhum. Com 1 GB de bound por workarea, as agregações em
4M linhas cabem inteiramente em PGA.

### Cenário B — PGA restrita (~949 MB / global bound baixo)

| Modo | Antes da carga | Com nova carga | Δ novos |
| --- | ---: | ---: | ---: |
| optimal | 16.293 | 21.056 | +4.763 |
| 1-pass | 0 | 0 | 0 |
| **multi-pass** | **64** | **64** | **0 novos** |

**PGA hit % = 43,0%** no target atual. Curva de advice com inflexão em **1.139 MB**.

**Wait events que surgem do zero no cenário B:**

| Evento | Total (s) | Avg ms | Status |
| --- | ---: | ---: | --- |
| direct path write temp | 1.722,8s | 75,2 ms | **NOVO** |
| direct path read temp | 195,5s | 12,3 ms | **NOVO** |
| direct path read | 101,0s | 18,9 ms | **NOVO** |
| db file sequential read | 11.950,2s | 8,5 ms | piorou 3× |

`direct path write/read temp` surgindo do zero = **assinatura diagnóstica de spill de PGA**.

---

## Passo 3 — Buffer Cache: impacto da LAB8_DATA

O efeito no buffer cache depende do tamanho relativo da tabela em relação ao cache.

| Métrica | Cenário A (cache 32 GB) | Cenário B (cache 1,1 GB) |
| --- | --- | --- |
| Tamanho do buffer cache | 32.320 MB | 1.120 MB |
| LAB8_DATA no cache | ~163 MB (~0,5%) | **839,9 MB (75%)** |
| Hit ratio | 99,96% | 99,55% (–0,27 pp) |
| SOE.INVENTORIES | 162,9 MB (estável) | 82,4 MB (–50%) |
| ADDRESSES / CARD_DETAILS / CUSTOMERS | presentes | **expulsos do cache** |
| Physical reads USERS | estável | **triplicaram (+202%)** |

No cenário B, LAB8_DATA entra no pool padrão e expulsa todos os objetos hot do SOE:
efeito de **thrashing**. O SOE começa a reler do disco o que antes estava em cache,
triplicando `db file sequential read`.

**Causa raiz encadeada no cenário B:**

```text
LAB8_DATA (839 MB) monopoliza buffer cache (75%)
  → expulsa objetos hot do SOE
  → SOE relê do disco → db file sequential read ×3
  → PGA insuficiente para sorts → spill → direct path write temp (75 ms avg)
  → latência I/O acumulada em toda a cadeia
```

---

## Passo 4 — Temp Tablespace

**Fonte:** `DBA_TEMP_FREE_SPACE`, `V$TEMPSEG_USAGE`

| | Cenário A (nosso) | Cenário B (restrito) |
| --- | --- | --- |
| Temp total | 20 MB | maior |
| **Temp usado** | **2 MB** | **spill ativo** |
| direct path write temp | ausente | 1.722,8s acumulados |

---

## Passo 5 — PGA Target Advice (cenário B)

**Fonte:** `V$PGA_TARGET_ADVICE`

| PGA target (MB) | Hit % | Extra I/O (MB) |
| ---: | ---: | ---: |
| **949 (atual)** | **43,0%** | **45.180** |
| 1.139 | 75,0% | 11.295 ← inflexão |
| 1.329 | 75,0% | 11.295 |
| 1.898 | 75,0% | 11.295 |

Aumentar de 949 MB para **1.139 MB resolve o cenário B** com impacto mínimo:
reduz o Extra I/O estimado de 45.180 MB para 11.295 MB (–75%).

---

## Findings & Recomendações

| # | Finding | Cenário | Impacto | Esforço | Risco |
| --- | --- | --- | --- | --- | --- |
| F1 | 100% optimal — zero spill | A (nosso) | Positivo | — | — |
| F2 | PGA sobre-dimensionada — advice plateau desde 1,6 GB | A (nosso) | Observar | Baixo | Baixo |
| F3 | direct path write/read temp surgindo do zero | B (restrito) | **Crítico** | Baixo | Baixo |
| F4 | Thrashing: LAB8_DATA expulsa SOE do cache (75%) | B (restrito) | **Alto** | Médio | Baixo |
| F5 | PGA hit 43% — fix em 1.139 MB | B (restrito) | **Alto** | Baixo | Baixo |

### Fix F3/F5 — Aumentar PGA target (cenário B)

```sql
ALTER SYSTEM SET pga_aggregate_target = 1200M SCOPE=BOTH;
```

### Fix F4 — Proteger objetos hot do SOE (cenário B)

```sql
-- Big Table Caching evita que LAB8_DATA compita com índices SOE
ALTER SYSTEM SET db_big_table_cache_percent_target = 20 SCOPE=BOTH;

-- Ou fixar objetos críticos no KEEP pool
ALTER TABLE soe.inventories  STORAGE (BUFFER_POOL KEEP);
ALTER TABLE soe.order_items  STORAGE (BUFFER_POOL KEEP);
```

### Queries de monitoramento para produção

```sql
-- 1) Workareas fora do optimal?
SELECT SUM(optimal_executions), SUM(onepass_executions),
       SUM(multipasses_executions),
       ROUND(SUM(optimal_executions)*100.0/NULLIF(SUM(total_executions),0),1) AS pct_optimal
FROM v$sql_workarea_histogram WHERE total_executions > 0;
-- Threshold: pct_optimal < 95% → investigar

-- 2) Quem está spillando para temp agora?
SELECT s.sid, s.username, s.program, ROUND(tu.blocks*8/1024,1) AS mb_temp
FROM v$tempseg_usage tu JOIN v$session s ON s.saddr = tu.session_addr
ORDER BY tu.blocks DESC;

-- 3) PGA advice calibrado?
SELECT ROUND(pga_target_for_estimate/1024/1024) AS pga_mb,
       estd_pga_cache_hit_percentage AS hit_pct, estd_overalloc_count AS overalloc
FROM v$pga_target_advice ORDER BY pga_target_for_estimate;
```

---

## Skills Utilizadas

| Skill | Utilizada | Motivo |
| --- | --- | --- |
| oracle-time-model | Sim | Entrada obrigatória — snapshot delta 60s |
| oracle-pga-space | Sim | Workload GROUP BY massivo — investigação de PGA, workareas e temp |
| oracle-buffer-io | Sim (complementar) | Impacto da LAB8_DATA no buffer cache avaliado |
| oracle-sql-diagnosis | Não | SQL dos jobs conhecido pelo contexto do lab |
| subagent-space | Não | Análise de temp não exigiu delegação |
