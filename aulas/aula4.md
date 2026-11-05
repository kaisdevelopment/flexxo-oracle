# Aula 4: Packages e Triggers (Cenário Indústria Moveleira)

## 1. Preparação do Ambiente Relacional (Fábrica de Móveis)
Criamos uma estrutura com chaves estrangeiras (`FOREIGN KEY`), controle de custo vs. venda e uma tabela dedicada à auditoria automática.

```sql
DROP TABLE auditoria_precos CASCADE CONSTRAINTS;
DROP TABLE produtos CASCADE CONSTRAINTS;
DROP TABLE linhas_moveis CASCADE CONSTRAINTS;

CREATE TABLE linhas_moveis (
    id NUMBER PRIMARY KEY,
    nome VARCHAR2(50) NOT NULL
);

CREATE TABLE produtos (
    id NUMBER PRIMARY KEY,
    id_linha NUMBER,
    descricao VARCHAR2(100) NOT NULL,
    preco_custo NUMBER(10,2) NOT NULL,
    preco_venda NUMBER(10,2) NOT NULL,
    estoque NUMBER DEFAULT 0,
    CONSTRAINT fk_prod_linha FOREIGN KEY (id_linha) REFERENCES linhas_moveis(id)
);

CREATE TABLE auditoria_precos (
    id_log NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_produto NUMBER,
    preco_antigo NUMBER(10,2),
    preco_novo NUMBER(10,2),
    data_hora DATE DEFAULT SYSDATE,
    usuario VARCHAR2(50) DEFAULT USER
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

## 2. Packages (Organizando os Setores da Fábrica)
Um `PACKAGE` agrupa Procedures e Functions correlatas em um único objeto no banco de dados, dividido em duas partes:
- **Specification (Spec):** É o "Catálogo da Loja". Declara publicamente quais rotinas existem e seus parâmetros.
- **Body (Corpo):** É o "Chão de Fábrica (Marcenaria)". Contém o código PL/SQL real e as regras de negócio encapsuladas.

### 2.1. Criando a Specification (Catálogo)
```sql
CREATE OR REPLACE PACKAGE pkg_expedicao AS
  FUNCTION fnc_tem_estoque (p_id_produto IN NUMBER, p_qtd_pedido IN NUMBER) RETURN BOOLEAN;
  PROCEDURE prc_baixar_estoque (p_id_produto IN NUMBER, p_qtd_pedido IN NUMBER);
END pkg_expedicao;
/
```

### 2.2. Criando o Body (Chão de Fábrica) com Validação Detalhada
*Regra importante:* O comando `DBMS_OUTPUT.PUT_LINE` não executa queries (`SELECT`) diretamente dentro dos parênteses. Sempre buscamos os dados antes usando `SELECT INTO` para variáveis locais e passamos as variáveis prontas para exibição.

```sql
CREATE OR REPLACE PACKAGE BODY pkg_expedicao AS

  FUNCTION fnc_tem_estoque (p_id_produto IN NUMBER, p_qtd_pedido IN NUMBER) RETURN BOOLEAN IS
    v_saldo_atual NUMBER;
  BEGIN
    SELECT estoque INTO v_saldo_atual FROM produtos WHERE id = p_id_produto;
    RETURN (v_saldo_atual >= p_qtd_pedido);
  END fnc_tem_estoque;

  PROCEDURE prc_baixar_estoque (p_id_produto IN NUMBER, p_qtd_pedido IN NUMBER) IS
    v_estoque_atual produtos.estoque%TYPE;
    v_nome_movel    produtos.descricao%TYPE;
  BEGIN
    IF fnc_tem_estoque(p_id_produto, p_qtd_pedido) THEN
      UPDATE produtos 
         SET estoque = estoque - p_qtd_pedido 
       WHERE id = p_id_produto;
      COMMIT;
      DBMS_OUTPUT.PUT_LINE('Expedição autorizada! Estoque atualizado.');
    ELSE
      -- Busca os dados na tabela para exibir o saldo real no bloqueio
      SELECT descricao, estoque 
        INTO v_nome_movel, v_estoque_atual
        FROM produtos
       WHERE id = p_id_produto;

      DBMS_OUTPUT.PUT_LINE('BLOQUEADO: Você pediu ' || p_qtd_pedido || 
                           ' un. de [' || v_nome_movel || 
                           '], mas só temos ' || v_estoque_atual || ' no galpão!');
    END IF;
  END prc_baixar_estoque;

END pkg_expedicao;
/
```

### 2.3. Executando Rotinas do Pacote
```sql
BEGIN
  -- Teste 1: Baixa válida (3 unidades de 8 disponíveis)
  pkg_expedicao.prc_baixar_estoque(p_id_produto => 102, p_qtd_pedido => 3);
  
  -- Teste 2: Bloqueio por falta de saldo (pedindo 20 unidades)
  pkg_expedicao.prc_baixar_estoque(p_id_produto => 102, p_qtd_pedido => 20);
END;
/
```

---

## 3. Triggers (Gatilhos Automáticos e Blindagem)
Um `TRIGGER` é disparado automaticamente pelo banco quando ocorre um evento DML (`INSERT`, `UPDATE` ou `DELETE`).
- **`:OLD`**: Representa o valor que estava gravado antes da alteração ou exclusão.
- **`:NEW`**: Representa o valor novo que está sendo inserido ou atualizado.

### 3.1. Trigger de Auditoria Invisível (`AFTER UPDATE`)
Registra automaticamente qualquer alteração no preço de venda na tabela `auditoria_precos`.

```sql
CREATE OR REPLACE TRIGGER trg_auditoria_precos
AFTER UPDATE OF preco_venda ON produtos
FOR EACH ROW
BEGIN
  IF :OLD.preco_venda != :NEW.preco_venda THEN
    INSERT INTO auditoria_precos (id_produto, preco_antigo, preco_novo)
    VALUES (:OLD.id, :OLD.preco_venda, :NEW.preco_venda);
  END IF;
END;
/
```

### 3.2. Trigger "Guarda-Costas de Margem" (`BEFORE UPDATE` + `RAISE_APPLICATION_ERROR`)
Intercepta a gravação antes de ir para o disco (`BEFORE`) e aborta a transação caso tentem cadastrar ou alterar o preço de venda para um valor abaixo do custo de produção.

```sql
CREATE OR REPLACE TRIGGER trg_protege_margem_moveis
BEFORE INSERT OR UPDATE OF preco_venda, preco_custo ON produtos
FOR EACH ROW
BEGIN
  IF :NEW.preco_venda < :NEW.preco_custo THEN
    RAISE_APPLICATION_ERROR(-20001, 
      'OPERAÇÃO BLOQUEADA PELA FÁBRICA: O preço de venda (R$ ' || :NEW.preco_venda || 
      ') do móvel [' || :NEW.descricao || '] não pode ser menor que o custo de produção (R$ ' || :NEW.preco_custo || ')!');
  END IF;
END;
/
```

---

## 4. Dica Técnica: Inspecionando a Estrutura da Tabela (Oracle vs. MySQL)
Diferente do MySQL (que utiliza `SHOW CREATE TABLE nome \G`), no Oracle utilizamos o pacote nativo `DBMS_METADATA` ou comandos do utilitário SQLcl/FreeSQL:

```sql
-- 1. Forma nativa SQL Oracle (Retorna o CREATE TABLE completo)
SELECT DBMS_METADATA.GET_DDL('TABLE', 'AUDITORIA_PRECOS') FROM DUAL;

-- 2. Atalhos de console (FreeSQL / SQL Developer Web / SQLcl)
DDL AUDITORIA_PRECOS;
DESC AUDITORIA_PRECOS;
```
