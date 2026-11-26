# Exercícios da Aula 8 - SQL Tuning, Window Functions e Controle Cirúrgico de Erros

## Exercício 1: Relatório "Campeões de Lucro por Linha" (CTE `WITH` + `DENSE_RANK`)
**Cenário:** Exibir apenas o Top 1 móvel mais lucrativo de cada linha (`Cozinha e Jantar`, `Dormitórios`, `Corporativo e Office`), utilizando uma CTE (`WITH`) para filtrar o resultado da Window Function `DENSE_RANK()`.

```sql
WITH ranking_moveis AS (
  SELECT l.nome AS linha,
         p.descricao,
         (p.preco_venda - p.preco_custo) AS lucro_unitario,
         DENSE_RANK() OVER (
           PARTITION BY p.id_linha 
           ORDER BY (p.preco_venda - p.preco_custo) DESC
         ) AS posicao
    FROM produtos p
    JOIN linhas_moveis l ON l.id = p.id_linha
)
SELECT linha, descricao, lucro_unitario
  FROM ranking_moveis
 WHERE posicao = 1
 ORDER BY lucro_unitario DESC;
```

---

## Exercício 2: Análise de Produção Acumulada no Mês (`SUM() OVER`)
**Cenário:** Consultar na tabela `historico_producao` todas as ordens do produto `101` entre `01/01/2026` e `10/01/2026`, exibindo a quantidade produzida em cada ordem e o total acumulado progressivo ao longo do período.

```sql
SELECT id_op,
       TO_CHAR(data_fabricacao, 'DD/MM/YYYY') AS data_fab,
       qtd_produzida,
       SUM(qtd_produzida) OVER (
         ORDER BY data_fabricacao, id_op
       ) AS total_acumulado_mes
  FROM historico_producao
 WHERE id_produto = 101
   AND data_fabricacao BETWEEN DATE '2026-01-01' AND DATE '2026-01-10'
 ORDER BY data_fabricacao, id_op;
```

---

## Exercício 3: Teste de Falha Parcial com `SAVEPOINT` e `FORMAT_ERROR_BACKTRACE`
**Cenário:** Executar a procedure `prc_fechamento_cirurgico` provocando divisão por zero no Passo 2 (`p_divisor => 0`), validando que o aumento de R$ 50 no móvel `101` é persistido, o aumento de R$ 500 no móvel `102` é desfeito e a linha exata do erro é impressa no console.

```sql
BEGIN
  prc_fechamento_cirurgico(
    p_id_movel_ok   => 101,
    p_id_movel_erro => 102,
    p_divisor       => 0
  );
END;
/

SELECT id, descricao, preco_venda
  FROM produtos
 WHERE id IN (101, 102);
```
