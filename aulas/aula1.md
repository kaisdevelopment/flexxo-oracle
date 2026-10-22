# Aula 1: Fundamentos do PL/SQL

## 1. O Motor PL/SQL e o "Hello World"
Diferente de linguagens de aplicação (como Java ou React) que trafegam dados pela rede, o PL/SQL processa tudo **dentro do próprio servidor de banco de dados**. Isso elimina gargalos de rede.

**Analogia:** Fazer consultas simples (SQL) é como pedir um prato por vez ao garçom. PL/SQL é como enviar a receita para o chef na cozinha e receber o banquete pronto de uma vez.

```sql
SET SERVEROUTPUT ON; 

DECLARE
  v_ambiente VARCHAR2(50) := 'FreeSQL';
BEGIN
  DBMS_OUTPUT.PUT_LINE('Sucesso! Rodando via: ' || v_ambiente);
END;
```

## 2. Variáveis, Tipos e a "Mágica" do %TYPE
No mercado, evitamos fixar o tamanho de variáveis. Usamos a ancoragem (`%TYPE`), que faz a variável herdar o tipo e o tamanho exato de uma coluna do banco.

**Analogia:** `VARCHAR2(50)` é uma calça de tamanho fixo (se engordar, rasga). `%TYPE` é uma calça de elástico (se o DBA aumentar a coluna da tabela, seu código se adapta automaticamente).

```sql
-- Exemplo de criação de tabela base
CREATE TABLE funcionarios (id NUMBER, nome VARCHAR2(100), cargo VARCHAR2(50));

DECLARE
  v_nome    VARCHAR2(100) := 'João Silva';
  v_salario NUMBER(10,2)  := 4500.50; 
  v_cargo   funcionarios.cargo%TYPE; 
BEGIN
  v_cargo := 'Desenvolvedor Full Stack';
  
  IF v_salario > 5000 THEN
    DBMS_OUTPUT.PUT_LINE(v_nome || ' é Sênior.');
  ELSIF v_salario BETWEEN 3000 AND 5000 THEN
    DBMS_OUTPUT.PUT_LINE(v_nome || ' é Pleno.');
  ELSE
    DBMS_OUTPUT.PUT_LINE(v_nome || ' é Júnior.');
  END IF;
END;
```

## 3. Conectando SQL e PL/SQL com SELECT INTO
O comando `SELECT INTO` é a ponte entre o banco de dados (SQL) e a lógica (PL/SQL). Ele exige que a consulta retorne **exatamente uma linha** (caminho feliz).

```sql
-- Populando a tabela para o teste
INSERT INTO funcionarios (id, nome, cargo) VALUES (100, 'Ana Costa', 'Arquiteta');
COMMIT; 

DECLARE
  v_nome_func  funcionarios.nome%TYPE;
  v_cargo_func funcionarios.cargo%TYPE;
BEGIN
  SELECT nome, cargo INTO v_nome_func, v_cargo_func 
    FROM funcionarios WHERE id = 100;
   
  DBMS_OUTPUT.PUT_LINE('Funcionário encontrado: ' || v_nome_func);
END;
```
