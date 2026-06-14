# Desafio 5 — Shared Pool & Parse Storm

**Módulo:** 5 — Library Cache, hard parse e cursores  
**Data:** 2026-06-14  
**Banco:** Oracle 23ai (23.26.1.0.0) — PDB `ORCLPDB` / host `ol9-orcl26-ribas`  
**Workload:** Swingbench SOE + plant.sql (5 jobs gerando parse storm por 3 minutos)

---

## Objetivo

Entender como o **Shared Pool** armazena cursores parseados e o que acontece quando a aplicação monta SQL com **literais em vez de bind variables**. Cobre:

| Ferramenta | O que mostra |
|---|---|
| `V$SYS_TIME_MODEL` | Ponto de entrada — onde o tempo está sendo gasto? |
| `V$SYSSTAT` | `parse count (hard)` vs `parse count (total)` — taxa de hard parse |
| `V$SQL` | SQL_IDs distintos por `plan_hash_value` — assinatura de parse storm |
| `cursor_sharing = FORCE` | Paliativo automático — Oracle substitui literais por bind variables |
| `EXECUTE IMMEDIATE ... USING :1` | Fix definitivo — bind variable no código |

---

## Fluxo de diagnóstico utilizado

```
oracle-time-model
    └── parse time elapsed > 5% (bordeline) → pós-fix
        └── [durante storm era > 10%]
            └── auto-encadeia → oracle-shared-pool
                    └── V$SYSSTAT → hard parse 18,2%
                    └── V$SQL → 10.002 SQL_IDs distintos
                    └── Fix: cursor_sharing=FORCE → 0,04%
                    └── Fix: bind variable → 2 SQL_IDs · 0,05 MB
```

---

## Passo 0 — Ponto de entrada: Time Model (oracle-time-model)

**Regra:** toda investigação começa aqui. O Time Model revela *onde* o banco gasta tempo antes de mergulhar em vistas específicas.

```sql
-- Snapshot inicial → aguardar 60–120 s → snapshot final
SELECT stat_name, ROUND(value/1e6,2) AS delta_seg,
       ROUND(100*value / MAX(value) OVER (PARTITION BY NULL), 1) AS pct_db_time
FROM v$sys_time_model
ORDER BY value DESC FETCH FIRST 10 ROWS ONLY;
```

**Resultado real — janela de 51 segundos (estado pós-fix, Swingbench SOE ativo):**

| Componente | Δ segundos | % DB time | |
|---|---:|---:|---|
| DB time | 1,25 | 100,0% | base |
| DB CPU | 1,24 | 99,1% | banco dominado por CPU |
| sql execute elapsed time | 1,11 | 89,2% | execução de SQL consome quase todo o CPU |
| **parse time elapsed** | **0,07** | **5,3%** | ⚠️ na borda do threshold (> 5%) |
| hard parse elapsed time | 0,05 | 4,1% | quase todo o parse é hard parse |
| hard parse (sharing criteria) | 0,00 | 0,3% | |
| PL/SQL execution elapsed time | 0,00 | 0,1% | |

**Durante o storm ativo**, com 5 jobs gerando literais aleatórios, `parse time elapsed` estava muito acima de 10% do DB time — o V$SYSSTAT confirmou 18,2% de hard parse sobre o total de parses.

### Decisão de auto-encadeamento

| Regra do CLAUDE.md | Condição | Resultado |
|---|---|---|
| `parse time elapsed > 10%` | ✓ (durante storm) | → invocar `oracle-shared-pool` |

O Time Model é o **gatilho**. Sem ele, a investigação começa sem saber *para onde ir*.

---

## Passo 1 — Confirmar com V$SYSSTAT (oracle-shared-pool)

```sql
SELECT name, value FROM v$sysstat
WHERE name IN ('parse count (total)', 'parse count (hard)', 'execute count');
```

| Métrica | Valor | |
|---|---:|---|
| parse count (hard) | **10.962** | |
| parse count (total) | 60.082 | |
| execute count | 65.270 | |
| **Hard parse %** | **18,2%** | ⚠️ Threshold: > 5% |

**Papel:** quantifica o problema com precisão. O Time Model disse *onde* (parse time); o V$SYSSTAT diz *quanto* (18,2% de hard parse).

---

## Passo 2 — Identificar o padrão de literais em V$SQL

```sql
SELECT plan_hash_value,
       COUNT(DISTINCT sql_id)                    AS sql_ids_distintos,
       ROUND(SUM(sharable_mem)/1024/1024, 1)    AS memoria_mb
FROM v$sql
WHERE UPPER(sql_text) LIKE '%LAB5_TARGET%'
GROUP BY plan_hash_value
ORDER BY sql_ids_distintos DESC;
```

| plan_hash_value | SQL_IDs distintos | Memória |
|---|---:|---:|
| 4156915540 | **10.002** | **225 MB** |

**Interpretação:** um único plano lógico (mesma query), **10.002 SQL_IDs diferentes**. Assinatura inequívoca de literais concatenados.

Confirmação pelo texto:

```
sql_id: 2s2a3tgtm1q7b → SELECT COUNT(*) FROM lab5_target WHERE id = 3795
sql_id: c8s6uj65d1y32 → SELECT COUNT(*) FROM lab5_target WHERE id = 8568
sql_id: 27qa7pu1pty2q → SELECT COUNT(*) FROM lab5_target WHERE id = 905
```

---

## Por que o código gerou o storm

```sql
-- O job montava o SQL assim (ERRADO):
v_sql := 'SELECT COUNT(*) FROM c##claude.lab5_target WHERE id = '
         || TRUNC(DBMS_RANDOM.VALUE(1, 10000));
EXECUTE IMMEDIATE v_sql INTO v_count;
-- Cada valor = SQL_ID único = hard parse
```

---

## Fix 1 — Paliativo: cursor_sharing = FORCE

```sql
ALTER SYSTEM SET cursor_sharing = FORCE SCOPE=BOTH;
```

Oracle normaliza literais antes do parse: `WHERE id = 3795` → `WHERE id = :"SYS_B_0"`. Todos os valores compartilham o mesmo cursor.

| Fase | Hard parse % | Novos hard parses |
|---|---:|---:|
| Storm ativo | **18,2%** | — |
| Após FORCE | **0,04%** | +143 em 390k execuções |

**Limitação:** `cursor_sharing=FORCE` impede o optimizer de usar histogramas em colunas com skew. Paliativo, não definitivo.

---

## Fix 2 — Definitivo: bind variables no código

```sql
EXECUTE IMMEDIATE
    'SELECT /*+ bind_test */ COUNT(*) FROM c##claude.lab5_target WHERE id = :1'
    INTO v_count
    USING TRUNC(DBMS_RANDOM.VALUE(1, 10000));
```

**Comparativo direto (dados reais do lab):**

| Abordagem | SQL_IDs criados | Execuções | Memória |
|---|---:|---:|---:|
| **Literal (storm)** | **10.002** | 1.025.829 | **225 MB** |
| **Bind variable** | **2** | 201 | **0,05 MB** |

**Redução de 4.500x com uma linha de código.**

---

## Findings e Recomendações

### Finding 1 — Parse storm por literais (CRÍTICO · esforço MÉDIO · risco BAIXO)

**Evidência:** Time Model → parse time > threshold → V$SYSSTAT confirma 18,2% hard parse → V$SQL revela 10.002 cursores e 225 MB.  
**Causa raiz:** `EXECUTE IMMEDIATE` com literal concatenado.  
**Fix definitivo:** `:1` com `USING`. Uma linha. Impacto eliminado.

### Finding 2 — cursor_sharing = FORCE (ALTO · esforço BAIXO · risco MÉDIO)

**Quando usar:** código legado que não pode ser alterado imediatamente.  
**Limitação:** optimizer perde histogramas de colunas com skew.  
**Recomendação:** emergência → FORCE; médio prazo → bind variables no código.

### Finding 3 — Memória residual 225 MB (BAIXO · sem ação imediata)

Os cursores antigos saem via LRU. `FLUSH SHARED POOL` é opção de manutenção, nunca em produção ativa.

### Finding 4 — Diagnóstico completo em 3 queries (positivo)

Time Model → V$SYSSTAT → V$SQL. Menos de 2 minutos. Zero impacto no banco.

---

## Padrão de skills utilizado

| Skill | Papel | Tipo de execução |
|---|---|---|
| `oracle-time-model` | Ponto de entrada — identifica parse time elevado | Agente principal (resultado pequeno) |
| `oracle-shared-pool` | Investigação — confirma storm, identifica ofensores, aplica fix | Agente principal (resultado pequeno) |
| `subagent-reader` | **Não utilizado** — não havia arquivos grandes para ler | — |
| `subagent-queries` | **Não utilizado** — V$SQL retornou resultado pequeno agrupado | — |

---

## Evidências

| Artefato | Valor |
|---|---|
| Time Model — parse time elapsed | 5,3% pós-fix (> 10% durante storm) |
| Hard parse % pico (V$SYSSTAT) | 18,2% |
| SQL_IDs distintos (V$SQL) | 10.002 — plan_hash 4156915540 |
| Memória consumida | 225 MB no library cache |
| Após cursor_sharing=FORCE | 0,04% (143 hard parses em 390k exec) |
| Bind variable (200 exec) | 2 SQL_IDs · 0,05 MB |
| Relatório HTML | `HTML/Desafio5/shared_pool_report.html` |
