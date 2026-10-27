# Exercícios da Aula 2 (Parte 2) - Atualização com Cursores

## Exercício 1: Queima de Estoque
**Cenário:** A diretoria pediu um desconto de 10% aplicado direto no banco para todos os produtos que custem **mais de R$ 300,00**.

### Solução PL/SQL:
```sql
DECLARE
  CURSOR c_estoque IS 
    SELECT descricao, preco 
      FROM produtos 
       FOR UPDATE;
BEGIN
  FOR r_item IN c_estoque LOOP
    IF r_item.preco > 300 THEN
      
      UPDATE produtos 
         SET preco = preco * 0.90 
       WHERE CURRENT OF c_estoque;
       
      DBMS_OUTPUT.PUT_LINE('Desconto aplicado: ' || r_item.descricao);
      
    END IF;
  END LOOP;
  COMMIT;
END;
```
**Por que funciona assim:** A declaração do cursor `c_estoque` com `FOR UPDATE` garante bloqueio em nível de linha. O comando `UPDATE ... WHERE CURRENT OF c_estoque` aponta diretamente para o ponteiro de memória da linha lida no laço atual, tornando a alteração imediata e à prova de falhas de concorrência.
