# Aula 5: Triggers Avançadas, Transações Autônomas, Bulk Collect e Árvores Hierárquicas

## 1. Preparação Complementar do Ambiente
Adicionamos ao modelo da indústria moveleira as tabelas de movimentação de pedidos (`itens_pedido`), auditoria de bloqueios (`log_tentativas_bloqueadas`) e explosão de engenharia (`estrutura_builder`).

```sql
CREATE TABLE itens_pedido (
    id_item NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    num_pedido NUMBER NOT NULL,
    id_produto NUMBER REFERENCES produtos(id),
    quantidade NUMBER NOT NULL
);

CREATE TABLE log_tentativas_bloqueadas (
    id_alerta NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    data_ocorrencia DATE DEFAULT SYSDATE,
    usuario VARCHAR2(50) DEFAULT USER,
    id_produto NUMBER,
    mensagem_alerta VARCHAR2(300)
);
```

---

## 2. Trigger Multi-Eventos e Kardex Automático (`INSERTING`, `UPDATING`, `DELETING`)
Permite concentrar toda a regra de movimentação de estoque em um único objeto utilizando predicados booleanos de evento.

**Analogia:** Funciona como a catraca eletrônica da fábrica, que identifica se a pessoa está entrando (`INSERTING`), saindo (`DELETING`) ou alterando o registro (`UPDATING`) e ajusta a contagem automaticamente.

```sql
CREATE OR REPLACE TRIGGER trg_movimenta_estoque_pedido
AFTER INSERT OR UPDATE OR DELETE ON itens_pedido
FOR EACH ROW
BEGIN
  -- 1. Venda nova: baixa o estoque (:NEW)
  IF INSERTING THEN
    UPDATE produtos 
       SET estoque = estoque - :NEW.quantidade
     WHERE id = :NEW.id_produto;
     
  -- 2. Cancelamento de item: devolve ao estoque (:OLD)
  ELSIF DELETING THEN
    UPDATE produtos 
       SET estoque = estoque + :OLD.quantidade
     WHERE id = :OLD.id_produto;
     
  -- 3. Alteração de quantidade: estorna o antigo (:OLD) e desconta o novo (:NEW)
  ELSIF UPDATING THEN
    UPDATE produtos 
       SET estoque = (estoque + :OLD.quantidade) - :NEW.quantidade
     WHERE id = :NEW.id_produto;
  END IF;
END;
/
```

---

## 3. Transações Autônomas (`PRAGMA AUTONOMOUS_TRANSACTION`)
Quando uma Trigger dispara `RAISE_APPLICATION_ERROR`, o Oracle executa um `ROLLBACK` na transação principal, desfazendo inclusive eventuais `INSERTs` de log feitos dentro da própria Trigger. A diretiva `PRAGMA AUTONOMOUS_TRANSACTION` abre uma sub-transação independente que permite executar `COMMIT` do log de auditoria mesmo que a operação principal seja abortada.

**Analogia:** A transação principal é um navio que bateu no iceberg (`ROLLBACK`). A Transação Autônoma é um bote salva-vidas motorizado que leva a caixa-preta (o registro do log) e salva os dados em segurança (`COMMIT` isolado).

### 3.1. Procedure de Gravação Autônoma
```sql
CREATE OR REPLACE PROCEDURE prc_gravar_alerta_fraude (
  p_id_produto IN NUMBER,
  p_mensagem   IN VARCHAR2
) IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO log_tentativas_bloqueadas (id_produto, mensagem_alerta)
  VALUES (p_id_produto, p_mensagem);
  
  COMMIT; -- Confirma apenas o log, sem interferir na transação principal
END prc_gravar_alerta_fraude;
/
```

### 3.2. Acionando o Log Autônomo antes do Bloqueio
```sql
CREATE OR REPLACE TRIGGER trg_protege_margem_moveis
BEFORE INSERT OR UPDATE OF preco_venda, preco_custo ON produtos
FOR EACH ROW
DECLARE
  v_msg_erro VARCHAR2(300);
BEGIN
  IF :NEW.preco_venda < :NEW.preco_custo THEN
    v_msg_erro := 'Tentativa de venda abaixo do custo no móvel [' || :NEW.descricao || 
                  ']. Custo: R$ ' || :NEW.preco_custo || ' | Tentativa: R$ ' || :NEW.preco_venda;
                  
    -- 1. Salva o alerta em transação isolada
    prc_gravar_alerta_fraude(:NEW.id, v_msg_erro);
    
    -- 2. Aborta a operação principal
    RAISE_APPLICATION_ERROR(-20001, 'BLOQUEADO PELA FÁBRICA: ' || v_msg_erro);
  END IF;
END;
/
```

---

## 4. A Armadilha Clássica: `ORA-04091: Mutating Table` (Tabela Mutante)
Uma Trigger de linha (`FOR EACH ROW`) associada a uma tabela **não pode executar `SELECT`, `UPDATE` ou `DELETE` na própria tabela** que está sofrendo a ação no mesmo instante, pois os dados estão em transição ("em mutação").
- **Regra Prática:** Dentro de uma Trigger `FOR EACH ROW` na tabela `produtos`, consulte ou altere outras tabelas livremente (`auditoria`, `estoque`), mas acesse os dados da linha atual de `produtos` exclusivamente pelas variáveis `:OLD` e `:NEW`.

---

## 5. Alta Performance em Memória: Collections e `BULK COLLECT`
Em vez de buscar registros linha a linha com um Cursor tradicional (gerando múltiplas trocas de contexto entre os motores PL/SQL e SQL), o `BULK COLLECT INTO` transfere todo o conjunto de resultados de uma única vez diretamente para um array (Collection) na memória RAM.

```sql
DECLARE
  TYPE t_lista_moveis IS TABLE OF produtos%ROWTYPE;
  v_moveis_memoria t_lista_moveis;
BEGIN
  -- Carrega todas as linhas de uma só vez para a RAM
  SELECT * 
    BULK COLLECT INTO v_moveis_memoria 
    FROM produtos;
    
  DBMS_OUTPUT.PUT_LINE('Total carregado na RAM: ' || v_moveis_memoria.COUNT);
  
  FOR i IN 1 .. v_moveis_memoria.COUNT LOOP
    DBMS_OUTPUT.PUT_LINE('Índice [' || i || ']: ' || v_moveis_memoria(i).descricao || 
                         ' | Lucro Bruto: R$ ' || (v_moveis_memoria(i).preco_venda - v_moveis_memoria(i).preco_custo));
  END LOOP;
END;
/
```

---

## 6. Estudo de Caso Real: Explosão de Engenharia (BOM / Builder) e Árvores Hierárquicas
Na indústria moveleira, um móvel acabado (Nível 1 - Pai Raiz) possui subconjuntos, caixarias, ferragens e insumos até o 5º ou 6º nível. Quando ocorre um erro em um componente de nível profundo durante a montagem (*builder*), varrer a tabela sequencialmente para descobrir qual móvel principal foi afetado gera alto custo de processamento.

### 6.1. Modelagem de Teste (5 Níveis de Profundidade)
```sql
DROP TABLE estrutura_builder CASCADE CONSTRAINTS;

CREATE TABLE estrutura_builder (
    id_componente   NUMBER PRIMARY KEY,
    id_pai          NUMBER REFERENCES estrutura_builder(id_componente),
    nivel_real      NUMBER,
    descricao       VARCHAR2(100),
    status_cadastro VARCHAR2(20) DEFAULT 'OK'
);

INSERT INTO estrutura_builder VALUES (1000, NULL, 1, 'ROUPEIRO CASAL MASTER 6 PORTAS', 'OK');
INSERT INTO estrutura_builder VALUES (2000, 1000, 2, 'MODULO GAVETEIRO INTERNO', 'OK');
INSERT INTO estrutura_builder VALUES (3000, 2000, 3, 'KIT GAVETA MDF BRANCO', 'OK');
INSERT INTO estrutura_builder VALUES (4000, 3000, 4, 'PAR CORREDIÇA TELESCÓPICA 40CM', 'OK');
INSERT INTO estrutura_builder VALUES (5001, 4000, 5, 'PARAFUSO FIXAÇÃO 3.5x14MM (SEM CUSTO!)', 'ERRO_CUSTO');
COMMIT;
```

### 6.2. Abordagem 1: Query Hierárquica Top-Down (`CONNECT_BY_ROOT` e `SYS_CONNECT_BY_PATH`)
Sem precisar de DDL (`ALTER TABLE`), o Oracle identifica o Pai Raiz e monta a trilha completa da falha nativamente:

```sql
SELECT LEVEL AS nivel_calculado,
       id_componente AS id_item_atual,
       descricao AS componente_atual,
       status_cadastro,
       CONNECT_BY_ROOT id_componente AS id_pai_raiz,
       CONNECT_BY_ROOT descricao AS nome_pai_raiz,
       SYS_CONNECT_BY_PATH(descricao, ' -> ') AS caminho_completo_arvore
  FROM estrutura_builder
 WHERE status_cadastro = 'ERRO_CUSTO'
 START WITH id_pai IS NULL
CONNECT BY PRIOR id_componente = id_pai;
```

### 6.3. Abordagem 2: Busca Invertida Bottom-Up (Do Filho com Erro até o Pai Raiz)
Quando o código PL/SQL já possui o ID do componente que falhou no 5º nível (`5001`), subimos pela chave primária dos pais sem ler o restante da tabela:

```sql
DECLARE
  v_id_item_com_erro NUMBER := 5001;
  v_id_pai_raiz      NUMBER;
  v_nome_pai_raiz    VARCHAR2(100);
  v_trilha_erro      VARCHAR2(500);
BEGIN
  SELECT id_componente, 
         descricao,
         SYS_CONNECT_BY_PATH(descricao, ' <- ')
    INTO v_id_pai_raiz, 
         v_nome_pai_raiz,
         v_trilha_erro
    FROM estrutura_builder
   WHERE id_pai IS NULL
   START WITH id_componente = v_id_item_com_erro
  CONNECT BY PRIOR id_pai = id_componente;

  DBMS_OUTPUT.PUT_LINE('🚨 ALERTA DO BUILDER:');
  DBMS_OUTPUT.PUT_LINE('Falha no componente ID: ' || v_id_item_com_erro);
  DBMS_OUTPUT.PUT_LINE('Móvel Principal Afetado (Pai Raiz): [' || v_id_pai_raiz || '] ' || v_nome_pai_raiz);
  DBMS_OUTPUT.PUT_LINE('Rastro da Falha: ' || v_trilha_erro);
END;
/
```

### 6.4. Arquitetura Corporativa Recomendada
1. **Curto Prazo (Zero DDL / Apenas Leitura):** Consultas hierárquicas invertidas (`CONNECT BY PRIOR id_pai = id_componente`) para identificar o Pai Raiz instantaneamente.
2. **Médio Prazo (Evolução de Banco / DBA):** Persistir o `id_pai_raiz` na tabela de explosão do pedido (transformando busca $O(N)$ em $O(1)$) e gravar falhas com `PRAGMA AUTONOMOUS_TRANSACTION`.
3. **Monitoramento e Automação:** Conectar um workflow no **n8n** lendo a tabela de logs de erro para disparar alertas com a trilha completa da engenharia em tempo real.
