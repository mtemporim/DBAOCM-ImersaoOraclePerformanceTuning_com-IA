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
| `V$SYSSTAT` | `parse count (hard)` vs `parse count (total)` — taxa de hard parse |
| `V$SQL` | SQL_IDs distintos por `plan_hash_value` — assinatura de parse storm |
| `cursor_sharing = FORCE` | Paliativo automático — Oracle substitui literais por bind variables |
| `EXECUTE IMMEDIATE ... USING :1` | Fix definitivo — bind variable no código |

---

## O que o plant.sql plantou

```sql
-- Tabela alvo com 10.000 linhas
CREATE TABLE c##claude.lab5_target (id NUMBER, valor VARCHAR2(50));
INSERT ... CONNECT BY level <= 10000;

-- 5 jobs executando SQL com LITERAL aleatório por 3 minutos
-- Cada execução = cursor único = hard parse
v_sql := 'SELECT COUNT(*) FROM c##claude.lab5_target WHERE id = '
         || TRUNC(DBMS_RANDOM.VALUE(1, 10000));
EXECUTE IMMEDIATE v_sql INTO v_count;
```

**Efeito:** cada valor aleatório gera um SQL_ID diferente. Em 3 minutos, ~10.000 cursores únicos no library cache.

---

## Diagnóstico — Passo 1: Taxa de hard parse

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

---

## Diagnóstico — Passo 2: Assinatura de literais em V$SQL

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

**Interpretação:** um único plano (mesma query lógica), **10.002 SQL_IDs diferentes** — cada literal gerou um cursor único.

Confirmação pelo texto:

```
sql_id: 2s2a3tgtm1q7b → SELECT COUNT(*) FROM lab5_target WHERE id = 3795
sql_id: c8s6uj65d1y32 → SELECT COUNT(*) FROM lab5_target WHERE id = 8568
sql_id: 27qa7pu1pty2q → SELECT COUNT(*) FROM lab5_target WHERE id = 905
...
```

Mesma query, literal diferente, cursor diferente, parse diferente.

---

## Fix 1 — Paliativo: cursor_sharing = FORCE

```sql
ALTER SYSTEM SET cursor_sharing = FORCE SCOPE=BOTH;
```

O Oracle passa a normalizar literais automaticamente antes do parse — `WHERE id = 3795` vira `WHERE id = :"SYS_B_0"`. Todos os valores compartilham o mesmo cursor.

**Resultado após FORCE:**

| Fase | Hard parse % | Novos hard parses |
|---|---:|---:|
| Storm ativo | **18,2%** | — |
| Após FORCE | **0,04%** | +143 em 390k execuções |

Hard parse essencialmente eliminado.

**Cuidado:** `cursor_sharing=FORCE` é um paliativo. Overhead de substituição de texto a cada parse + pode interferir com histogramas em colunas com distribuição não uniforme (optimizer perde visibilidade de skew).

---

## Fix 2 — Definitivo: bind variables no código

Simulado com 200 execuções via `EXECUTE IMMEDIATE ... USING :1`:

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

200 execuções com bind criaram **2 cursores e 0,05 MB**.  
1 milhão com literal criaram **10.002 cursores e 225 MB**.  
Mesma query lógica, resultado idêntico — impacto **4.500x maior** com literais.

---

## Findings e Recomendações

### Finding 1 — Parse storm por literais (impacto CRÍTICO · esforço MÉDIO · risco BAIXO)

**Evidência:** 10.002 SQL_IDs distintos para o mesmo plano · 225 MB de library cache consumidos · hard parse = 18,2%.  
**Causa raiz:** `EXECUTE IMMEDIATE` montando SQL com literal concatenado.  
**Fix definitivo:** usar bind variables — `:1` com `USING v_id`. Uma linha de código, impacto eliminado.

### Finding 2 — cursor_sharing = FORCE como paliativo (impacto ALTO · esforço BAIXO · risco MÉDIO)

**Quando usar:** código legado que não pode ser alterado rapidamente.  
**Limitação:** Oracle substitui literais por `:"SYS_B_N"` — optimizer não consegue usar histogramas de colunas com skew (perde a informação de distribuição).  
**Recomendação:** aplicar como medida emergencial + planejar refactor do código para bind variables.

### Finding 3 — Memória residual (225 MB) (impacto BAIXO · sem ação imediata)

Os 10.002 cursores antigos permanecem no library cache e saem via LRU naturalmente.  
Em caso de urgência: `ALTER SYSTEM FLUSH SHARED POOL` libera imediatamente, mas causa pico de hard parse para todos os SQLs legítimos (uso em manutenção, nunca em produção ativa).

### Finding 4 — Diagnóstico em 3 queries (positivo)

`V$SYSSTAT` → hard parse %. `V$SQL` agrupado por `plan_hash_value` → identifica o padrão. Texto do SQL → confirma os literais. Sem AWR, sem trace, sem parar nada.

---

## Padrão de delegação aplicado

Todas as queries deste módulo retornam resultados pequenos — agente principal executou direto via MCP SQLcl. Sem delegação a subagents.

---

## Próximos Passos

| Pergunta | Próximo Módulo |
|---|---|
| O optimizer está tomando decisões certas com as estatísticas? | Módulo 6 — CBO & Optimizer |
| Hit ratio do buffer cache está adequado? | Módulo 7 — Buffer Cache & I/O |

---

## Evidências

| Artefato | Localização |
|---|---|
| Script de plant | `modulo-5-shared-pool/plant.sql` |
| Script de cleanup | `modulo-5-shared-pool/cleanup.sql` |
| SQL_IDs distintos (storm) | 10.002 — plan_hash 4156915540 |
| Memória consumida | 225 MB no library cache |
| Hard parse % pico | 18,2% |
| Após cursor_sharing=FORCE | 0,04% (143 hard parses em 390k exec) |
| Bind variable (200 exec) | 2 SQL_IDs · 0,05 MB |
| Relatório HTML | `HTML/Desafio5/shared_pool_report.html` |
