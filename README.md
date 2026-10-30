# Curso Prático de Oracle PL/SQL Corporativo

Este repositório contém o material de apoio, planos de aula e exercícios para o treinamento prático focado em desenvolvimento Oracle PL/SQL. O objetivo é capacitar profissionais de TI para atuar em cenários reais de mercado (integrações, validações de negócio e automação de rotinas).

## 📌 Estrutura e Controle de Carga Horária
- **Carga Horária Total Contratada:** 28 horas
- **Formato Atual:** Encontros práticos de 3 horas
- **Público-alvo:** Profissionais de TI com conhecimento prévio em lógica de programação e SQL básico.
- **Metodologia:** Foco 100% em aplicação prática, usando analogias de fácil compreensão e resolução de problemas corporativos.

### ⏱️ Progresso das Aulas (Horas Computadas)
| Encontro | Tópicos Principais | Duração | Status |
| :--- | :--- | :---: | :---: |
| **Aula 1** | Fundamentos, Blocos Anônimos, Variáveis, `%TYPE`, `IF/ELSIF`, `SELECT INTO` | 3h | ✅ Concluída |
| **Aula 2** | Exceções (`NO_DATA_FOUND`, `TOO_MANY_ROWS`), Cursores, SQL Dinâmico, `FOR UPDATE` | 3h | ✅ Concluída |
| **Aula 3** | Procedures (`IN`/`OUT`), Primary Key, Functions (`SELECT`/`UPDATE`), Múltiplos Retornos | 3h | ✅ Concluída |
| **Total Realizado** | **9 horas concluídas** *(Restam 19 horas para atingir o teto de 28h)* | **9h / 28h** | 🔄 Em andamento |

*(Sugestão para as 19h restantes: 5 encontros de 3h + 1 encontro final de 4h para Projeto Prático)*

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
