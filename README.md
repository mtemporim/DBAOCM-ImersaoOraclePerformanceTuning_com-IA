# DBAOCM — Imersão Oracle Performance Tuning com IA

Imersão Oracle da DBAOCM de Performance Tuning com IA — 2 dias de diagnóstico e tuning no Oracle 23ai.

## Ambiente

- **Banco:** Oracle 23ai (23.26.1.0.0) — PDB `ORCLPDB`
- **Host:** ol9-orcl26-ribas (Oracle Linux 9)
- **Workload base:** Swingbench SOE (Order Entry)

## Desafios

| Desafio | Tema | Ferramenta principal |
| --- | --- | --- |
| [Desafio 1](Desafio1/relatorio-desafio1.md) | Time Model — onde o banco gasta o tempo? | `V$SYS_TIME_MODEL` |
| [Desafio 2](Desafio2/relatorio-desafio2.md) | Alert Log & Trace Files (Evento 10046) | `DBMS_MONITOR` · alert log |
| [Desafio 3](Desafio3/relatorio-desafio3.md) | AWR, ASH & ADDM | `DBMS_WORKLOAD_REPOSITORY` · `DBMS_ADDM` |
| [Desafio 4](Desafio4/relatorio-desafio4.md) | SQL Diagnosis & Execution Plans | `V$SQL` · `DBMS_XPLAN` |
| [Desafio 5](Desafio5/relatorio-desafio5.md) | Shared Pool & Library Cache — Parse Storm | `V$SYSSTAT` · `V$SQL` · `cursor_sharing` |
| [Desafio 6](Desafio6/relatorio-desafio6.md) | CBO — Cost-Based Optimizer & Estatísticas | `DBA_TAB_STATISTICS` · `DBMS_STATS` · `DBMS_XPLAN` |
| [Desafio 7](Desafio7/relatorio-desafio7.md) | Buffer Cache & I/O | `V$BH` · `V$FILESTAT` · `V$SYSTEM_EVENT` · `V$SGAINFO` |
| [Desafio 8](Desafio8/relatorio-desafio8.md) | PGA, Temp Tablespace & Segments | `V$SQL_WORKAREA_HISTOGRAM` · `V$PGA_TARGET_ADVICE` · `DBA_TEMP_FREE_SPACE` |
| [Desafio 9](Desafio9/relatorio-desafio9.md) | Cost Reduction: Chaining, HWM & Compressão | `DBA_TABLES.chain_cnt` · `DBA_SEGMENTS` · `ROW STORE COMPRESS ADVANCED` |
