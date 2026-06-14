# Desafio 7 — Buffer Cache & I/O

**Data:** 2026-06-14  
**Banco:** Oracle 23ai (23.26.1.0.0) — PDB ORCLPDB  
**Host:** ol9-orcl26-ribas  
**Workload:** Swingbench SOE (sem plant.sql adicional)

---

## Cadeia de Skills

```
oracle-time-model  →  [DB CPU 93,1% | I/O waits 6,8%]  →  oracle-buffer-io
```

O componente dominante (DB CPU > 50%) indicaria `oracle-sql-diagnosis` pelo auto-chain padrão.
A presença de 6,8% de DB time em waits de I/O (38,4s absolutos) justificou investigar o perfil
de buffer cache e I/O antes de encerrar o módulo — e descartar pressão de storage como contribuinte.

---

## Passo 0 — Time Model (entrada obrigatória)

**Janela:** 60 segundos  
**Snapshots:** v$sys_time_model antes/depois de DBMS_SESSION.SLEEP(60)

| Componente | Δ segundos | % DB time |
|---|---:|---:|
| **DB time** | **562,6s** | 100,0% |
| **DB CPU** | **524,2s** | **93,1%** |
| sql execute elapsed | 361,4s | 64,2% |
| I/O waits (DB time − DB CPU) | 38,4s | 6,8% |
| parse time | 0,07s | ~0% |

**Interpretação:** banco CPU-bound com workload intenso (~9,4 cores médios em 60s).
I/O waits de 6,8% são baixos em termos absolutos mas presentes — investigação de buffer/IO indicada.

---

## Passo 1 — Buffer Cache

**Fonte:** `V$SGAINFO`, `V$PARAMETER`

| Métrica | Valor |
|---|---|
| Tamanho real (V$SGAINFO) | **32.320 MB (~31,6 GB)** |
| Hit Ratio | **99,96%** |
| sga_target | 0 (gerenciamento manual) |
| db_cache_size | 0 (Oracle auto-dimensionou) |
| db_big_table_cache_percent_target | **0 — Big Table Caching desabilitado** |
| db_keep_cache_size | 0 |

Hit ratio de 99,96% indica que praticamente todo I/O é servido do cache.
O risco não está na taxa de acerto, mas em saber o que ocupa o cache e se full scans
podem expulsar objetos quentes (thrashing).

---

## Passo 2 — Top Objetos no Buffer Cache

**Fonte:** `V$BH` + `DBA_OBJECTS`

| Owner | Objeto | Tipo | Blocos | MB cache | % do cache |
|---|---|---:|---:|---:|---:|
| SOE | INVENTORIES | TABLE | 20.849 | 162,9 | **17,1%** |
| SOE | ORDER_ITEMS | TABLE | 17.502 | 136,7 | 14,4% |
| SOE | ORDERS | TABLE | 13.751 | 107,4 | 11,3% |
| SOE | CUSTOMERS | TABLE | 9.999 | 78,1 | 8,2% |
| SOE | ORDER_ITEMS_PK | INDEX | 7.458 | 58,3 | 6,1% |
| SOE | ADDRESSES | TABLE | 7.177 | 56,1 | 5,9% |
| SOE | ITEM_ORDER_IX | INDEX | 6.788 | 53,5 | 5,6% |
| SOE | ORD_WAREHOUSE_IX | INDEX | 3.714 | 29,3 | 2,4% |
| SOE | CUST_EMAIL_IX | INDEX | 3.254 | 25,4 | 2,1% |
| SOE | ITEM_PRODUCT_IX | INDEX | 3.151 | 24,6 | 2,0% |

**INVENTORIES** ocupa 17,1% do cache (162,9 MB) isoladamente.
Sem Big Table Caching, full scans nesta tabela competem com índices quentes pelo mesmo pool.

---

## Passo 3 — I/O por Tablespace

**Fonte:** `V$FILESTAT` + `V$DATAFILE` + `V$TABLESPACE`

| Tablespace | Phys Reads | Phys Writes | Read ms avg | Write ms avg |
|---|---:|---:|---:|---:|
| **USERS** | **52.479** | **67.515** | **0,1 ms** | 0,7 ms |
| SYSTEM | 8.171 | 2.878 | ~0 ms | 0,6 ms |
| SYSAUX | 7.758 | 13.029 | ~0 ms | 0,6 ms |
| UNDOTBS1 | 121 | 11.262 | ~0 ms | 0,9 ms |

USERS concentra todo o workload SOE e tem o maior volume de I/O físico.
Latência de 0,1 ms confirma **storage SSD**. Referência: SSD saudável < 1 ms, HDD saudável 5–10 ms.

---

## Passo 4 — Top Wait Events de I/O

**Fonte:** `V$SYSTEM_EVENT`

| Evento | Classe | Waits | Total (s) | Latência avg |
|---|---|---:|---:|---:|
| **db file sequential read** | User I/O | 51.219 | 3,4s | **0,1 ms** |
| db file scattered read | User I/O | 15.218 | 2,1s | 0,1 ms |
| **read by other session** | User I/O | 192 | 0,5s | **2,4 ms** |

`db file sequential read` domina em volume mas com latência muito baixa (SSD).
`read by other session` com 2,4 ms médio indica **hot blocks**: sessões aguardando
outra trazer o mesmo bloco — padrão esperado em workload OLTP concorrente como SOE.

---

## Findings & Recomendações

| # | Finding | Evidência | Impacto | Esforço | Risco |
|---|---|---|---|---|---|
| F1 | Big Table Caching desabilitado | `db_big_table_cache_percent_target=0` | Médio | Baixo | Baixo |
| F2 | INVENTORIES ocupa 17,1% do cache | 162,9 MB sem proteção de pool | Médio | — | — |
| F3 | read by other session: 2,4 ms | Hot blocks no workload SOE | Baixo | Médio | Baixo |
| F4 | Buffer cache bem dimensionado | 99,96% hit ratio, 32 GB | Positivo | — | — |
| F5 | Storage SSD: 0,1 ms | Abaixo do limiar crítico | Positivo | — | — |

### Fix F1 — Habilitar Big Table Caching

```sql
ALTER SYSTEM SET db_big_table_cache_percent_target = 20 SCOPE=BOTH;
```

Reserva 20% do buffer cache (~6,4 GB) para tabelas grandes.
Full scans em INVENTORIES deixam de competir com índices quentes no pool padrão.

### Fix F2 — Proteger objetos críticos com KEEP pool (alternativo)

```sql
ALTER TABLE soe.inventories STORAGE (BUFFER_POOL KEEP);
```

Fixa INVENTORIES no KEEP pool, imune a aging mesmo sob pressão de full scan.
Requer `db_keep_cache_size` configurado.

---

## Skills Utilizadas

| Skill | Utilizada | Motivo |
|---|---|---|
| oracle-time-model | Sim | Entrada obrigatória — snapshot delta 60s |
| oracle-buffer-io | Sim | Componente de I/O presente; perfil de buffer/storage investigado |
| oracle-sql-diagnosis | Não | SQL ofensor não identificado neste módulo |
| oracle-awr-ash-addm | Não | Waits de I/O não justificaram drill-down em ASH |
| subagent-reader | Não | Sem arquivos grandes para leitura |
| subagent-queries | Não | V$BH retornou em volume aceitável |
