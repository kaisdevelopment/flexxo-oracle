# Curso Prático de Oracle PL/SQL Corporativo

Este repositório contém o material de apoio, planos de aula e exercícios para o treinamento prático focado em desenvolvimento Oracle PL/SQL. O objetivo é capacitar profissionais de TI para atuar em cenários reais de mercado (integrações, validações de negócio e automação de rotinas).

## 📌 Estrutura do Curso
- **Carga Horária:** 28 horas
- **Formato:** 7 encontros de 4 horas
- **Público-alvo:** Profissionais de TI com conhecimento prévio em lógica de programação e SQL básico.
- **Metodologia:** Foco 100% em aplicação prática, usando analogias de fácil compreensão e resolução de problemas corporativos.

## 🛠️ Setup do Ambiente
Os códigos deste repositório podem ser executados nas seguintes plataformas:
1. **[FreeSQL](https://freesql.com/):** Ambiente web limpo para execução rápida sem necessidade de infraestrutura.
2. **Docker / WSL (Oracle 23c Free):** Padrão corporativo sugerido para simulação real de ambiente.
   ```bash
   docker run -d --name oracle-db -p 1521:1521 -e ORACLE_PASSWORD=SuaSenha gvenzl/oracle-free
   ```

## 📂 Organização do Repositório
- **Aula 1:** Fundamentos, Variáveis, %TYPE, SELECT INTO.
  - `/aulas/aula1.md`
  - `/exercicios/aula1_exercicios.md`
- **Aula 2:** Processamento em Lote com Cursores (Explícitos, FOR LOOP, SQL Dinâmico, FOR UPDATE).
  - `/aulas/aula2.md`
  - `/aulas/aula2_update.md`
  - `/exercicios/aula2_exercicios.md`
  - `/exercicios/aula2_update_exercicios.md`
