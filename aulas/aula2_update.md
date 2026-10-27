# Aula 2 (Parte 2): Atualizando Dados com Cursores

## 1. O Conceito: FOR UPDATE e WHERE CURRENT OF
Quando lemos dados em lote para fazer alterações, corremos o risco de outro usuário ou sistema alterar a mesma linha simultaneamente. Para evitar isso, adicionamos `FOR UPDATE` na declaração do cursor. Isso cria um "lock" (trava) exclusivo nas linhas selecionadas. 

Para atualizar a linha exata que o cursor está processando naquele instante, usamos a cláusula `WHERE CURRENT OF`. 
**Nota de sintaxe:** O `WHERE CURRENT OF` exige que o cursor tenha um nome explícito no bloco `DECLARE`.

## 2. A Analogia
O `FOR UPDATE` é como colocar uma placa de "Reservado" na sua mesa do restaurante; ninguém mais senta lá até você sair. O `WHERE CURRENT OF` é dizer ao garçom: *"Traga o pedido exatamente para esta mesa que reservei"*, em vez de fazê-lo procurar seu nome no restaurante inteiro.

## 3. Exemplo Prático de Código

```sql
DECLARE
  -- O cursor precisa ser nomeado para usarmos o WHERE CURRENT OF
  CURSOR c_produtos IS 
    SELECT id, descricao, preco 
      FROM produtos 
       FOR UPDATE;
BEGIN
  FOR r_prod IN c_produtos LOOP
    
    -- Regra: Aumentar em 20% os produtos que custam menos de 150
    IF r_prod.preco < 150 THEN
      
      UPDATE produtos 
         SET preco = preco * 1.20 
       WHERE CURRENT OF c_produtos;
       
      DBMS_OUTPUT.PUT_LINE('Preço atualizado: ' || r_prod.descricao);
      
    END IF;
  END LOOP;
  -- O COMMIT salva as alterações e libera a trava das linhas
  COMMIT; 
END;
```

## 4. Aplicação Corporativa
Em sistemas bancários, durante a rotina de rendimento da poupança, o PL/SQL abre um cursor `FOR UPDATE` nas contas. Isso garante que, enquanto o banco deposita os juros, um saque simultâneo no caixa eletrônico não consiga alterar o saldo daquela mesma conta, evitando inconsistências financeiras.
