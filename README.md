# Curso Prático de Oracle PL/SQL Corporativo

Este repositório contém o material de apoio, planos de aula, fluxos de integração e exercícios para o treinamento prático focado em desenvolvimento Oracle PL/SQL. O objetivo é capacitar profissionais de TI para atuar em cenários reais de mercado (integrações, validações de negócio e automação de rotinas).

## 📌 Estrutura e Controle de Carga Horária
- **Carga Horária Total Contratada:** 28 horas
- **Formato Atual:** Encontros práticos de 3 horas *(Aula inaugural de 4 horas)*
- **Público-alvo:** Profissionais de TI com conhecimento prévio em lógica de programação e SQL básico.
- **Metodologia:** Foco 100% em aplicação prática, usando analogias de fácil compreensão e resolução de problemas corporativos (cenário indústria moveleira).

### ⏱️ Progresso das Aulas (Horas Computadas)
| Encontro | Tópicos Principais | Duração | Status |
| :--- | :--- | :---: | :---: |
| **Aula 1** | Fundamentos, Blocos Anônimos, Variáveis, `%TYPE`, `IF/ELSIF`, `SELECT INTO` | 4h | ✅ Concluída |
| **Aula 2** | Exceções (`NO_DATA_FOUND`, `TOO_MANY_ROWS`), Cursores, SQL Dinâmico, `FOR UPDATE` | 3h | ✅ Concluída |
| **Aula 3** | Procedures (`IN`/`OUT`), Primary Key, Functions (`SELECT`/`UPDATE`), Múltiplos Retornos | 3h | ✅ Concluída |
| **Aula 4** | Modelagem Relacional, Packages (`SPEC`/`BODY`), Triggers (`BEFORE`/`AFTER`, `:OLD`/`:NEW`), `RAISE_APPLICATION_ERROR` | 3h | ✅ Concluída |
| **Aula 5** | Triggers Multi-Eventos, `PRAGMA AUTONOMOUS_TRANSACTION`, `ORA-04091`, `BULK COLLECT` e Árvores Hierárquicas (`CONNECT BY`) | 3h | ✅ Concluída |
| **Aula 6** | Gravação em Lote (`FORALL` + `SAVE EXCEPTIONS`), `COMPOUND TRIGGER`, JSON Nativo (`JSON_OBJECT`) e Integração de Alertas com n8n | 3h | ✅ Concluída |
| **Aula 7** | Integração Corporativa: `JSON_TABLE` (`NESTED PATH`), Upsert com `MERGE INTO`, `SYS_REFCURSOR` e Formulário Web n8n ➔ Oracle | 3h | ✅ Concluída |
| **Total Realizado** | **22 horas concluídas** *(Restam 6 horas para atingir o teto de 28h)* | **22h / 28h** | 🔄 Em andamento |

*(Planejamento para as 6h restantes: 2 encontros finais de 3h — fechamento de tópicos avançados + Projeto Prático Integrado)*

## 🛠️ Setup do Ambiente
1. **[FreeSQL](https://freesql.com/):** Ambiente web limpo para execução rápida.
2. **Docker / WSL (Oracle 23c Free):** Padrão corporativo.
   ```bash
   docker run -d --name oracle-db -p 1521:1521 -e ORACLE_PASSWORD=SuaSenha gvenzl/oracle-free
   ```

## 📂 Organização do Repositório
- **Aula 1:** Fundamentos e `SELECT INTO`
  - `/aulas/aula1.md` | `/exercicios/aula1_exercicios.md`
- **Aula 2:** Exceções, Cursores, SQL Dinâmico e Atualização em Lote (`FOR UPDATE`)
  - `/aulas/aula2.md` | `/aulas/aula2_update.md`
  - `/exercicios/aula2_exercicios.md` | `/exercicios/aula2_update_exercicios.md`
- **Aula 3:** Procedures, Integridade com Primary Key e Functions Avançadas
  - `/aulas/aula3.md` | `/exercicios/aula3_exercicios.md`
- **Aula 4:** Packages, Triggers de Auditoria e Blindagem de Regras de Negócio (Cenário Moveleiro)
  - `/aulas/aula4.md` | `/exercicios/aula4_exercicios.md`
- **Aula 5:** Triggers Avançadas, Transações Autônomas, Alta Performance (`BULK COLLECT`) e Explosão de Engenharia (BOM)
  - `/aulas/aula5.md` | `/exercicios/aula5_exercicios.md`
- **Aula 6:** Processamento em Massa (`FORALL`), Compound Triggers, Geração de JSON Nativo e Automação com n8n
  - `/aulas/aula6.md` | `/exercicios/aula6_exercicios.md`
- **Aula 7:** Ingestão de JSON (`JSON_TABLE`), Upsert (`MERGE INTO`), Listas Dinâmicas (`SYS_REFCURSOR`) e Fluxo Formulário n8n ➔ Oracle
  - `/aulas/aula7.md` | `/aulas/aula7_workflow_n8n.json` | `/exercicios/aula7_exercicios.md`
