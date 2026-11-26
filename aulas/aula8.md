# Aula 8: SQL Tuning (`EXPLAIN PLAN` e Índices), Funções Analíticas (Window Functions) e Cirurgia de Erros (`BACKTRACE` + `SAVEPOINT`)

## 1. Preparação do Ambiente e Gerador de Massa (10.000 Registros)
Para testar planos de execução e índices em um cenário realista, utilizamos a técnica nativa de multiplicação de linhas com `DUAL` e `CONNECT BY LEVEL <= 10000`, inserindo 10.000 ordens de produção em menos de 1 segundo sem loops PL/SQL.

```sql
DROP TABLE historico_producao CASCADE CONSTRAINTS;
DROP TABLE auditoria_precos CASCADE CONSTRAINTS;
DROP TABLE produtos CASCADE CONSTRAINTS;
DROP TABLE linhas_moveis CASCADE CONSTRAINTS;

CREATE TABLE linhas_moveis (
    id NUMBER PRIMARY KEY,
    nome VARCHAR2(50) NOT NULL
);

CREATE TABLE produtos (
    id NUMBER PRIMARY KEY,
    id_linha NUMBER REFERENCES linhas_moveis(id),
    descricao VARCHAR2(100) NOT NULL,
    preco_custo NUMBER(10,2) NOT NULL,
    preco_venda NUMBER(10,2) NOT NULL,
    estoque NUMBER DEFAULT 0
);

CREATE TABLE auditoria_precos (
    id_log NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_produto NUMBER REFERENCES produtos(id),
    preco_registrado NUMBER(10,2),
    data_reajuste DATE
);

CREATE TABLE historico_producao (
    id_op NUMBER PRIMARY KEY,
    codigo_lote VARCHAR2(30) NOT NULL,
    id_produto NUMBER REFERENCES produtos(id),
    qtd_produzida NUMBER NOT NULL,
    data_fabricacao DATE NOT NULL
);

INSERT INTO linhas_moveis VALUES (1, 'Cozinha e Jantar');
INSERT INTO linhas_moveis VALUES (2, 'Dormitórios');
INSERT INTO linhas_moveis VALUES (3, 'Corporativo e Office');

INSERT INTO produtos VALUES (101, 1, 'Mesa de Jantar 6 Lugares Carvalho', 850.00, 1690.00, 12);
INSERT INTO produtos VALUES (102, 2, 'Roupeiro Casal 6 Portas MDF 18mm', 1400.00, 2890.00, 8);
INSERT INTO produtos VALUES (103, 1, 'Cristaleira Rústica Vidro Temperado', 620.00, 1250.00, 5);
INSERT INTO produtos VALUES (104, 3, 'Mesa Diretor em L com Gaveteiro', 580.00, 1150.00, 20);
INSERT INTO produtos VALUES (105, 2, 'Cômoda 5 Gavetas Corrediça Telescópica', 390.00, 790.00, 15);
INSERT INTO produtos VALUES (106, 3, 'Estação de Trabalho Dupla Freijó', 980.00, 1990.00, 6);
INSERT INTO produtos VALUES (107, 1, 'Balcão Buffet Ripado 180cm', 720.00, 1580.00, 9);

INSERT INTO auditoria_precos (id_produto, preco_registrado, data_reajuste) VALUES (101, 1450.00, DATE '2026-01-10');
INSERT INTO auditoria_precos (id_produto, preco_registrado, data_reajuste) VALUES (101, 1590.00, DATE '2026-05-15');
INSERT INTO auditoria_precos (id_produto, preco_registrado, data_reajuste) VALUES (101, 1690.00, DATE '2026-10-01');
INSERT INTO auditoria_precos (id_produto, preco_registrado, data_reajuste) VALUES (102, 2500.00, DATE '2026-02-01');
INSERT INTO auditoria_precos (id_produto, preco_registrado, data_reajuste) VALUES (102, 2890.00, DATE '2026-09-20');

-- Gerador de 10.000 Ordens de Produção em SQL puro
INSERT INTO historico_producao (id_op, codigo_lote, id_produto, qtd_produzida, data_fabricacao)
SELECT LEVEL,
       'LOTE-' || LPAD(LEVEL, 6, '0'),
       101 + MOD(LEVEL, 7),
       1 + MOD(LEVEL, 15),
       DATE '2026-01-01' + MOD(LEVEL, 320)
  FROM DUAL
CONNECT BY LEVEL <= 10000;

COMMIT;
```

### Anatomia do Gerador de Massa (`CONNECT BY LEVEL`)
| Expressão | Coluna | Comportamento |
| :--- | :--- | :--- |
| `LEVEL` | `id_op` | Contador sequencial de `1` a `10000` usado como PK. |
| `'LOTE-' \|\| LPAD(LEVEL, 6, '0')` | `codigo_lote` | Preenche zeros à esquerda até 6 dígitos (`LOTE-000001` a `LOTE-010000`). |
| `101 + MOD(LEVEL, 7)` | `id_produto` | Resto da divisão por 7 (`0..6`) somado a `101`, travando os IDs entre `101` e `107` (respeitando a FK). |
| `1 + MOD(LEVEL, 15)` | `qtd_produzida` | Garante quantidades entre `1` e `15` móveis por ordem (nunca zero). |
| `DATE '2026-01-01' + MOD(LEVEL, 320)` | `data_fabricacao` | Soma inteiros diretamente à data, espalhando as ordens ao longo de 320 dias em 2026. |

---

## 2. SQL Tuning na Prática (`EXPLAIN PLAN`, Índices e Function-Based Index)
- **`TABLE ACCESS FULL`:** Leitura integral de todos os blocos da tabela (lento em grandes volumes).
- **`INDEX RANGE SCAN`:** Acesso direto via árvore B-Tree apontando para o endereço exato das linhas.
- **Armadilha de ERP:** Aplicar funções diretamente sobre a coluna indexada no `WHERE` (ex: `UPPER(codigo_lote)`) invalida o índice comum e força o retorno para `TABLE ACCESS FULL`. A solução é remover a função da coluna ou criar um **Function-Based Index**.

```sql
-- 1. Antes do índice (TABLE ACCESS FULL)
EXPLAIN PLAN FOR
SELECT * FROM historico_producao WHERE codigo_lote = 'LOTE-008450';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- 2. Criando o índice B-Tree na coluna de busca
CREATE INDEX idx_hist_codigo_lote ON historico_producao (codigo_lote);

-- 3. Depois do índice (INDEX RANGE SCAN com queda drástica de Cost)
EXPLAIN PLAN FOR
SELECT * FROM historico_producao WHERE codigo_lote = 'LOTE-008450';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- 4. Armadilha: função na coluna invalida o índice tradicional
EXPLAIN PLAN FOR
SELECT * FROM historico_producao WHERE UPPER(codigo_lote) = 'LOTE-008450';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- 5. Solução Sênior: Function-Based Index
CREATE INDEX idx_hist_lote_upper ON historico_producao (UPPER(codigo_lote));

EXPLAIN PLAN FOR
SELECT * FROM historico_producao WHERE UPPER(codigo_lote) = 'LOTE-008450';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Dica Operacional no FreeSQL (Sessão Web e `PLAN_TABLE`)
Como o FreeSQL usa pool de conexões web, rodar `EXPLAIN PLAN FOR` e `DBMS_XPLAN.DISPLAY` em cliques separados pode alternar a sessão e exibir a `PLAN_TABLE` vazia. Formas garantidas de visualizar o plano:
1. **Visual:** Deixar apenas o `SELECT` puro e clicar no 3º ícone da barra superior (**Explain Plan**).
2. **Script:** Selecionar os dois comandos juntos e clicar em **Run Script** (2º botão da barra).
3. **Bloco PL/SQL (Mesma Sessão Garantida via `DBMS Output`):**
```sql
BEGIN
  EXECUTE IMMEDIATE 'EXPLAIN PLAN FOR SELECT * FROM historico_producao WHERE codigo_lote = ''LOTE-008450''';
  FOR r IN (SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY)) LOOP
    DBMS_OUTPUT.PUT_LINE(r.plan_table_output);
  END LOOP;
END;
/
```

---

## 3. Funções Analíticas / Window Functions (`OVER`, `PARTITION BY`, `DENSE_RANK`, `LAG`)
Permitem calcular rankings, somas acumuladas, percentuais e comparações entre linhas adjacentes **sem colapsar as linhas individuais** (ao contrário do `GROUP BY`).

### 3.1. Ranking, Total por Setor e Participação no Capital da Fábrica
```sql
SELECT l.nome AS linha,
       p.descricao,
       (p.preco_venda - p.preco_custo) AS lucro_unitario,
       DENSE_RANK() OVER (
         PARTITION BY p.id_linha 
         ORDER BY (p.preco_venda - p.preco_custo) DESC
       ) AS ranking_na_linha,
       SUM(p.preco_custo * p.estoque) OVER (PARTITION BY p.id_linha) AS capital_parado_linha,
       ROUND(
         (p.preco_custo * p.estoque) * 100 / SUM(p.preco_custo * p.estoque) OVER (), 
         2
       ) AS pct_capital_fabrica
  FROM produtos p
  JOIN linhas_moveis l ON l.id = p.id_linha
 ORDER BY l.nome, ranking_na_linha;
```

### 3.2. Auditoria de Evolução de Preços com `LAG()`
Compara cada reajuste com o registro imediatamente anterior do mesmo produto sem precisar de auto-join:

```sql
SELECT p.descricao,
       TO_CHAR(a.data_reajuste, 'DD/MM/YYYY') AS data_vigencia,
       LAG(a.preco_registrado, 1, 0) OVER (
         PARTITION BY a.id_produto 
         ORDER BY a.data_reajuste
       ) AS preco_anterior,
       a.preco_registrado AS preco_novo,
       a.preco_registrado - LAG(a.preco_registrado) OVER (
         PARTITION BY a.id_produto 
         ORDER BY a.data_reajuste
       ) AS aumento_reais
  FROM auditoria_precos a
  JOIN produtos p ON p.id = a.id_produto
 ORDER BY a.id_produto, a.data_reajuste;
```

---

## 4. Cirurgia de Erros (`FORMAT_ERROR_BACKTRACE`) e Controle Parcial (`SAVEPOINT`)
- **`DBMS_UTILITY.FORMAT_ERROR_BACKTRACE`:** Retorna a pilha e o número exato da linha onde a exceção foi disparada (diferente do `SQLERRM`, que informa apenas a mensagem do erro).
- **`SAVEPOINT` + `ROLLBACK TO`:** Cria um ponto de salvamento intermediário na transação, permitindo reverter apenas o passo que falhou e confirmar (`COMMIT`) o trabalho válido executado anteriormente.

```sql
CREATE OR REPLACE PROCEDURE prc_fechamento_cirurgico (
  p_id_movel_ok   IN NUMBER,
  p_id_movel_erro IN NUMBER,
  p_divisor       IN NUMBER
) IS
  v_preco_teste NUMBER;
BEGIN
  -- PASSO 1: Reajuste válido (+R$ 50)
  UPDATE produtos SET preco_venda = preco_venda + 50 WHERE id = p_id_movel_ok;
  DBMS_OUTPUT.PUT_LINE('Passo 1 executado para o móvel ' || p_id_movel_ok);

  -- CHECKPOINT intermediário
  SAVEPOINT sp_antes_passo_2;

  BEGIN
    -- PASSO 2: Operação arriscada
    UPDATE produtos SET preco_venda = preco_venda + 500 WHERE id = p_id_movel_erro;
    SELECT preco_custo / p_divisor INTO v_preco_teste FROM produtos WHERE id = p_id_movel_erro;
  EXCEPTION
    WHEN OTHERS THEN
      -- Reverte apenas o Passo 2, preservando o Passo 1
      ROLLBACK TO sp_antes_passo_2;
      
      DBMS_OUTPUT.PUT_LINE('🚨 FALHA CIRÚRGICA DETECTADA NO PASSO 2:');
      DBMS_OUTPUT.PUT_LINE('Erro: ' || SQLERRM);
      DBMS_OUTPUT.PUT_LINE('Linha Exata (Backtrace): ' || DBMS_UTILITY.FORMAT_ERROR_BACKTRACE);
  END;

  COMMIT;
  DBMS_OUTPUT.PUT_LINE('✅ Fechamento concluído: Passo 1 salvo no disco, Passo 2 revertido.');
END prc_fechamento_cirurgico;
/
```
