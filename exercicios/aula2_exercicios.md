# Exercícios da Aula 2 - Resoluções e Explicações

## Exercício 1: Tratamento de Exceções
**Cenário:** Faça um `SELECT INTO` sem cláusula `WHERE` para forçar o erro `TOO_MANY_ROWS` e trate a exceção.
```sql
DECLARE
  v_desc VARCHAR2(50);
BEGIN
  SELECT descricao INTO v_desc FROM produtos;
EXCEPTION
  WHEN TOO_MANY_ROWS THEN
    DBMS_OUTPUT.PUT_LINE('Atenção: Filtro de busca inválido. Mais de um produto retornado.');
END;
```

## Exercício 2: Reajuste de Preços (FOR LOOP)
**Cenário:** Simular no console um reajuste de 10% em todos os produtos usando um Cursor FOR LOOP.
```sql
BEGIN
  FOR r_produto IN (SELECT descricao, preco FROM produtos) LOOP
    DBMS_OUTPUT.PUT_LINE('[' || r_produto.descricao || '] Antigo: ' || r_produto.preco || ' -> Novo: ' || (r_produto.preco * 1.10));
  END LOOP;
END;
```

## Exercício 3: Queima de Estoque (FOR UPDATE)
**Cenário:** Aplicar 10% de desconto na base de dados para produtos acima de R$ 300,00, usando bloqueio seguro de linha.
```sql
DECLARE
  CURSOR c_estoque IS SELECT descricao, preco FROM produtos FOR UPDATE;
BEGIN
  FOR r_item IN c_estoque LOOP
    IF r_item.preco > 300 THEN
      UPDATE produtos SET preco = preco * 0.90 WHERE CURRENT OF c_estoque;
      DBMS_OUTPUT.PUT_LINE('Desconto aplicado: ' || r_item.descricao);
    END IF;
  END LOOP;
  COMMIT;
END;
```
