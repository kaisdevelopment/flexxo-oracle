# Exercícios da Aula 1 - Resoluções e Explicações

## Exercício 1: Regra de Aprovação de Crédito (IF/THEN/ELSIF)
**Cenário:** Criar uma regra de aprovação de crédito validando a variável limite. Maior que 10.000 (VIP), entre 5.000 e 10.000 (GOLD), senão (STANDARD).

### Solução:
```sql
DECLARE
  v_limite NUMBER := 8500;
  v_status VARCHAR2(20);
BEGIN
  IF v_limite > 10000 THEN
    v_status := 'CLIENTE VIP';
  ELSIF v_limite BETWEEN 5000 AND 10000 THEN
    v_status := 'CLIENTE GOLD';
  ELSE
    v_status := 'CLIENTE STANDARD';
  END IF;
  
  DBMS_OUTPUT.PUT_LINE('Status do Cliente: ' || v_status);
END;
```
**Por que funciona assim:** No PL/SQL, a cláusula `ELSE IF` é escrita como `ELSIF` (sem a letra "E" extra e junto). A lógica flui de cima para baixo; assim que a primeira condição verdadeira é encontrada, o bloco a executa e sai do condicional.

---

## Exercício 2: Extraindo dados com SELECT INTO (Caminho Feliz)
**Cenário:** Criar a tabela `produtos`, inserir um registro e buscar esse registro para dentro de variáveis usando ancoragem (`%TYPE`).

### Preparação do Ambiente:
```sql
CREATE TABLE produtos (id NUMBER, descricao VARCHAR2(50), preco NUMBER);
INSERT INTO produtos VALUES (5, 'Teclado Mecânico', 350.00);
COMMIT;
```

### Solução PL/SQL:
```sql
DECLARE
  v_desc  produtos.descricao%TYPE;
  v_preco produtos.preco%TYPE;
BEGIN
  SELECT descricao, preco 
    INTO v_desc, v_preco 
    FROM produtos 
   WHERE id = 5;
   
  DBMS_OUTPUT.PUT_LINE('O produto ' || v_desc || ' custa R$ ' || v_preco);
END;
```
**Por que funciona assim:** O `SELECT INTO` faz a leitura dos campos do banco diretamente para a memória da execução (variáveis). O script teve sucesso porque o filtro `WHERE id = 5` garantiu que o banco retornasse **exatamente uma única linha**, que é a regra de ouro para o comando `SELECT INTO` não gerar exceções.
