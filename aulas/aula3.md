# Aula 3: Blocos Nomeados (Procedures e Functions) e Integridade de Dados

## 1. Procedures com Parâmetros de Entrada (IN)
Diferente dos blocos anônimos (que evaporam após a execução), uma `PROCEDURE` é compilada e armazenada permanentemente no dicionário de dados do Oracle, pronta para ser chamada por sistemas externos (APIs Java, Node.js, n8n, etc.).

**Analogia:** Um bloco anônimo é uma receita anotada num guardanapo que vai para o lixo. Uma `PROCEDURE` é uma receita gravada no livro oficial do restaurante: qualquer cozinheiro pode executá-la chamando apenas pelo nome e passando os ingredientes (parâmetros).

```sql
CREATE OR REPLACE PROCEDURE prc_aplicar_desconto (
  p_id_produto    IN NUMBER, -- Parâmetro de entrada (recebe valor)
  p_perc_desconto IN NUMBER
) IS
BEGIN
  UPDATE produtos 
     SET preco = preco - (preco * (p_perc_desconto / 100))
   WHERE id = p_id_produto;
   
  COMMIT;
  DBMS_OUTPUT.PUT_LINE('Desconto de ' || p_perc_desconto || '% aplicado no produto ' || p_id_produto);
END prc_aplicar_desconto;
/

-- Chamada da Procedure (Passagem de Parâmetro Nominal com '=>')
BEGIN
  prc_aplicar_desconto(p_id_produto => 1, p_perc_desconto => 15);
END;
/
```

---

## 2. Procedures com Parâmetros de Saída (OUT)
Usamos o parâmetro `OUT` quando a procedure precisa processar algo e devolver uma resposta direta para a aplicação que a chamou (como um valor calculado, um ID gerado ou uma mensagem de status).

**Analogia:** O parâmetro `IN` é você fazendo o pedido no balcão da lanchonete. O parâmetro `OUT` é a senha impressa que o atendente te devolve para você acompanhar o resultado.

```sql
CREATE OR REPLACE PROCEDURE prc_consultar_preco (
  p_id_produto IN  NUMBER, -- Entrada fornecida pelo usuário/sistema
  p_preco_item OUT NUMBER  -- Saída preenchida e devolvida pela procedure
) IS
BEGIN
  SELECT preco 
    INTO p_preco_item 
    FROM produtos 
   WHERE id = p_id_produto;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    p_preco_item := 0;
  WHEN TOO_MANY_ROWS THEN
    p_preco_item := 0;
    DBMS_OUTPUT.PUT_LINE('ALERTA DE DADOS: O ID ' || p_id_produto || ' está duplicado na base!');
END prc_consultar_preco;
/

-- Chamando e capturando o retorno em uma variável local
DECLARE
  v_preco_retorno NUMBER; 
BEGIN
  prc_consultar_preco(p_id_produto => 10, p_preco_item => v_preco_retorno);
  DBMS_OUTPUT.PUT_LINE('O preço retornado foi: ' || v_preco_retorno);
END;
/
```

---

## 3. Blindagem de Dados: Evitando `ORA-01422` com Primary Key
Quando um `SELECT INTO` dentro de uma procedure encontra mais de um registro para o mesmo ID, o Oracle dispara o erro `ORA-01422: exact fetch returned more than requested number of rows` (`TOO_MANY_ROWS`). Para impedir fisicamente que IDs duplicados entrem na tabela, aplicamos uma **Chave Primária (Primary Key)**.

```sql
-- 1. Limpeza de eventuais registros duplicados
DELETE FROM produtos WHERE id = 10;
INSERT INTO produtos (id, descricao, preco) VALUES (10, 'Cadeira Gamer', 1200);

-- 2. Aplicação da restrição de Chave Primária
ALTER TABLE produtos ADD CONSTRAINT pk_produtos PRIMARY KEY (id);
COMMIT;
```
*Nota técnica:* A partir deste ponto, qualquer tentativa de inserir um ID já existente será bloqueada imediatamente pelo motor do banco com o erro `ORA-00001: unique constraint violated`.

---

## 4. Functions (Funções no SQL e PL/SQL)
As `FUNCTIONS` têm a obrigação de retornar um valor através da cláusula `RETURN`. O grande diferencial corporativo das Functions é que elas podem ser executadas diretamente dentro de comandos SQL (`SELECT`, `UPDATE`, `WHERE`).

**Analogia:** Se a Procedure é um empreiteiro que executa uma obra inteira, a Function é uma fórmula do Excel (`=SOMA()`) que você aplica direto na célula para calcular e exibir um valor instantaneamente.

```sql
CREATE OR REPLACE FUNCTION fnc_calcular_imposto (
  p_preco IN NUMBER
) RETURN NUMBER 
IS
  v_imposto NUMBER(10,2);
BEGIN
  v_imposto := p_preco * 0.18; -- 18% de imposto
  RETURN v_imposto; 
END fnc_calcular_imposto;
/
```

### 4.1. Utilizando a Function direto no `SELECT`
```sql
SELECT id, 
       descricao, 
       preco AS preco_bruto,
       fnc_calcular_imposto(preco) AS valor_imposto,
       (preco + fnc_calcular_imposto(preco)) AS preco_final_cliente
  FROM produtos;
```

### 4.2. Utilizando a Function direto no `UPDATE` (Set-Based Processing)
Em vez de abrir um cursor para atualizar linha por linha, podemos aplicar a Function diretamente no comando `UPDATE` para atualizar milhares de registros de uma só vez.
*Cuidado de Regra de Negócio:* Como a função retorna apenas o valor do imposto (18%), devemos somá-la ao preço original (`preco + fnc...`) para não sobrescrever o valor do produto apenas pelo valor do imposto.

```sql
UPDATE produtos
   SET preco = preco + fnc_calcular_imposto(preco);
COMMIT;
```

---

## 5. Uma Function pode ter dois retornos?
Por regra de arquitetura, a cláusula `RETURN` devolve apenas **uma única estrutura**. Existem três formas de lidar com múltiplos valores:

1. **Texto Formatado ou JSON (Para uso em `SELECT`):**
   ```sql
   CREATE OR REPLACE FUNCTION fnc_moedas (p_preco IN NUMBER) 
   RETURN VARCHAR2 IS
   BEGIN
     RETURN 'USD: ' || ROUND(p_preco / 5, 2) || ' | EUR: ' || ROUND(p_preco / 6, 2);
   END;
   /
   ```
2. **Registro Completo com `%ROWTYPE` (Apenas para blocos PL/SQL):**
   ```sql
   CREATE OR REPLACE FUNCTION fnc_dados_produto (p_id IN NUMBER) 
   RETURN produtos%ROWTYPE IS
     v_linha produtos%ROWTYPE;
   BEGIN
     SELECT * INTO v_linha FROM produtos WHERE id = p_id;
     RETURN v_linha; -- Empacota e retorna todas as colunas juntas
   END;
   /
   ```
3. **Regra de Ouro Corporativa:** Nunca utilize parâmetros `OUT` dentro de uma `FUNCTION` (isso impede que ela seja usada dentro de um `SELECT`). Se precisar retornar múltiplos valores separados para uma aplicação, utilize uma **Procedure com múltiplos parâmetros `OUT`**.
