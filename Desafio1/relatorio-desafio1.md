# Desafio 1 — DB Time e Time Model

**Módulo:** 1 — Onde está o tempo?  
**Data:** 2026-06-13  
**Banco:** Oracle 23ai (23.26.1.0.0) — PDB `ORCLPDB` / host `ol9-orcl26-ribas`  
**Workload:** Swingbench SOE + plant.sql (3 jobs CPU-burn via DBMS_SCHEDULER)  
**Janela de observação:** 105 segundos  

---

## Objetivo

Responder à pergunta mais fundamental de performance Oracle: **onde está sendo gasto o DB time?**

O Time Model decompõe o `DB time` em componentes hierárquicos, permitindo identificar rapidamente se o banco está esperando (waits) ou trabalhando (CPU). Essa análise é o ponto de entrada obrigatório de qualquer investigação de performance.

---

## Metodologia

### 1. Preparação de carga (plant.sql)

Antes de medir, injetamos carga extra controlada via `plant.sql`. O script cria 3 jobs no `DBMS_SCHEDULER` sob o usuário `C##CLAUDE`, cada um executando um loop CPU-bound puro por ~3 minutos:

```sql
DBMS_SCHEDULER.create_job(
    job_name   => 'C##CLAUDE.LAB1_CPU_BURN_' || i,
    job_type   => 'PLSQL_BLOCK',
    job_action => q'[
        DECLARE
            v_end TIMESTAMP := SYSTIMESTAMP + INTERVAL ''3'' MINUTE;
            v_sum NUMBER := 0;
        BEGIN
            WHILE SYSTIMESTAMP < v_end LOOP
                FOR k IN 1 .. 100000 LOOP
                    v_sum := v_sum + SQRT(k) * LN(k + 1);
                END LOOP;
            END LOOP;
        END;
    ]',
    enabled    => TRUE,
    auto_drop  => TRUE
);
```

**Propósito pedagógico:** forçar DB CPU a dominar o Time Model, tornando o padrão CPU-bound visível e inequívoco na medição delta.

### 2. Snapshot inicial

Capturamos os valores acumulados de `V$SYS_TIME_MODEL` em uma GTT (Global Temporary Table) `c##claude.tm_snapshot`, criada com `ON COMMIT PRESERVE ROWS` para persistir os dados durante a sessão:

```sql
CREATE GLOBAL TEMPORARY TABLE IF NOT EXISTS c##claude.tm_snapshot (
    stat_name VARCHAR2(64),
    value     NUMBER,
    captured  TIMESTAMP DEFAULT SYSTIMESTAMP
) ON COMMIT PRESERVE ROWS;

MERGE INTO c##claude.tm_snapshot t
USING (SELECT stat_name, value FROM v$sys_time_model) s
ON (t.stat_name = s.stat_name)
WHEN NOT MATCHED THEN INSERT (stat_name, value) VALUES (s.stat_name, s.value);
```

**Snapshot capturado às:** 14:52:47

### 3. Janela de observação

Aguardamos 90 segundos (via `DBMS_SESSION.SLEEP(90)`) com o Swingbench e os 3 jobs CPU-burn rodando simultaneamente.

### 4. Snapshot final e cálculo do delta

Calculamos a diferença entre o estado atual de `V$SYS_TIME_MODEL` e o snapshot inicial, convertendo de microssegundos para segundos e calculando o percentual em relação ao `DB time` total:

```sql
WITH atual   AS (SELECT stat_name, value AS valor_atual FROM v$sys_time_model),
     inicial AS (SELECT stat_name, value AS valor_inicial, captured FROM c##claude.tm_snapshot),
     db_time AS (
         SELECT NULLIF(MAX(a.valor_atual - i.valor_inicial), 0) AS db_time_delta
         FROM atual a JOIN inicial i ON i.stat_name = a.stat_name
         WHERE a.stat_name = 'DB time'
     )
SELECT
    a.stat_name,
    ROUND((a.valor_atual - i.valor_inicial) / 1e6, 2)                   AS delta_segundos,
    ROUND(100 * (a.valor_atual - i.valor_inicial) / d.db_time_delta, 1) AS pct_db_time,
    ROUND((SYSDATE - CAST(i.captured AS DATE)) * 86400, 0)              AS janela_segundos
FROM atual a
JOIN inicial i ON i.stat_name = a.stat_name
CROSS JOIN db_time d
WHERE (a.valor_atual - i.valor_inicial) > 0
ORDER BY delta_segundos DESC
FETCH FIRST 15 ROWS ONLY;
```

---

## Resultados

| Componente | Δ segundos | % DB Time | Janela (s) |
|---|---:|---:|---:|
| **DB time** | **317,36** | **100,0%** | 105 |
| sql execute elapsed time | 317,14 | 99,9% | 105 |
| **PL/SQL execution elapsed time** | **314,96** | **99,2%** | 105 |
| **DB CPU** | **313,98** | **98,9%** | 105 |
| parse time elapsed | 0,04 | ~0% | 105 |
| hard parse elapsed time | 0,01 | ~0% | 105 |
| hard parse (sharing criteria) elapsed time | 0,01 | ~0% | 105 |
| PL/SQL compilation elapsed time | 0,00 | ~0% | 105 |

> **Janela real:** 105 segundos (90s de sleep + overhead de conexão e queries).

---

## Análise e Findings

### Finding 1 — Workload 100% CPU-bound (impacto: ALTO)

**DB CPU representa 98,9% do DB time.** Isso significa que o banco Oracle está consumindo praticamente todo o seu tempo útil em processamento de CPU — sem esperas significativas por I/O, locks ou latches.

Com 313,98 segundos de CPU em uma janela de 105 segundos de relógio, há em média **~3 sessões simultâneas consumindo CPU 100%** durante o período (313,98 / 105 ≈ 3,0 — os 3 jobs do plant.sql mais o Swingbench).

### Finding 2 — PL/SQL domina a execução (impacto: MÉDIO)

**PL/SQL execution elapsed time = 99,2% do DB time.** Esse valor elevado é esperado e explicado pelos 3 jobs `LAB1_CPU_BURN_*` que executam loops PL/SQL contínuos. Em um cenário de produção, PL/SQL execution próximo de 100% indicaria packages ou procedures monopolizando sessões — candidatos imediatos a revisão de algoritmo.

### Finding 3 — Zero pressão de parse (impacto: BAIXO)

**Parse time = 0,04s (~0%)** e **hard parse = 0,01s (~0%).** O cursor sharing está funcionando perfeitamente. Não há parse storm, sem pressão no Shared Pool. Em produção isso é sinal positivo: a aplicação usa bind variables e os cursores estão sendo reutilizados.

### Finding 4 — Waits não aparecem

Com DB CPU em 98,9%, sobram apenas 1,1% de DB time para waits (eventos de espera). Não há I/O wait, latch contention, lock wait ou wait de rede significativo. O banco não está bloqueado em nada — está trabalhando.

---

## Contraste: baseline vs com plant

| Cenário | DB CPU | PL/SQL | sql execute | Parse |
|---|---:|---:|---:|---:|
| Swingbench SOE (sem plant) | 87,0% | ~0% | 57,2% | ~0% |
| Swingbench + plant (3 jobs) | 98,9% | 99,2% | 99,9% | ~0% |

O plant amplificou o sinal de CPU de 87% para 98,9%, tornando o padrão inequívoco. Sem o plant, o SOE sozinho já era CPU-bound — o banco pequeno (scale 0.1, ~50MB) cabe no buffer cache, eliminando I/O físico.

---

## Próximos Passos

Dado que `DB CPU` domina com 98,9%, o próximo passo natural da investigação é:

| Pergunta | Próximo Módulo |
|---|---|
| **Qual SQL consome mais CPU?** | Módulo 4 — SQL Diagnosis |
| **O CPU é gasto em leituras lógicas excessivas?** | Módulo 7 — Buffer Cache & I/O |
| **Há parse implícito sendo ignorado?** | Módulo 5 — Shared Pool |

**Recomendação:** prosseguir pelo **Módulo 4 (SQL Diagnosis)** para identificar os top SQLs por CPU e validar se são do workload SOE esperado ou anomalias.

---

## Evidências

| Artefato | Localização |
|---|---|
| Script de plant | `modulo-1-time-model/plant.sql` |
| Script de delta | `.claude/skills/oracle-time-model/queries/time_model_delta.sql` |
| GTT de snapshot | `C##CLAUDE.TM_SNAPSHOT` (sessão Oracle) |
| Relatório HTML | `HTML/Desafio1/time_model_report.html` |
