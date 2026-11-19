# Aula 7: Integração Corporativa — `JSON_TABLE`, `MERGE INTO`, `SYS_REFCURSOR` e Automação n8n ➔ Oracle

Na Aula 6, o Oracle enviou dados estruturados para fora (`JSON_OBJECT` ➔ n8n). Nesta aula, fechamos o ciclo completo de integração de um ERP moveleiro moderno: **receber dados em JSON**, sincronizar o catálogo de forma atômica (**Upsert**), devolver listas dinâmicas para aplicações externas e acionar todo esse pipeline a partir de um **formulário web no n8n**.

---

## 1. Script Base de Restauração do Ambiente (FreeSQL)
```sql
DROP TABLE itens_pedido CASCADE CONSTRAINTS;
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

CREATE TABLE itens_pedido (
    id_item NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    num_pedido NUMBER NOT NULL,
    id_produto NUMBER REFERENCES produtos(id),
    quantidade NUMBER NOT NULL
);

INSERT INTO linhas_moveis VALUES (1, 'Cozinha e Jantar');
INSERT INTO linhas_moveis VALUES (2, 'Dormitórios');
INSERT INTO linhas_moveis VALUES (3, 'Corporativo e Office');

INSERT INTO produtos VALUES (101, 1, 'Mesa de Jantar 6 Lugares Carvalho', 850.00, 1690.00, 12);
INSERT INTO produtos VALUES (102, 2, 'Roupeiro Casal 6 Portas MDF 18mm', 1400.00, 2890.00, 8);
INSERT INTO produtos VALUES (103, 1, 'Cristaleira Rústica Vidro Temperado', 620.00, 1250.00, 5);
INSERT INTO produtos VALUES (104, 3, 'Mesa Diretor em L com Gaveteiro', 580.00, 1150.00, 20);
INSERT INTO produtos VALUES (105, 2, 'Cômoda 5 Gavetas Corrediça Telescópica', 390.00, 790.00, 15);
COMMIT;
```

---

## 2. `JSON_TABLE` e `NESTED PATH` (Desembalando Payloads JSON no SQL)
Permite converter um documento JSON (`CLOB`/`VARCHAR2`) em linhas e colunas relacionais diretamente no motor SQL, eliminando manipulações manuais de strings.
- **Analogia:** Enquanto o `JSON_OBJECT` é a embaladora na expedição, o `JSON_TABLE` é o setor de recebimento abrindo o caminhão e separando cada item na prateleira correta.
- **`NESTED PATH`:** Percorre automaticamente arrays aninhados (ex: os itens dentro de um pedido).

```sql
DECLARE
  v_json_pedido CLOB := '{
    "num_pedido": 750,
    "cliente": "Móveis Serra Gaúcha Ltda",
    "itens": [
      {"id_produto": 101, "quantidade": 2},
      {"id_produto": 104, "quantidade": 3},
      {"id_produto": 105, "quantidade": 1}
    ]
  }';
BEGIN
  FOR r IN (
    SELECT jt.num_pedido, jt.cliente, jt.id_produto, jt.quantidade
      FROM JSON_TABLE(
             v_json_pedido, '$'
             COLUMNS (
               num_pedido NUMBER        PATH '$.num_pedido',
               cliente    VARCHAR2(100) PATH '$.cliente',
               NESTED PATH '$.itens[*]'
               COLUMNS (
                 id_produto NUMBER PATH '$.id_produto',
                 quantidade NUMBER PATH '$.quantidade'
               )
             )
           ) jt
  ) LOOP
    DBMS_OUTPUT.PUT_LINE(
      'Pedido #' || r.num_pedido || ' (' || r.cliente || ') -> ' ||
      'Móvel ID: ' || r.id_produto || ' | Qtd: ' || r.quantidade
    );
  END LOOP;
END;
/
```

---

## 3. `MERGE INTO` (Upsert Corporativo de Alta Performance)
Executa `UPDATE` nos registros que já existem (`WHEN MATCHED`) e `INSERT` nos lançamentos novos (`WHEN NOT MATCHED`) em uma única operação atômica no banco, substituindo verificações manuais linha a linha.

### 3.1. Combo Corporativo: `MERGE INTO` + `JSON_TABLE`
Procedure corporativa pronta para sincronizar catálogos e entradas de estoque recebidos em formato JSON (via API, ERP ou n8n):

```sql
CREATE OR REPLACE PROCEDURE prc_sincronizar_catalogo_json (
  p_json_catalogo IN CLOB
) IS
BEGIN
  MERGE INTO produtos p
  USING (
    SELECT jt.id, jt.id_linha, jt.descricao, jt.preco_custo, jt.preco_venda, jt.estoque
      FROM JSON_TABLE(
             p_json_catalogo, '$[*]'
             COLUMNS (
               id          NUMBER        PATH '$.id',
               id_linha    NUMBER        PATH '$.id_linha',
               descricao   VARCHAR2(100) PATH '$.descricao',
               preco_custo NUMBER(10,2)  PATH '$.preco_custo',
               preco_venda NUMBER(10,2)  PATH '$.preco_venda',
               estoque     NUMBER        PATH '$.estoque'
             )
           ) jt
  ) origem
  ON (p.id = origem.id)
  WHEN MATCHED THEN
    UPDATE SET
      p.preco_custo = origem.preco_custo,
      p.preco_venda = origem.preco_venda,
      p.estoque     = p.estoque + origem.estoque -- Soma nova produção ao saldo atual
  WHEN NOT MATCHED THEN
    INSERT (id, id_linha, descricao, preco_custo, preco_venda, estoque)
    VALUES (origem.id, origem.id_linha, origem.descricao, origem.preco_custo, origem.preco_venda, origem.estoque);

  DBMS_OUTPUT.PUT_LINE('Sincronização concluída! Total de itens inseridos/atualizados: ' || SQL%ROWCOUNT);
  COMMIT;
END prc_sincronizar_catalogo_json;
/
```

---

## 4. `SYS_REFCURSOR` (Retornando Listas Dinâmicas via `PROCEDURE`)
Quando uma tela do ERP, relatório ou API precisa chamar uma `PROCEDURE` e receber um conjunto completo de linhas (result set), utilizamos um parâmetro `OUT` do tipo **`SYS_REFCURSOR`**.
- **Analogia:** Entrega um ponteiro para a gaveta do arquivo aberta; a aplicação consumidora faz o `FETCH` linha por linha e obrigatoriamente executa o `CLOSE` ao terminar para liberar memória.

```sql
CREATE OR REPLACE PROCEDURE prc_relatorio_linha (
  p_id_linha IN  NUMBER,
  p_cursor   OUT SYS_REFCURSOR
) IS
BEGIN
  OPEN p_cursor FOR
    SELECT id, descricao, preco_venda, estoque
      FROM produtos
     WHERE id_linha = p_id_linha
     ORDER BY preco_venda DESC;
END prc_relatorio_linha;
/

-- Bloco consumidor simulando a aplicação externa (ERP/API):
DECLARE
  v_cur       SYS_REFCURSOR;
  v_id        produtos.id%TYPE;
  v_descricao produtos.descricao%TYPE;
  v_preco     produtos.preco_venda%TYPE;
  v_estoque   produtos.estoque%TYPE;
BEGIN
  prc_relatorio_linha(p_id_linha => 1, p_cursor => v_cur);

  LOOP
    FETCH v_cur INTO v_id, v_descricao, v_preco, v_estoque;
    EXIT WHEN v_cur%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE('[' || v_id || '] ' || v_descricao || ' | R$ ' || v_preco || ' | Saldo: ' || v_estoque);
  END LOOP;

  CLOSE v_cur;
END;
/
```

---

## 5. Laboratório Prático de Integração: Formulário Web n8n ➔ Oracle `MERGE INTO`
Invertendo o fluxo da Aula 6, construímos uma automação completa no **n8n** simulando o setor de Engenharia de Produto alimentando o catálogo da fábrica sem acesso direto ao banco de dados.

### Arquitetura do Fluxo (3 Nós — arquivo `/aulas/aula7_workflow_n8n.json`)
`[1. Form Trigger (Formulário Engenharia)]` ➔ `[2. Code (Montar JSON + Margem + Bloco PL/SQL)]` ➔ `[3. Send Email (SMTP)]`

1. **Nó 1 — Formulário Engenharia (`n8n-nodes-base.formTrigger`):** Gera uma interface web pública/interna para entrada de ID do Móvel, Linha, Descrição, Custo, Preço de Venda e Entrada de Estoque.
2. **Nó 2 — Montar JSON e PL/SQL (`n8n-nodes-base.code`):** Valida tipos numéricos, calcula a margem bruta (`APROVADO` vs `ABAIXO DO CUSTO`), serializa o array JSON com tratamento de aspas simples (`replace(/'/g, "''")`) e gera o bloco anônimo pronto chamando `prc_sincronizar_catalogo_json`.
3. **Nó 3 — Enviar Carga por E-mail (`n8n-nodes-base.emailSend`):** Despacha via SMTP a confirmação da carga com o JSON estruturado e o comando pronto para execução no FreeSQL.
4. **Visão de Produção (Rede Interna da Empresa):** Em ambiente corporativo on-premise/VPN, o 3º nó pode ser substituído ou complementado pelo nó nativo **Oracle Database** do n8n executando diretamente `BEGIN prc_sincronizar_catalogo_json(:1); END;`, eliminando planilhas e redigitação no ERP.
