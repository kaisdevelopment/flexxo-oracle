# Exercícios da Aula 5 - Triggers Avançadas, Bulk Collect e Hierarquia

## Exercício 1: Trava de Estoque Negativo no Pedido com Log Autônomo
**Cenário:** Impedir que um item de pedido seja inserido ou atualizado na tabela `itens_pedido` caso a quantidade solicitada seja maior que o estoque disponível em `produtos`. Antes de bloquear com `RAISE_APPLICATION_ERROR`, registrar a tentativa na tabela `log_tentativas_bloqueadas` usando a procedure autônoma `prc_gravar_alerta_fraude`.

```sql
CREATE OR REPLACE TRIGGER trg_valida_saldo_pedido
BEFORE INSERT OR UPDATE ON itens_pedido
FOR EACH ROW
DECLARE
  v_estoque_disponivel produtos.estoque%TYPE;
  v_nome_movel         produtos.descricao%TYPE;
BEGIN
  SELECT descricao, estoque 
    INTO v_nome_movel, v_estoque_disponivel
    FROM produtos
   WHERE id = :NEW.id_produto;
   
  IF :NEW.quantidade > v_estoque_disponivel THEN
    prc_gravar_alerta_fraude(
      :NEW.id_produto, 
      'Falta de estoque no pedido ' || :NEW.num_pedido || ': pediu ' || :NEW.quantidade || ', tinha ' || v_estoque_disponivel
    );
    
    RAISE_APPLICATION_ERROR(-20005, 
      'VENDA NÃO AUTORIZADA: O pedido exige ' || :NEW.quantidade || 
      ' un. de [' || v_nome_movel || '], mas só existem ' || v_estoque_disponivel || ' no galpão.');
  END IF;
END;
/

-- Teste do Bloqueio e do Log Autônomo:
INSERT INTO itens_pedido (num_pedido, id_produto, quantidade) VALUES (999, 101, 50);
SELECT * FROM log_tentativas_bloqueadas;
```

---

## Exercício 2: Relatório VIP em Memória com `BULK COLLECT`
**Cenário:** Carregar em uma única viagem ao banco todos os móveis com preço de venda superior a R$ 1.000,00 para uma Collection em memória RAM e listar os itens no console.

```sql
DECLARE
  TYPE t_moveis_vip IS TABLE OF produtos%ROWTYPE;
  v_lista_vip t_moveis_vip;
BEGIN
  SELECT * 
    BULK COLLECT INTO v_lista_vip 
    FROM produtos 
   WHERE preco_venda > 1000;
   
  DBMS_OUTPUT.PUT_LINE('--- MÓVEIS ACIMA DE R$ 1.000 (' || v_lista_vip.COUNT || ' encontrados) ---');
  
  FOR i IN 1 .. v_lista_vip.COUNT LOOP
    DBMS_OUTPUT.PUT_LINE('Móvel: ' || v_lista_vip(i).descricao || ' | Preço: R$ ' || v_lista_vip(i).preco_venda);
  END LOOP;
END;
/
```

---

## Exercício 3: Rastreamento de Erro em Árvore de Engenharia (Caso Real Builder)
**Cenário:** Simular a captura de erro de um componente de 5º nível (`id_componente = 5001`), descobrir o móvel principal afetado (Pai Raiz de Nível 1) via query hierárquica invertida e gravar o diagnóstico completo na tabela `log_tentativas_bloqueadas` usando `PRAGMA AUTONOMOUS_TRANSACTION`.

```sql
DECLARE
  v_id_erro       NUMBER := 5001;
  v_id_pai_raiz   NUMBER;
  v_nome_pai_raiz VARCHAR2(100);
  v_trilha        VARCHAR2(500);
BEGIN
  SELECT id_componente, 
         descricao,
         SYS_CONNECT_BY_PATH(descricao, ' <- ')
    INTO v_id_pai_raiz, 
         v_nome_pai_raiz,
         v_trilha
    FROM estrutura_builder
   WHERE id_pai IS NULL
   START WITH id_componente = v_id_erro
  CONNECT BY PRIOR id_pai = id_componente;

  -- Registra no log autônomo qual móvel raiz foi impactado pelo componente de 5º nível
  prc_gravar_alerta_fraude(
    p_id_produto => v_id_pai_raiz,
    p_mensagem   => 'ERRO BUILDER no item ' || v_id_erro || ' | Pai Raiz: ' || v_nome_pai_raiz || ' | Trilha: ' || v_trilha
  );
  
  DBMS_OUTPUT.PUT_LINE('Log de engenharia registrado com sucesso para o Pai Raiz ID: ' || v_id_pai_raiz);
END;
/

SELECT * FROM log_tentativas_bloqueadas ORDER BY id_alerta DESC;
```
