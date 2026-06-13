# DBAOCM — Imersão Oracle Performance Tuning com IA

Imersão Oracle da DBAOCM de Performance Tuning com IA — 2 dias de diagnóstico e tuning no Oracle 23ai.

## Ambiente

- **Banco:** Oracle 23ai (23.26.1.0.0) — PDB `ORCLPDB`
- **Host:** ol9-orcl26-ribas (Oracle Linux 9)
- **Workload base:** Swingbench SOE (Order Entry)

## Desafios

| Desafio | Tema | Ferramenta principal |
|---|---|---|
| [Desafio 1](Desafio1/relatorio-desafio1.md) | Time Model — onde o banco gasta o tempo? | `V$SYS_TIME_MODEL` |
| [Desafio 2](Desafio2/relatorio-desafio2.md) | Alert Log & Trace Files (Evento 10046) | `DBMS_MONITOR` · alert log |
| [Desafio 3](Desafio3/relatorio-desafio3.md) | AWR, ASH & ADDM | `DBMS_WORKLOAD_REPOSITORY` · `DBMS_ADDM` |
