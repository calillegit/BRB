# BRB — Squad de Dados de RH

Repositório de consultas, planilhas, dashboards e automações de **dados de RH** do BRB, sobre a base Oracle do sistema de RH (schema `RHPRODUCAO`).

## Squad

| Agente | Arquivo | Papel |
|---|---|---|
| `especialista-dados-rh` | `.claude/agents/especialista-dados-rh.md` | Especialista em dados de RH: Oracle SQL, Excel, Power BI e automações |

Invocação: peça ao Claude "use o especialista-dados-rh para …" ou mencione `@especialista-dados-rh`.

## Estrutura do repositório

```
queries/        Consultas Oracle SQL (uma por assunto, nome em snake_case)
docs/           Dicionário de dados, regras de negócio, decisões
.claude/agents/ Definição dos agentes do squad
```

Sugestão de crescimento: `excel/` (modelos e macros), `powerbi/` (medidas DAX, notas de modelo), `automacoes/` (scripts Python/PowerShell/VBA).

## Contexto do domínio (extraído de `queries/quadro_pessoal_substituicao.sql`)

- Schema: `RHPRODUCAO`. Chaves técnicas são `HANDLE` (junção sempre por `HANDLE`, nunca por nome).
- Funcionários: `DO_FUNCIONARIOS` (`MATRICULA`, `SITUACAO`, `CARGO`, `DEMISSAODATA`).
- Histórico de função: `DO_FUNCIONARIOFUNCOES` (`TIPOFUNCAO` 1 = função atual, 2 = substituição; `INICIO`/`FIM`).
- Histórico de lotação: `DO_FUNCIONARIOHIERARQUIAS`; cessões/adições: `DO_FUNCIONARIOCESSOES` (`CESSAOINICIO`/`CESSAOFIM`, `HIERARQUIAINTERNA`).
- Estrutura organizacional: `ADM_HIERARQUIAS` (`APELIDO`, `ESTRUTURA`, `ATIVA`) + `K_BRB_HIERARQUIAS` (`EMPRESA`, `DIRETORIA`, `SUPERINTENDENCIA`, `GERENCIA`).
- Quadro de pessoal: `QP_QUADROPESSOAL` (`PLANO` = quadro aprovado, `TOTALQUADRO` = quadro efetivo, `VAGAS`), `QP_QUADROCARGOS`, `QP_QUADROHIERARQUIAS`.
- Cargos/funções: `CS_UNIFICACARGOSFUNCOES`, `CS_CARGOS`, `CS_FUNCOES`, `CS_CLASSES`, carreiras/trilhas `K9_CS_CARREIRAS`/`K9_CS_TRILHAS`.
- Filtros recorrentes: `K9_ATIVO = 'S'`, `ATIVA = 'S'`, `ESTRUTURA LIKE '2020%'`, `SITUACAO NOT IN (1,5)`.
- Classificação local: apelido começando com `AGÊ` ou `PA` = **Rede**; demais = **Direção Geral**.

> Os significados dos códigos acima foram inferidos da query existente. Confirme com o time de RH antes de tratá-los como regra e registre em `docs/`.

## Convenções de Oracle SQL

- Somente leitura: **nunca** gerar `INSERT/UPDATE/DELETE/DDL` sem pedido explícito.
- Usar CTEs (`WITH`) nomeadas por propósito; um bloco por regra de negócio.
- Datas: `TO_DATE('31/12/2025','DD/MM/YYYY')` com máscara explícita; evitar depender de `NLS_DATE_FORMAT`.
- Períodos em aberto: `NVL(FIM, <data_referência>)`. Sobreposição de períodos: `INICIO <= :fim AND NVL(FIM, :fim) >= :inicio`.
- Parametrizar datas/filtros com bind variables (`:dt_ref`) em vez de literais espalhados.
- Evitar `SELECT *` na saída final; nomear colunas em português, em maiúsculas, sem acento nos aliases técnicos.
- Evitar `ORDER BY` dentro de subqueries/CTEs sem necessidade; filtrar cedo, juntar tarde.
- Conferir a granularidade antes de agregar (duplicidade por `LEFT JOIN` é o erro mais comum).
- Validar: contagem total, soma de controle e amostra de 3–5 matrículas conhecidas.

## Excel, Power BI e automações

- **Excel**: dados em tabelas estruturadas, fórmulas legíveis (preferir `XLOOKUP`/`SUMIFS` a fórmulas aninhadas), sem valores fixos no meio de fórmulas; abas separadas para Dados, Parâmetros e Resultado.
- **Power BI**: modelo estrela (fatos × dimensões), medidas DAX em tabela própria `_Medidas`, tabela Calendário dedicada, relacionamentos 1:N de filtro único, nomes de campos em português de negócio.
- **Automações**: scripts idempotentes, credenciais **fora do código** (variáveis de ambiente / cofre), logs de execução, tratamento de erro e saída reprodutível.

## Privacidade e LGPD (obrigatório)

Dados de RH são dados pessoais, parte deles sensíveis (salário, saúde, afastamentos).

- Nunca commitar extrações reais, planilhas com dados de pessoas, credenciais ou strings de conexão.
- Usar dados anonimizados/sintéticos em exemplos e testes.
- Mostrar o mínimo necessário (matrícula em vez de nome quando possível; agregar quando houver risco de reidentificação em grupos pequenos).
- Em dashboards, aplicar RLS (segurança em nível de linha) quando o público for diferente do dono dos dados.
- `.gitignore` já deve cobrir `*.xlsx`, `*.csv`, `*.pbix` com dados reais.

## Fluxo de trabalho

1. Entender a pergunta de negócio e a definição de cada indicador (headcount, vagas, rotatividade, etc.).
2. Escrever/ajustar a query em `queries/`, comentando as regras no cabeçalho do arquivo.
3. Validar números contra uma fonte conhecida.
4. Entregar no formato pedido (Excel, Power BI, automação) documentando premissas.
5. Commit pequeno e descritivo, em português.

## Pendências conhecidas em `queries/quadro_pessoal_substituicao.sql`

Ao retomar esta query, revisar: `WEHRE` (typo, alias de subquery), `SELECT D.` incompleto no `MAX (D.`, `ROW_NUMBER() OVER (...)` sem `FROM` em subconsulta `H`, `SELECT * FROM(` externo, datas hardcoded (`31/07/2026`, calendário a partir de 01/01/1960) e `ORDER BY` dentro de subqueries.
