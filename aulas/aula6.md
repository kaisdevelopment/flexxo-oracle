# Aula 6: Gravação em Lote (FORALL), Compound Triggers, JSON Nativo e Integração n8n

## 1. Gravação em Massa na Memória: `FORALL` e `SAVE EXCEPTIONS`
Enquanto o `BULK COLLECT` carrega dados do disco para a memória RAM em uma única viagem, o comando `FORALL` envia todas as alterações (`INSERT`, `UPDATE`, `DELETE`) de uma Collection de volta para o motor SQL de uma só vez, eliminando o gargalo de troca de contexto linha a linha.

**Diferencial Corporativo (`SAVE EXCEPTIONS`):** Permite que um processamento em lote continue mesmo se algumas linhas falharem (por violação de constraint ou trigger). As linhas válidas são processadas e os erros ficam isolados na coleção interna `SQL%BULK_EXCEPTIONS` (capturados pelo erro `ORA-24381`).

```sql
-- Adicionando constraint para testar violação em lote:
ALTER TABLE produtos ADD CONSTRAINT chk_margem_positiva CHECK (preco_venda >= preco_custo);

DECLARE
  TYPE t_lista_ids IS TABLE OF produtos.id%TYPE;
  v_ids t_lista_ids := t_lista_ids(101, 102, 103, 104, 105);
  
  e_erro_lote EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_erro_lote, -24381);
BEGIN
  -- Reduz R$ 700 de todos os móveis da lista de uma só vez, salvando exceções
  FORALL i IN 1 .. v_ids.COUNT SAVE EXCEPTIONS
    UPDATE produtos
       SET preco_venda = preco_venda - 700
     WHERE id = v_ids(i);

EXCEPTION
  WHEN e_erro_lote THEN
    DBMS_OUTPUT.PUT_LINE('⚠️ Processamento em lote concluído com ressalvas!');
    DBMS_OUTPUT.PUT_LINE('Total de itens que falharam: ' || SQL%BULK_EXCEPTIONS.COUNT);
    
    FOR j IN 1 .. SQL%BULK_EXCEPTIONS.COUNT LOOP
      DBMS_OUTPUT.PUT_LINE(
        'Erro no índice [' || SQL%BULK_EXCEPTIONS(j).ERROR_INDEX || 
        '] -> Móvel ID: ' || v_ids(SQL%BULK_EXCEPTIONS(j).ERROR_INDEX) ||
        ' | Código Oracle: ORA-' || SQL%BULK_EXCEPTIONS(j).ERROR_CODE
      );
    END LOOP;
    
    COMMIT; -- Confirma os itens que não deram erro!
END;
/
```

---

## 2. `COMPOUND TRIGGER`: Resolvendo a Tabela Mutante (`ORA-04091`)
Uma `COMPOUND TRIGGER` permite combinar diferentes momentos de disparo (`BEFORE STATEMENT`, `BEFORE EACH ROW`, `AFTER EACH ROW` e `AFTER STATEMENT`) dentro de um único corpo, compartilhando variáveis na memória. 
Isso resolve o erro `ORA-04091`, pois podemos capturar dados linha a linha e deixar para executar o `SELECT` na própria tabela apenas no bloco `AFTER STATEMENT`, quando a tabela já não está mais em mutação.

```sql
CREATE OR REPLACE TRIGGER trg_controle_disparidade_precos
FOR INSERT OR UPDATE OF preco_venda ON produtos
COMPOUND TRIGGER

  v_menor_preco NUMBER;
  v_maior_preco NUMBER;

  BEFORE EACH ROW IS
  BEGIN
    DBMS_OUTPUT.PUT_LINE('Validando linha do móvel: ' || :NEW.descricao);
  END BEFORE EACH ROW;

  AFTER STATEMENT IS
  BEGIN
    -- Consulta a própria tabela produtos sem disparar ORA-04091
    SELECT MIN(preco_venda), MAX(preco_venda)
      INTO v_menor_preco, v_maior_preco
      FROM produtos;
      
    IF v_maior_preco > (v_menor_preco * 10) THEN
      RAISE_APPLICATION_ERROR(-20010, 
        'TRAVA DE TABELA: O maior preço (R$ ' || v_maior_preco || 
        ') ultrapassou 10x o menor preço da fábrica (R$ ' || v_menor_preco || ')!');
    END IF;
  END AFTER STATEMENT;

END trg_controle_disparidade_precos;
/
```

---

## 3. Oracle + JSON Nativo (`JSON_OBJECT`) para Automação e Webhooks (n8n)
O Oracle possui suporte nativo para geração e leitura de documentos JSON diretamente em queries SQL e blocos PL/SQL, facilitando a integração com orquestradores modernos como o **n8n**.

```sql
SELECT JSON_OBJECT(
         'id_movel'    VALUE p.id,
         'produto'     VALUE p.descricao,
         'linha'       VALUE l.nome,
         'financeiro'  VALUE JSON_OBJECT(
                               'custo_fabrica' VALUE p.preco_custo,
                               'preco_lojista' VALUE p.preco_venda,
                               'margem_lucro'  VALUE (p.preco_venda - p.preco_custo)
                             ),
         'saldo_atual' VALUE p.estoque
       ) AS payload_webhook_n8n
  FROM produtos p
  JOIN linhas_moveis l ON l.id = p.id_linha
 WHERE p.id = 101;
```

### 3.1. Empacotando Logs de Falha / Builder para o n8n
Combinando `PRAGMA AUTONOMOUS_TRANSACTION` (visto na Aula 5) com `JSON_OBJECT`, podemos extrair o último erro registrado na fábrica pronto para envio HTTP:

```sql
CREATE OR REPLACE FUNCTION fnc_exportar_log_n8n RETURN VARCHAR2 IS
  v_payload_json VARCHAR2(2000);
BEGIN
  SELECT JSON_OBJECT(
           'origem'        VALUE 'ERP_MOVELEIRO_ORACLE',
           'id_alerta'     VALUE l.id_alerta,
           'data_hora'     VALUE TO_CHAR(l.data_ocorrencia, 'DD/MM/YYYY HH24:MI:SS'),
           'usuario_banco' VALUE l.usuario,
           'id_movel'      VALUE l.id_produto,
           'movel_afetado' VALUE NVL(p.descricao, 'Componente Builder 5º Nível'),
           'detalhe_erro'  VALUE l.mensagem_alerta
         )
    INTO v_payload_json
    FROM log_tentativas_bloqueadas l
    LEFT JOIN produtos p ON p.id = l.id_produto
   ORDER BY l.id_alerta DESC
   FETCH FIRST 1 ROW ONLY;

  RETURN v_payload_json;
END;
/
```
