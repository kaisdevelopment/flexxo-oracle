# Aula 2: Exceções, Cursores e Processamento em Lote

## 1. Tratamento de Exceções (O Airbag do Código)
O comando `SELECT INTO` exige exatamente uma linha. Se não encontrar nada, dispara `NO_DATA_FOUND`. Se encontrar mais de uma, dispara `TOO_MANY_ROWS`. O bloco `EXCEPTION` captura esses erros para que o sistema não "quebre" em produção.

**Analogia:** É como pedir a um arquivista a pasta do "João". Se ele não acha a pasta (NO_DATA_FOUND) ou acha 50 pastas (TOO_MANY_ROWS), as exceções ensinam o assistente a relatar o problema de forma limpa em vez de travar o sistema.

```sql
DECLARE
  v_nome_func  VARCHAR2(100);
BEGIN
  -- Forçando erro com filtro genérico que traz várias linhas
  SELECT nome INTO v_nome_func FROM funcionarios;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('ERRO: Nenhum registro encontrado.');
  WHEN TOO_MANY_ROWS THEN
    DBMS_OUTPUT.PUT_LINE('ERRO: A busca retornou mais de uma linha!');
END;
```

## 2. Introdução aos Cursores (Lidando com Múltiplas Linhas)
Para processar múltiplos registros em lote, usamos o Cursor. Ele guarda o resultado da consulta na memória e permite que o PL/SQL processe uma linha de cada vez.

**Cursor Explícito (Clássico):** O desenvolvedor tem controle total: declara, abre, busca linha a linha e fecha.
```sql
DECLARE
  CURSOR c_produtos IS SELECT descricao, preco FROM produtos;
  v_desc  VARCHAR2(50);
  v_preco NUMBER;
BEGIN
  OPEN c_produtos;
  LOOP
    FETCH c_produtos INTO v_desc, v_preco;
    EXIT WHEN c_produtos%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE('Produto: ' || v_desc || ' - R$ ' || v_preco);
  END LOOP;
  CLOSE c_produtos;
END;
```

## 3. Cursor FOR LOOP (O Padrão de Mercado)
O banco de dados gerencia a abertura, cria as variáveis de controle (um *record* implícito), faz a busca e fecha o cursor sozinho. É mais rápido e limpo.
```sql
BEGIN
  FOR r_prod IN (SELECT descricao, preco FROM produtos) LOOP
    DBMS_OUTPUT.PUT_LINE('FOR LOOP Produto: ' || r_prod.descricao || ' - R$ ' || r_prod.preco);
  END LOOP;
END;
```

## 4. Cursor + SQL Dinâmico (EXECUTE IMMEDIATE)
Usamos cursores para ler parâmetros e montar queries em tempo de execução. O uso do `USING` (Bind Variables) previne SQL Injection e aumenta a performance.
```sql
DECLARE
    v_dolar  NUMBER := 5.37;
    v_euro   NUMBER := 6.01;
    v_sql    VARCHAR2(200);
    v_precod NUMBER(10,3);
    v_precoe NUMBER(10,3);  
BEGIN
    FOR r_prod IN (SELECT descricao, preco FROM produtos) LOOP
        -- Query dinâmica com múltiplos retornos
        v_sql := 'SELECT :val_usd / :cot_usd, :val_eur / :cot_eur FROM DUAL';

        EXECUTE IMMEDIATE v_sql
                     INTO v_precod, v_precoe
                    USING r_prod.preco, v_dolar, r_prod.preco, v_euro;

        DBMS_OUTPUT.PUT_LINE(r_prod.descricao || ': R$ ' || r_prod.preco || ' = € ' || v_precoe || ' = US$ ' || v_precod);    
    END LOOP;
END;
```

## 5. FOR UPDATE e WHERE CURRENT OF (Atualização Segura)
Para evitar concorrência (outro sistema alterar o dado enquanto lemos), usamos `FOR UPDATE` para travar (lock) a linha. O `WHERE CURRENT OF` garante a atualização exatamente no ponteiro de memória atual.
```sql
DECLARE
  CURSOR c_produtos IS SELECT id, preco FROM produtos FOR UPDATE;
BEGIN
  FOR r_prod IN c_produtos LOOP
    IF r_prod.preco < 150 THEN
      UPDATE produtos SET preco = preco * 1.20 WHERE CURRENT OF c_produtos;
    END IF;
  END LOOP;
  COMMIT; 
END;
```
