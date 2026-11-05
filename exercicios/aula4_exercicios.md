# Exercícios da Aula 4 - Packages e Triggers

## Exercício 1: Diagnóstico de Estoque no Bloqueio de Expedição (`PACKAGE`)
**Cenário:** Ao bloquear a expedição de um móvel por falta de estoque na procedure `pkg_expedicao.prc_baixar_estoque`, consultar a tabela `produtos` e exibir na mensagem de erro o nome do móvel e a quantidade exata disponível no galpão.

```sql
-- Trecho aplicado no ELSE da procedure prc_baixar_estoque:
SELECT descricao, estoque 
  INTO v_nome_movel, v_estoque_atual
  FROM produtos
 WHERE id = p_id_produto;

DBMS_OUTPUT.PUT_LINE('BLOQUEADO: Você pediu ' || p_qtd_pedido || 
                     ' un. de [' || v_nome_movel || 
                     '], mas só temos ' || v_estoque_atual || ' no galpão!');
```

---

## Exercício 2: Testes de Auditoria e Proteção de Margem (`TRIGGER`)
**Cenário:** Testar o disparo automático da auditoria aplicando 10% de aumento na linha de Cozinha e Jantar (`id_linha = 1`) e, em seguida, testar o bloqueio de margem tentando reduzir o preço do Roupeiro Casal (`id = 102`, custo R$ 1.400,00) para R$ 1.100,00.

```sql
-- 1. Reajuste válido de 10% (Dispara trg_auditoria_precos)
UPDATE produtos 
   SET preco_venda = preco_venda * 1.10 
 WHERE id_linha = 1;
COMMIT;

SELECT * FROM auditoria_precos;

-- 2. Tentativa de venda abaixo do custo (Bloqueado pela trg_protege_margem_moveis com erro ORA-20001)
UPDATE produtos 
   SET preco_venda = 1100.00 
 WHERE id = 102;
```

---

## Exercício 3: O "Fiscal do Galpão" (`BEFORE DELETE`)
**Cenário:** Impedir que um móvel seja excluído do cadastro (`DELETE`) caso ainda possua unidades físicas no galpão (`estoque > 0`), evitando estoque fantasma na fábrica.
*Regra de Ouro:* Em operações de `DELETE`, não existe o ponteiro `:NEW`, apenas o `:OLD`.

```sql
CREATE OR REPLACE TRIGGER trg_protege_exclusao_estoque
BEFORE DELETE ON produtos
FOR EACH ROW
BEGIN
  IF :OLD.estoque > 0 THEN
    RAISE_APPLICATION_ERROR(-20002, 
      'EXCLUSÃO BLOQUEADA: O móvel [' || :OLD.descricao || 
      '] ainda possui ' || :OLD.estoque || ' unidades físicas no galpão! Zere o estoque antes de tirar de linha.');
  END IF;
END;
/

-- Roteiro de Teste:
-- 1. Tentativa de excluir a Cristaleira Rústica (ID 103, possui 5 em estoque) -> Bloqueado com ORA-20002!
DELETE FROM produtos WHERE id = 103;

-- 2. Zerando o estoque da Cristaleira
UPDATE produtos SET estoque = 0 WHERE id = 103;

-- 3. Nova tentativa de exclusão com estoque zerado -> Exclusão permitida!
DELETE FROM produtos WHERE id = 103;
COMMIT;
```
