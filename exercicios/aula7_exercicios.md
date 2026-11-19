# Exercícios da Aula 7 - `JSON_TABLE`, `MERGE INTO`, `SYS_REFCURSOR` e n8n

## Exercício 1: Importador de Pedidos em JSON (`JSON_TABLE` + `INSERT INTO ... SELECT`)
**Cenário:** Criar a procedure `prc_importar_pedido_json` que recebe um pedido estruturado em `CLOB` (contendo `num_pedido` e um array `itens`) e grava todos os itens diretamente na tabela `itens_pedido` em um único comando SQL.

```sql
CREATE OR REPLACE PROCEDURE prc_importar_pedido_json (
  p_json IN CLOB
) IS
BEGIN
  INSERT INTO itens_pedido (num_pedido, id_produto, quantidade)
  SELECT jt.num_pedido, jt.id_produto, jt.quantidade
    FROM JSON_TABLE(
           p_json, '$'
           COLUMNS (
             num_pedido NUMBER PATH '$.num_pedido',
             NESTED PATH '$.itens[*]'
             COLUMNS (
               id_produto NUMBER PATH '$.id_produto',
               quantidade NUMBER PATH '$.quantidade'
             )
           )
         ) jt;

  DBMS_OUTPUT.PUT_LINE('Pedido importado com sucesso! Itens gravados: ' || SQL%ROWCOUNT);
  COMMIT;
END prc_importar_pedido_json;
/

-- Teste prático:
BEGIN
  prc_importar_pedido_json('{
    "num_pedido": 801,
    "itens": [
      {"id_produto": 102, "quantidade": 2},
      {"id_produto": 104, "quantidade": 1}
    ]
  }');
END;
/

SELECT * FROM itens_pedido;
```

---

## Exercício 2: Alerta de Reposição para o PCP via `SYS_REFCURSOR`
**Cenário:** Criar a procedure `prc_estoque_critico` que receba `p_limite_minimo IN NUMBER` e devolva via `p_cursor OUT SYS_REFCURSOR` os móveis com estoque menor ou igual ao limite informado, ordenados pelo menor saldo.

```sql
CREATE OR REPLACE PROCEDURE prc_estoque_critico (
  p_limite_minimo IN  NUMBER,
  p_cursor        OUT SYS_REFCURSOR
) IS
BEGIN
  OPEN p_cursor FOR
    SELECT id, descricao, estoque
      FROM produtos
     WHERE estoque <= p_limite_minimo
     ORDER BY estoque ASC;
END prc_estoque_critico;
/

-- Bloco consumidor de teste (limite = 10 unidades):
DECLARE
  v_cur     SYS_REFCURSOR;
  v_id      produtos.id%TYPE;
  v_desc    produtos.descricao%TYPE;
  v_estoque produtos.estoque%TYPE;
BEGIN
  prc_estoque_critico(p_limite_minimo => 10, p_cursor => v_cur);

  LOOP
    FETCH v_cur INTO v_id, v_desc, v_estoque;
    EXIT WHEN v_cur%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE('⚠️ REPOR PRODUÇÃO: [' || v_id || '] ' || v_desc || ' (Saldo: ' || v_estoque || ')');
  END LOOP;

  CLOSE v_cur;
END;
/
```

---

## Exercício 3: Teste Ponta a Ponta — Formulário n8n ➔ E-mail ➔ `MERGE INTO` no Oracle
**Cenário:** Validar o comportamento dual do `MERGE INTO` (Upsert) utilizando o payload JSON gerado pelo workflow `/aulas/aula7_workflow_n8n.json` no n8n:
1. **Lançamento Novo (`WHEN NOT MATCHED` ➔ `INSERT`):** Submeter o ID `107` no formulário web do n8n e executar o bloco recebido por e-mail.
2. **Atualização + Soma de Saldo (`WHEN MATCHED` ➔ `UPDATE`):** Submeter o ID `101` (já existente) no formulário web do n8n com novos valores e entrada de estoque, validando o incremento automático do saldo.

```sql
-- 1. Exemplo de comando gerado pelo n8n para lançamento novo (INSERT):
BEGIN
  prc_sincronizar_catalogo_json('[{"id":107,"id_linha":1,"descricao":"Painel Ripado TV 220cm","preco_custo":700,"preco_venda":1490,"estoque":8}]');
END;
/

-- 2. Exemplo de comando gerado pelo n8n para produto existente (UPDATE + soma estoque):
BEGIN
  prc_sincronizar_catalogo_json('[{"id":101,"id_linha":1,"descricao":"Mesa de Jantar 6 Lugares Carvalho","preco_custo":900,"preco_venda":1850,"estoque":5}]');
END;
/

-- 3. Conferência final no catálogo:
SELECT id, descricao, preco_custo, preco_venda, estoque
  FROM produtos
 ORDER BY id;
```
