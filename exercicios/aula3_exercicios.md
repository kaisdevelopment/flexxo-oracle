# Exercícios da Aula 3 - Procedures e Functions

## Exercício 1: Cadastro de Produto via Procedure (`IN`)
**Cenário:** Criar uma rotina padrão no banco para cadastrar novos produtos e testá-la inserindo o item ID 10 ("Cadeira Gamer", R$ 1200,00).

```sql
CREATE OR REPLACE PROCEDURE prc_novo_produto (
  p_id        IN NUMBER,
  p_descricao IN VARCHAR2,
  p_preco     IN NUMBER
) IS
BEGIN
  INSERT INTO produtos (id, descricao, preco) 
  VALUES (p_id, p_descricao, p_preco);
  
  COMMIT;
  DBMS_OUTPUT.PUT_LINE('Produto cadastrado com sucesso: ' || p_descricao);
END prc_novo_produto;
/

-- Bloco de Execução
BEGIN
  prc_novo_produto(
    p_id        => 10, 
    p_descricao => 'Cadeira Gamer', 
    p_preco     => 1200
  );
END;
/
```

---

## Exercício 2: Validador de Alçada de Preço (`IN` e `OUT`)
**Cenário:** Criar uma procedure `prc_validar_estoque` que receba o ID do produto (`IN`) e devolva um status textual (`OUT`). Se o preço for maior que R$ 1.000,00, retornar `'PRODUTO CARO - PEDIR APROVAÇÃO'`; caso contrário, `'PRODUTO BARATO - VENDA LIVRE'`.

```sql
CREATE OR REPLACE PROCEDURE prc_validar_estoque (
  p_id_produto IN  NUMBER,
  p_status     OUT VARCHAR2
) IS
  v_preco produtos.preco%TYPE;
BEGIN
  SELECT preco 
    INTO v_preco 
    FROM produtos 
   WHERE id = p_id_produto;
   
  IF v_preco > 1000 THEN
    p_status := 'PRODUTO CARO - PEDIR APROVAÇÃO';
  ELSE
    p_status := 'PRODUTO BARATO - VENDA LIVRE';
  END IF;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    p_status := 'ERRO: PRODUTO NÃO ENCONTRADO';
END prc_validar_estoque;
/

-- Bloco de Teste
DECLARE
  v_resultado VARCHAR2(100);
BEGIN
  prc_validar_estoque(p_id_produto => 10, p_status => v_resultado);
  DBMS_OUTPUT.PUT_LINE('Status da Venda: ' || v_resultado);
END;
/
```

---

## Exercício 3: Tag de Frete no E-commerce (`FUNCTION` no `SELECT`)
**Cenário:** Criar uma Function `fnc_regra_frete` que receba o preço do produto e retorne `'FRETE GRÁTIS'` para valores acima de R$ 200,00 ou `'FRETE PAGO (R$ 25,00)'` para os demais, exibindo o resultado diretamente em uma consulta SQL.

```sql
CREATE OR REPLACE FUNCTION fnc_regra_frete (
  p_preco IN NUMBER
) RETURN VARCHAR2 
IS
BEGIN
  IF p_preco > 200 THEN
    RETURN 'FRETE GRÁTIS';
  ELSE
    RETURN 'FRETE PAGO (R$ 25,00)';
  END IF;
END fnc_regra_frete;
/

-- Consulta do Relatório
SELECT id,
       descricao, 
       preco, 
       fnc_regra_frete(preco) AS status_frete 
  FROM produtos;
```
