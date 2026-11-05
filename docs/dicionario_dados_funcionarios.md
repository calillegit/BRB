# Dicionário de dados e guia da query `Funcionarios`

Fonte: [`queries/funcionarios.sql`](../queries/funcionarios.sql) (Oracle, schema `RHPRODUCAO`).
Versão tabular do dicionário (para catálogos/Excel): [`dicionario_dados_funcionarios.csv`](dicionario_dados_funcionarios.csv).

Os tipos de dados são **inferidos** do código (não há DDL). Valide contra o catálogo do banco antes de publicar.

## 1. O que a query entrega

**Granularidade:** uma linha por **colaborador × data de referência (`DIA`)**.
Cada linha é uma foto do colaborador naquele dia: cargo, função, lotação, substituição, benefício educacional, férias/afastamento etc.

**Data de referência.** O calendário é gerado a partir de `DO_FUNCIONARIOFUNCOES` usando `ROWNUM` (truque para criar linhas), agrupado por mês desde 01/01/2018. O filtro final `WHERE DIA = TO_DATE(SYSDATE)` reduz tudo a **uma única data: hoje**.
Para série histórica mensal (um ponto por mês, hoje no mês corrente), troque o filtro pela linha comentada `--) WHERE DIA >= TO_DATE('31/12/2018', 'DD/MM/YYYY')`.

**Quem entra:** `SITUACAO <> 1`, `TIPOCOLABORADOR IN (1,2,3)`, `EMPRESA IN (1,2,3)`, e `DIA` entre admissão e o fim do mês da demissão. Quem saiu em mês anterior não aparece. Quem saiu no mês corrente aparece como `Desligado`.

## 2. Mapa dos blocos (CTEs)

| CTE | Papel | Chave de junção | Fontes principais |
|---|---|---|---|
| `TB_CALENDARIO` | Datas de referência (`DIA`, `P_DIA`) | — | `DO_FUNCIONARIOFUNCOES` (só para gerar linhas) |
| `TB_COLABORADOR` | **Base**: cadastro + cálculos (idade, tempo de casa, geração, escolaridade, salário do cargo) | `HANDLE`, `DIA` | `DO_FUNCIONARIOS`, `CS_CARGOS`, `TA_RACACOR`, `TA_NIVEISESCOLARIDADE`, `K9_CS_CARREIRAS`, `CS_CLASSES`, `DO_FUNCIONARIOFORMACOES`, `CS_CLASSESTEPS` |
| `TB_PRIMEIRA_LOTACAO` | Primeira unidade do colaborador (ignora `ONBRB`) | `FUNCIONARIO` | `DO_FUNCIONARIOCESSOES`, `DO_FUNCIONARIOHIERARQUIAS`, `K_BRB_HIERARQUIAS` |
| `TB_HIERARQUIA` | Unidade formal vigente no dia | `FUNCIONARIO`, `DIA` | `DO_FUNCIONARIOHIERARQUIAS` |
| `TB_ADICAO` | Unidade de "adição" (cessão interna) vigente no dia | `FUNCIONARIO`, `DIA` | `DO_FUNCIONARIOCESSOES` |
| `TB_CESSAO` | Cessão a órgão externo vigente no dia | `FUNCIONARIO`, `DIA` | `K9_DO_FUNCIONARIOCESSOES`, `ADM_OUTRASEMPRESAS` |
| `TB_FUNCAO` | Função efetiva (`TIPOFUNCAO = 1`) vigente no dia | `FUNCIONARIO`, `DIA` | `DO_FUNCIONARIOFUNCOES`, `CS_FUNCOES`, `K9_CS_TRILHAS`, `K9_CS_MOTIVOSALTERACAOFUNCAO` |
| `TB_TEMPO_FUNCAO` | Anos acumulados em funções já encerradas | `FUNCIONARIO`, `DIA` | `DO_FUNCIONARIOFUNCOES` |
| `TB_SUBSTITUICAO` | Substituição (`TIPOFUNCAO = 2`) no dia | `FUNCIONARIO`, `DIA` | `DO_FUNCIONARIOFUNCOES`, `CS_FUNCOES` |
| `TB_BENEFICIO_Q` | Contagem de benefícios educacionais | `FUNCIONARIO`, `DIA` | `BS_REEMBOLSOFUNCIONARIOS`, `BS_REEMBOLSOS` |
| `TB_BENEFICIO_ATUAL` | Detalhe do benefício educacional vigente | `FUNCIONARIO`, `DIA` | `BS_REEMBOLSOFUNCIONARIOS`, `K9_BS_BENEFICIOEDUCACIONAL`, `TA_TERCEIROS`, `TA_FORMACOES` |
| `TB_AFASTAMENTO` | Afastamento vigente no dia | `FUNCIONARIO`, `DIA` | `DO_FUNCIONARIOAFASTAMENTOS`, `FP_MOTIVOSAFASTAMENTOS` |
| `TB_FERIAS_GOZADAS` | Férias em curso no dia | `FUNCIONARIO`, `DIA` | `FP_FUNCIONARIOFERIASGOZADAS` (`SITUACAO='G'`) |
| `TB_FERIAS_PROGRAMADAS` | Próxima férias programada | `FUNCIONARIO`, `DIA` | `FP_FUNCIONARIOFERIASGOZADAS` (`SITUACAO='P'`) |
| `TB_MOTIVO_DEMISSAO` | Causa da demissão no mês | `FUNCIONARIO`, `DIA` | `FP_FUNCIONARIODEMISSAO`, `FP_CAUSASDEMISSAO` |

Padrão geral: todas as CTEs partem de `TB_COLABORADOR`, escolhem **um** registro vigente por colaborador/dia (`ROW_NUMBER ... RANK = 1`, preferindo o de fim mais recente) e voltam ao `SELECT` final por `LEFT JOIN` em `HANDLE + DIA`.

## 3. Conceitos de negócio

- **Hierarquia × Lotação × Primeira lotação:** *hierarquia* é o vínculo formal; *lotação* é onde a pessoa efetivamente está (`NVL(adição, hierarquia)`); *primeira lotação* é a unidade inicial na carreira. Para análise de efetivo por unidade, use `LOTACAO_*`.
- **Cargo × Função × Substituição:** cargo é do concurso/contrato; função é a posição gratificada (`TIPOFUNCAO = 1`); substituição é a função interina (`TIPOFUNCAO = 2`). `FUNCAO_DIA` / `FUNCAO_TIPO_DIA` resolvem a precedência: substituição > função > cargo.
- **Adição × Cessão:** adição é a cessão *interna* (outra unidade do banco); cessão é para *órgão externo*.
- **Colaborador:** 0 Pensionista, 1 Empregados, 2 Diretores/Conselheiros, 3 Estagiários. Pensionistas não recebem hierarquia, função, substituição etc. (as CTEs filtram `COLABORADOR_ID IN (1,2,3)`). Para headcount de empregados, filtre `COLABORADOR_ID = 1`.
- **Regras com listas fixas de matrículas no código** (governança: mudam só editando a query):
  - `CARGOFUNCAO_PROGRAMA = 'Liderança Feminina'` (89 matrículas, função iniciada em 2019+)
  - `CAUSA_DEMISSAO = 'PDVI'` (105 matrículas)
  - Tipos de bolsa em `TB_BENEFICIO_Q` (graduação × pós)
  - Códigos `K9_CODIGO` de funções de liderança (9113, 9114, ... 9160), repetidos em `TB_FUNCAO` e `TB_SUBSTITUICAO`.

## 4. Dicionário de colunas

Saída: 92 colunas. Grupos na ordem em que aparecem no `SELECT`.

### Período / Identificação

| Coluna | Tipo | Origem | Regra / descrição | Valores |
|---|---|---|---|---|
| `DIA` | DATE | TB_CALENDARIO | Data de referência da foto. Com o filtro atual (`DIA = SYSDATE`) é sempre hoje. Sem o filtro, é o último dia de cada mês (ou hoje, no mês corrente). |  |
| `P_DIA` | DATE | TB_CALENDARIO | Primeiro dia do mês de DIA. Janela [P_DIA, DIA] usada nos indicadores 'do mês'. |  |
| `HANDLE` | NUMBER | DO_FUNCIONARIOS.HANDLE | Chave interna do RH. É o `FUNCIONARIO` em todas as tabelas filhas. Usar em joins; não é a matrícula. |  |
| `MATRICULA` | NUMBER | DO_FUNCIONARIOS.MATRICULA | Matrícula sem dígito verificador (o dígito está em K9_MATRICULADIGITO, hoje comentado). |  |
| `NOME` | VARCHAR2 | DO_FUNCIONARIOS.NOME | Nome do colaborador. Dado pessoal (LGPD). |  |

### Colaborador

| Coluna | Tipo | Origem | Regra / descrição | Valores |
|---|---|---|---|---|
| `STATUS` | VARCHAR2 | DEMISSAODATA vs P_DIA/DIA | 'Desligado' se a demissão ocorreu dentro do mês de referência; senão 'Ativo'. Desligados em meses anteriores não aparecem. | Ativo \| Desligado |
| `ADMISSAO_MES` | VARCHAR2 | DATAADMISSAO vs P_DIA/DIA | Admitido dentro do mês de referência. | Sim \| Não |
| `DEMISSAO_MES` | VARCHAR2 | DEMISSAODATA vs P_DIA/DIA | Desligado dentro do mês de referência. | Sim \| Não |
| `NASCIMENTO` | DATE | DO_FUNCIONARIOS.DATANASCIMENTO | Data de nascimento. Dado pessoal. |  |
| `IDADE` | NUMBER | calculado | TRUNC((DIA - NASCIMENTO + 1) / 365). Aproximação (ignora bissextos). | inteiro |
| `SEXO` | VARCHAR2 | DO_FUNCIONARIOS.SEXO | Valor cru do cadastro, sem tradução. |  |
| `RACA_COR` | VARCHAR2 | TA_RACACOR.NOME | Raça/cor autodeclarada. Os handles 1, 4 e 8 estavam agrupados como 'Preta/Parda/Indígena' (linha comentada). |  |
| `COLABORADOR_ID` | NUMBER | DO_FUNCIONARIOS.TIPOCOLABORADOR + CS_CARGOS.TITULO | 0 quando o título do cargo começa com 'PENSIO'; senão TIPOCOLABORADOR. | 0 \| 1 \| 2 \| 3 |
| `COLABORADOR` | VARCHAR2 | idem | Descrição de COLABORADOR_ID. | Pensionista \| Empregados \| Diretores/Conselheiros \| Estagiários |
| `TEMPO_BANCO` | NUMBER | calculado | TRUNC((NVL(DEMISSAO, DIA) - ADMISSAO + 1) / 365). Anos completos de casa. | inteiro |
| `TEMPO_BANCO_GRUPO` | VARCHAR2 | calculado | Faixa do tempo de casa usando o valor fracionário (não o truncado). Ver observação nas análises. | 0 \| 4 · 5 \| 9 · 10 \| 19 · 20 \| 29 · 30 \| 39 · 40+ |
| `PCD` | VARCHAR2 | DO_FUNCIONARIOS.DEFICIENCIA | Pessoa com deficiência (DEFICIENCIA = 1). | Sim \| Não |
| `TPADMISSAO` | VARCHAR2 | DO_FUNCIONARIOS.CONCURSADO | Forma de ingresso (CONCURSADO = 1). | Concursados \| Externos |
| `CARGO` | VARCHAR2 | CS_CARGOS.TITULO | Título do cargo. Títulos iniciados em 'ESCRITU' são normalizados para 'Escriturário'. |  |
| `CARREIRA` | VARCHAR2 | K9_CS_CARREIRAS.NOME, senão CS_CLASSES.NOME | Carreira do cargo. 'PCS Extintos' é renomeado para 'Bancária'. |  |
| `FORMACAO_GRAU` | VARCHAR2 | DO_FUNCIONARIOFORMACOES / TA_NIVEISESCOLARIDADE | Escolaridade vigente em DIA: último curso concluído (graduação, pós, mestrado, doutorado) com vigência até o próximo; se não houver, deriva de NIVELESCOLARIDADE (<=8 Ensino Médio; 'incompleto' ou 9 Graduação; 10 Pós; 11 Mestrado; 12 Doutorado). | Ensino Médio \| Graduação \| Pós-Graduação \| Mestrado \| Doutorado (e nomes de TA_GRAUSENSINO) |
| `DEMISSAO` | DATE | DO_FUNCIONARIOS.DEMISSAODATA | Data de demissão; nula para quem não saiu. |  |
| `GERACAO` | VARCHAR2 | calculado | Pelo ano de nascimento: <1960, 1960-79, 1980-99, >=2000. Corte interno, não é a classificação acadêmica usual. | BOOMER \| GERACAO_X \| GERACAO_Y \| GERACAO_Z |
| `SALARIO` | NUMBER | CS_CLASSESTEPS.SALARIO | Menor salário (degrau inicial) do cargo, empresas 1,2,3. Não é o salário individual do colaborador. |  |
| `ADMISSAO` | DATE | DO_FUNCIONARIOS.DATAADMISSAO | Data de admissão. |  |

### Hierarquia (lotação formal)

| Coluna | Tipo | Origem | Regra / descrição | Valores |
|---|---|---|---|---|
| `HIERARQUIA_ID` | NUMBER | TB_HIERARQUIA → DO_FUNCIONARIOHIERARQUIAS.HIERARQUIA | Unidade da estrutura hierárquica em que o colaborador está vinculado em DIA (vigência INICIO–FIM; se a vigência termina no mês da demissão, estende até o fim do mês). |  |
| `HIERARQUIA_AOD` | VARCHAR2 | TB_HIERARQUIA → K_BRB_HIERARQUIAS.AOD | Atributo da unidade formal (aod). |  |
| `HIERARQUIA_APELIDO` | VARCHAR2 | TB_HIERARQUIA → K_BRB_HIERARQUIAS.HIERARQUIA | Atributo da unidade formal (hierarquia). |  |
| `HIERARQUIA_SUP` | VARCHAR2 | TB_HIERARQUIA → K_BRB_HIERARQUIAS.SUPERINTENDENCIA | Atributo da unidade formal (superintendencia). |  |
| `HIERARQUIA_DIR` | VARCHAR2 | TB_HIERARQUIA → K_BRB_HIERARQUIAS.DIRETORIA | Atributo da unidade formal (diretoria). |  |
| `HIERARQUIA_EMPRESA` | VARCHAR2 | TB_HIERARQUIA → K_BRB_HIERARQUIAS.EMPRESA | Atributo da unidade formal (empresa). |  |
| `HIERARQUIA_ESTADO` | VARCHAR2 | TB_HIERARQUIA → K_BRB_HIERARQUIAS.ESTADO | Atributo da unidade formal (estado). |  |
| `HIERARQUIA_LOCALIZACAO` | VARCHAR2 | TB_HIERARQUIA → K_BRB_HIERARQUIAS.LOCALIZACAO | Atributo da unidade formal (localizacao). |  |
| `HIERARQUIA_TIPO` | VARCHAR2 | TB_HIERARQUIA → K_BRB_HIERARQUIAS.TIPOHIERARQUIA | Atributo da unidade formal (tipohierarquia). |  |

### Lotação efetiva

| Coluna | Tipo | Origem | Regra / descrição | Valores |
|---|---|---|---|---|
| `LOTACAO_ID` | NUMBER | NVL(ID_ADICAO, HIERARQUIA_ID) | Unidade onde o colaborador efetivamente está: a 'adição' (cessão interna) vigente tem prioridade sobre a hierarquia formal. |  |
| `LOTACAO_AOD` | VARCHAR2 | NVL(adição, hierarquia) | AOD da lotação. |  |
| `LOTACAO_APELIDO` | VARCHAR2 | idem | Nome/apelido da lotação. |  |
| `LOTACAO_SUP` | VARCHAR2 | idem | Superintendência da lotação. |  |
| `LOTACAO_DIR` | VARCHAR2 | idem | Diretoria da lotação. |  |
| `LOTACAO_EMPRESA` | VARCHAR2 | idem | Empresa da lotação. |  |
| `LOTACAO_ESTADO` | VARCHAR2 | idem | UF da lotação. |  |
| `LOTACAO_LOCALIZACAO` | VARCHAR2 | idem | Localização/cidade da lotação. |  |
| `LOTACAO_TIPO` | VARCHAR2 | idem | TIPOHIERARQUIA da lotação; vira 'TI' quando a diretoria é DITEC. | TI + domínio de TIPOHIERARQUIA |
| `LOTACAO_SEGMENTO` | VARCHAR2 | idem | Segmentação da lotação. |  |

### Primeira lotação

| Coluna | Tipo | Origem | Regra / descrição | Valores |
|---|---|---|---|---|
| `PRIMEIRA_LOTACAO_ID` | NUMBER | TB_PRIMEIRA_LOTACAO → K_BRB_HIERARQUIAS | Primeira unidade do colaborador (a mais antiga entre cessões internas e hierarquias, desconsiderando 'ONBRB'). Não varia com DIA. |  |
| `PRIMEIRA_LOTACAO_AOD` | VARCHAR2 | TB_PRIMEIRA_LOTACAO → K_BRB_HIERARQUIAS | Atributo da primeira lotação. |  |
| `PRIMEIRA_LOTACAO_APELIDO` | VARCHAR2 | TB_PRIMEIRA_LOTACAO → K_BRB_HIERARQUIAS | Atributo da primeira lotação. |  |
| `PRIMEIRA_LOTACAO_SUP` | VARCHAR2 | TB_PRIMEIRA_LOTACAO → K_BRB_HIERARQUIAS | Atributo da primeira lotação. |  |
| `PRIMEIRA_LOTACAO_DIR` | VARCHAR2 | TB_PRIMEIRA_LOTACAO → K_BRB_HIERARQUIAS | Atributo da primeira lotação. |  |
| `PRIMEIRA_LOTACAO_EMPRESA` | VARCHAR2 | TB_PRIMEIRA_LOTACAO → K_BRB_HIERARQUIAS | Atributo da primeira lotação. |  |
| `PRIMEIRA_LOTACAO_ESTADO` | VARCHAR2 | TB_PRIMEIRA_LOTACAO → K_BRB_HIERARQUIAS | Atributo da primeira lotação. |  |
| `PRIMEIRA_LOTACAO_LOCALIZACAO` | VARCHAR2 | TB_PRIMEIRA_LOTACAO → K_BRB_HIERARQUIAS | Atributo da primeira lotação. |  |
| `PRIMEIRA_LOTACAO_TIPO` | VARCHAR2 | TB_PRIMEIRA_LOTACAO → K_BRB_HIERARQUIAS | Atributo da primeira lotação. DITEC vira 'TI' no TIPO. |  |
| `PRIMEIRA_LOTACAO_SEGMENTACAO` | VARCHAR2 | TB_PRIMEIRA_LOTACAO → K_BRB_HIERARQUIAS | Atributo da primeira lotação. |  |

### Cessão externa

| Coluna | Tipo | Origem | Regra / descrição | Valores |
|---|---|---|---|---|
| `CESSAO_ID` | NUMBER | K9_DO_FUNCIONARIOCESSOES.ORGAOCESSIONARIO | Órgão cessionário (cedido a outra empresa/órgão) em DIA. |  |
| `ADICAO_AOD` | VARCHAR2 | ADM_OUTRASEMPRESAS.NOME | ATENÇÃO: o alias é enganoso, é o NOME do órgão cessionário, não um AOD de adição. |  |

### Cargo / Função

| Coluna | Tipo | Origem | Regra / descrição | Valores |
|---|---|---|---|---|
| `CARGOFUNCAO_NOME` | VARCHAR2 | CS_FUNCOES.NOME, senão CARGO | Nome da função efetiva em DIA; sem função, o cargo. |  |
| `CARGOFUNCAO_TEMPO` | NUMBER | TB_TEMPO_FUNCAO | Anos (1 casa) somados em funções efetivas já encerradas até DIA. Não inclui a função em aberto. |  |
| `CARGOFUNCAO_LIDER` | VARCHAR2 | CS_FUNCOES | Classificação da função por K9_CODIGO e nome. | Liderança \| Gerência \| Técnico \| Sem função |
| `ATIVIDADE_GRATIFICADA` | VARCHAR2 | CS_FUNCOES.K9_ATIVIDADEGRATIFICADA | 'S' = Atividade, 'N' = Função, nulo = Cargo. | Atividade \| Função \| Cargo |
| `CARGOFUNCAO_PROGRAMA` | VARCHAR2 | lista fixa de matrículas | 'Liderança Feminina' para matrículas de uma lista no código cuja função começou em 2019 ou depois. | Liderança Feminina \| Sem Programa \| Sem programa |
| `CARGOFUNCAO_PROMOCAO` | VARCHAR2 | LAG sobre o VR | Compara o VR com o da função anterior da mesma matrícula. Primeira função, valor igual ou não comparável caem em 'Promoção'. | Promoção \| Rebaixamento \| Sem promoção |
| `CARGOFUNCAO_CARREIRA` | VARCHAR2 | K9_CS_TRILHAS.NOME, senão CARREIRA | Trilha da função; sem função, carreira do cargo. |  |
| `CARGOFUNCAO_MOTIVO` | VARCHAR2 | K9_CS_MOTIVOSALTERACAOFUNCAO | Motivo de inclusão/alteração da função, senão o de encerramento, senão 'Outros'. 'Cargo' quando não há função. |  |
| `CARGOFUNCAO_TIPO` | VARCHAR2 | calculado | Se há função efetiva em DIA. | Função \| Cargo |
| `CARGOFUNCAO_VR` | NUMBER | CS_FUNCOES.K9_VALORREFERENCIA / VALOR, senão SALARIO | Valor de referência da função (K9_VALORREFERENCIA se > 0, senão VALOR); sem função, o salário base do cargo. Marcado 'REAVALIAR' no código. |  |

### Substituição

| Coluna | Tipo | Origem | Regra / descrição | Valores |
|---|---|---|---|---|
| `SUBSTITUICAO_ID` | NUMBER | CS_FUNCOES.K9_CODIGO (TIPOFUNCAO = 2) | Código da função em que o colaborador substitui em DIA. Havendo várias, vale a de maior VR. |  |
| `SUBSTITUICAO_NOME` | VARCHAR2 | CS_FUNCOES.NOME | Nome da função substituída. |  |
| `SUBSTITUICAO_LIDER` | VARCHAR2 | CS_FUNCOES | Mesma regra de CARGOFUNCAO_LIDER, sem 'Sem função'. | Liderança \| Gerência \| Técnico |
| `SUBSTITUICAO_CARREIRA` | VARCHAR2 | K9_CS_TRILHAS.NOME | Trilha da função substituída. |  |

### Benefício educacional

| Coluna | Tipo | Origem | Regra / descrição | Valores |
|---|---|---|---|---|
| `BENEFICIOS_QTD` | NUMBER | TB_BENEFICIO_Q | Quantidade de benefícios educacionais (reembolsos 2,4,5,6,15,22). Zero quando não há. Ver observações sobre a lógica de datas. |  |
| `BENEFICIO_ATIVO` | VARCHAR2 | TB_BENEFICIO_ATUAL | 'S' se há benefício vigente em DIA. Na prática só aparece 'S' ou nulo. | S \| nulo |
| `BENEFICIO_INSTITUICAO` | VARCHAR2 | TA_TERCEIROS.NOME | Instituição de ensino. |  |
| `BENEFICIO_CURSO` | VARCHAR2 | TA_FORMACOES.NOME | Curso. |  |
| `BENEFICIO_TIPO` | VARCHAR2 | K9_BS_BENEFICIOEDUCACIONAL.TIPO | 1001 Auxílio Instrução Pós; 1002 Auxílio Instrução Graduação; 1003 Bolsa Estudo; 1004 Crédito Educação; 1006 Auxílio Instrução Mestrado; 1007 Auxílio Instrução Doutorado. |  |
| `BENEFICIO_GRAU` | VARCHAR2 | TA_GRAUSENSINO.NOME | Grau do curso. |  |
| `BENEFICIO_AREACONHECIMENTO` | VARCHAR2 | regras por palavra-chave | Agrupamento do curso por busca de texto no nome e handles de TA_AREASCONHECIMENTO. Ordem das regras importa (a primeira que casa vence). | Inovação e Tendências \| Gestão \| Economia e Finanças \| Ciência de Dados \| Comunicação / Marketing \| Controladoria / Auditoria \| Direito / Jurídico \| Ciências Exatas \| Tecnologia \| Outros |
| `BENEFICIO_INICIO` | DATE | BS_REEMBOLSOFUNCIONARIOS.INICIO | Início do reembolso. |  |
| `BENEFICIO_FIM` | DATE | BS_REEMBOLSOFUNCIONARIOS.FIM | Fim do reembolso. |  |
| `BENEFICIO_INICIOCURSO` | DATE | K9_BS_BENEFICIOEDUCACIONAL.INICIOCURSO | Início do curso. |  |
| `BENEFICIO_TERMINOCURSO` | DATE | K9_BS_BENEFICIOEDUCACIONAL.TERMINOCURSO | Término previsto do curso. |  |

### Férias

| Coluna | Tipo | Origem | Regra / descrição | Valores |
|---|---|---|---|---|
| `GOZO_INICIO` | DATE | FP_FUNCIONARIOFERIASGOZADAS (SITUACAO='G') | Início das férias em curso em DIA. |  |
| `GOZO_FIM` | DATE | idem | Fim das férias em curso. |  |
| `GOZO_DIAS` | NUMBER | idem | Duração das férias em curso (dias). |  |
| `PROGRAMADAS_INICIO` | DATE | FP_FUNCIONARIOFERIASGOZADAS (SITUACAO='P') | Início das próximas férias programadas (a mais próxima futura). |  |
| `PROGRAMADAS_FIM` | DATE | idem | Fim das próximas férias programadas. |  |
| `PROGRAMADA_DIAS` | NUMBER | idem | Duração das férias programadas (dias). Nome no singular, diferente dos demais. |  |

### Situação / Afastamento

| Coluna | Tipo | Origem | Regra / descrição | Valores |
|---|---|---|---|---|
| `SITUACAO` | VARCHAR2 | TB_FERIAS_GOZADAS / TB_AFASTAMENTO | 'Férias' se há gozo em DIA; senão o motivo do afastamento vigente; nulo = em atividade normal. Motivos 1 e 2 são separados em com/sem INSS. | Férias \| Saúde \| Doença INSS \| Acidente \| Acidente INSS \| Maternidade - Prorrogação \| Licença mandato de Diretor \| demais de FP_MOTIVOSAFASTAMENTOS \| nulo |

### Outros

| Coluna | Tipo | Origem | Regra / descrição | Valores |
|---|---|---|---|---|
| `REGIUS` | VARCHAR2 | FP_FUNCIONARIOPAGAMENTO.K_PREVREGIUS | Plano de previdência complementar (Regius). | Não optante \| Plano 1 \| Plano 3 \| Plano CD05 |
| `FUNCAO_DIA` | VARCHAR2 | calculado | Posição exercida no dia, por precedência: substituição, função efetiva, cargo. |  |
| `FUNCAO_TIPO_DIA` | VARCHAR2 | calculado | Origem de FUNCAO_DIA. | CARGO \| FUNCAO \| SUBSTITUICAO |
| `CAUSA_DEMISSAO` | VARCHAR2 | FP_CAUSASDEMISSAO.NOME | Causa do desligamento (só preenchida para desligados no mês). Matrículas de uma lista fixa viram 'PDVI'. | PDVI + FP_CAUSASDEMISSAO |

## 5. Pontos de atenção encontrados na query

Classifiquei por impacto no resultado. São observações da leitura do código; **não testei contra o banco**, então vale validar cada uma antes de alterar.

### Podem alterar números

1. **Linhas duplicadas escondidas por `SELECT DISTINCT`.** `TB_FUNCAO` calcula `RANK` mas o `SELECT` final nunca filtra `RANK = 1`; e a junção de escolaridade (`E3`) pode devolver dois graus com o mesmo `INICIO`. O `DISTINCT` só remove duplicatas idênticas: se uma coluna divergir (ex.: duas funções simultâneas), o colaborador sai **duplicado** e infla headcount. Teste: `COUNT(*)` vs `COUNT(DISTINCT HANDLE||DIA)`.
2. **`TEMPO_BANCO_GRUPO` não bate com `TEMPO_BANCO`.** A faixa usa o tempo fracionário com `<= 4`, `<= 9`...: quem tem 4,5 anos (`TEMPO_BANCO = 4`) cai em `'5 | 9'`, e 9,5 anos cai em `'10 | 19'`. Corrigir usando `TEMPO_BANCO` (inteiro) ou `< 5`, `< 10`...
3. **`BENEFICIOS_QTD` (`TB_BENEFICIO_Q`).** A condição `D.DIA <= CASE WHEN B.FIM < SYSDATE THEN SYSDATE ELSE LAST_DAY(B.FIM) END` parece invertida e deixa de fora benefícios com `FIM` nulo (`LAST_DAY(NULL)`). Também não respeita a data de início. E junta com `RHPRODUCAO.TB_CALENDARIO` (tabela física), não com a CTE de mesmo nome.
4. **`BENEFICIO_ATIVO` só vale 'S' ou nulo.** `TB_BENEFICIO_ATUAL` pega `FUNCIONARIO` de `K9_BS_BENEFICIOEDUCACIONAL` e filtra `IS NOT NULL`: reembolso vigente sem cadastro correspondente nessa tabela não aparece, e nunca sai 'N'.
5. **Férias gozadas só consideram a última férias do colaborador** (`RANK_F = 1`). Para a data de hoje funciona se for a mais recente; em série histórica, férias antigas ficam de fora (e `SITUACAO` ≠ 'Férias').
6. **`CARGOFUNCAO_PROMOCAO`.** A primeira função na janela (desde 2018), ou VR igual, é rotulada 'Promoção' (`ELSE 'Promoção'`). Não é promoção real. Use com cautela em indicador de promoções.
7. **`SALARIO` e `CARGOFUNCAO_VR` quando sem função** são o piso do cargo (menor degrau), não o salário pago.
8. **`CARGOFUNCAO_TEMPO` ignora a função atual**, pois compara `NVL(FIM, SYSDATE) <= DIA` e `SYSDATE` tem hora. Quem está há 10 anos na mesma função aparece com tempo 0/nulo.
9. **`REGIUS` usa subquery escalar**: se `FP_FUNCIONARIOPAGAMENTO` tiver duas linhas por funcionário, a query falha com ORA-01427.

### Qualidade / manutenção

10. **Alias enganoso:** `CESSAO ADICAO_AOD` é o nome do órgão cessionário (sugestão: `CESSAO_NOME`). `PROGRAMADA_DIAS` no singular; `CARGOFUNCAO_PROGRAMA` retorna 'Sem Programa' (CTE) ou 'Sem programa' (NVL), com caixas diferentes.
11. **Joins sem uso** (custo sem retorno): `K_BRB_GENERO`, `K_BRB_ORIENTACAOSEXUAL`, `TA_ESTADOSCIVIS` e o bloco de primeira função (`E5`) estão no `FROM`, mas as colunas estão comentadas. Gênero/orientação podem ser reativados quando houver demanda (tratar como dado sensível).
12. **Escolaridade:** 'incompleto/incompleta' é mapeado para 'Graduação'; confirme se é intencional.
13. **Datas e NLS:** `TO_DATE(SYSDATE)` e `TO_DATE('01/07/2024')` dependem do `NLS_DATE_FORMAT` da sessão. Prefira `TRUNC(SYSDATE)` e `DATE 'YYYY-MM-DD'`.
14. **Calendário via `ROWNUM` sobre `DO_FUNCIONARIOFUNCOES`** só cobre todos os meses se a tabela tiver ao menos ~3.500 linhas. Alternativa robusta: `CONNECT BY LEVEL` sobre `DUAL` (como faz `queries/quadro_pessoal_substituicao.sql`).
15. **Dados pessoais:** `NOME`, `NASCIMENTO`, `SEXO`, `RACA_COR`, `PCD` e (comentados) CPF/e-mail. Para dashboards, avaliar anonimização e acesso por perfil (LGPD).
16. **Performance:** `TB_COLABORADOR` é lido por ~10 CTEs. Em modo histórico o volume é colaboradores × meses; considere `/*+ MATERIALIZE */` ou persistir a base em tabela/MV.

## 6. Receitas de uso

Todas sobre o resultado da query (ex.: `FROM (query) T`). Confirme antes a unicidade por colaborador/dia (item 1 acima).

```sql
-- Efetivo ativo de empregados por diretoria de lotação
SELECT LOTACAO_DIR, COUNT(DISTINCT MATRICULA) HEADCOUNT
FROM (...) T
WHERE COLABORADOR_ID = 1 AND STATUS = 'Ativo'
GROUP BY LOTACAO_DIR;

-- Admissões e desligamentos do mês, com causa
SELECT ADMISSAO_MES, DEMISSAO_MES, CAUSA_DEMISSAO, COUNT(*)
FROM (...) T
WHERE COLABORADOR_ID = 1 AND (ADMISSAO_MES = 'Sim' OR DEMISSAO_MES = 'Sim')
GROUP BY ADMISSAO_MES, DEMISSAO_MES, CAUSA_DEMISSAO;

-- Perfil da liderança por sexo e geração
SELECT CARGOFUNCAO_LIDER, SEXO, GERACAO, COUNT(*)
FROM (...) T
WHERE COLABORADOR_ID = 1 AND STATUS = 'Ativo'
GROUP BY CARGOFUNCAO_LIDER, SEXO, GERACAO;

-- Quem está fora da operação hoje (férias/afastamentos)
SELECT SITUACAO, COUNT(*) FROM (...) T
WHERE STATUS = 'Ativo' AND SITUACAO IS NOT NULL
GROUP BY SITUACAO;
```
