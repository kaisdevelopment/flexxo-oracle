# Exercícios da Aula 6 - FORALL, JSON Nativo e Monitoramento com n8n

## Exercício 1: Reajuste em Massa com `BULK COLLECT` e `FORALL`
**Cenário:** Reajustar em 15% o `preco_custo` e o `preco_venda` de todos os móveis da linha Dormitórios (`id_linha = 2`), carregando os IDs para a memória RAM de uma só vez e aplicando a atualização em lote com `FORALL`.

```sql
DECLARE
  TYPE t_ids_dormitorio IS TABLE OF produtos.id%TYPE;
  v_ids t_ids_dormitorio;
BEGIN
  SELECT id 
    BULK COLLECT INTO v_ids 
    FROM produtos 
   WHERE id_linha = 2;
   
  FORALL i IN 1 .. v_ids.COUNT
    UPDATE produtos
       SET preco_custo = ROUND(preco_custo * 1.15, 2),
           preco_venda = ROUND(preco_venda * 1.15, 2)
     WHERE id = v_ids(i);
     
  DBMS_OUTPUT.PUT_LINE('Total de móveis de Dormitório reajustados em lote: ' || SQL%ROWCOUNT);
  COMMIT;
END;
/
```

---

## Exercício 2: Function Geradora de Alerta JSON (`fnc_gerar_json_alerta`)
**Cenário:** Criar uma Function que receba o ID do móvel e o detalhe de erro (ex: falha no 5º nível do Builder) e devolva um JSON padronizado pronto para disparo de Webhook.

```sql
CREATE OR REPLACE FUNCTION fnc_gerar_json_alerta (
  p_id_produto  IN NUMBER,
  p_motivo_erro IN VARCHAR2
) RETURN VARCHAR2 IS
  v_json_retorno VARCHAR2(1000);
BEGIN
  SELECT JSON_OBJECT(
           'sistema'      VALUE 'ERP_MOVELEIRO_BUILDER',
           'data_alerta'  VALUE TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS'),
           'id_item'      VALUE id,
           'descricao'    VALUE descricao,
           'estoque'      VALUE estoque,
           'erro_detalhe' VALUE p_motivo_erro
         )
    INTO v_json_retorno
    FROM produtos
   WHERE id = p_id_produto;
   
  RETURN v_json_retorno;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    RETURN '{"erro": "Produto nao localizado na base"}';
END fnc_gerar_json_alerta;
/

-- Geração do Payload no FreeSQL:
SELECT fnc_gerar_json_alerta(102, 'Falha na explosão de engenharia no 5º nível (Parafuso sem custo)') AS json_para_n8n 
  FROM DUAL;
```

---

## Exercício 3: Integração Prática Oracle ➔ Webhook n8n ➔ Alerta por E-mail
**Cenário:** Disparar o payload JSON gerado pelo Oracle para um Webhook no n8n (`POST /alerta-oracle`) que formata e envia automaticamente um alerta crítico por e-mail para a engenharia/gerência.

```bash
# Disparo via cURL simulando o conector do banco enviando o JSON gerado pela Function:
curl -X POST https://n8n.kaisdigital.com.br/webhook-test/alerta-oracle \
  -H "Content-Type: application/json" \
  -d '{"sistema":"ERP_MOVELEIRO_BUILDER","data_alerta":"2026-11-17 00:16:08","id_item":102,"descricao":"Roupeiro Casal 6 Portas MDF 18mm","estoque":8,"erro_detalhe":"Falha na explosão de engenharia no 5º nível (Parafuso sem custo)"}'
```
