# Dashboards Autorais criados no Power BI
[![licensebuttons by](https://licensebuttons.net/l/by/3.0/88x31.png)](https://creativecommons.org/licenses/by/4.0)

## Descrição do Repositório
Este repositório reúne um projeto de análise de dados, abrangendo desde a coleta e tratamento dos dados até a construção de painéis visuais interativos. 
O objetivo do projeto é disponibilizar:
- Os conjuntos de dados originais e tratados;
- O processo de higienização, tratamento e validação de dados via **SQL**;
- Dashboards analíticos desenvolvidos no **Power BI**;
- Interpretação estratégica e insights extraídos a partir das informações obtidas.

## Descrição da Base de Dados
A base de dados da Agência Nacional do Cinema ([ANCINE](https://www.gov.br/ancine/pt-br)) utilizada reúne informações de lançamentos comerciais de obras audiovisuais no Brasil, disponibilizada pelo [Portal Brasileiro de Dados Abertos](https://dados.gov.br/dados/conjuntos-dados/lancamentos-comerciais-por-distribuidoras). 
- **Arquivo Original:** `lancamentos-comerciais-por-distribuidoras.csv`
- **Data de Coleta/Atualização:** 17/07/2026

### Dicionário Autoral de Dados

| Atributo | Descrição |
| :--- | :--- |
| `DATA_LANCAMENTO_OBRA` | Data de lançamento comercial da obra no Brasil |
| `TITULO_ORIGINAL` | Nome original da obra |
| `CPB_ROE` | Certificado de Produto Brasileiro (CPB) ou Registro de Obra Estrangeira (ROE) - Identificador único da obra |
| `TIPO_OBRA` | Gênero da obra (ex.: Animação, Documentário, Ficção, etc.) |
| `PAIS_OBRA` | País de origem/produção da obra |
| `PUBLICO_TOTAL` | Total acumulado de espectadores da obra |
| `RENDA_TOTAL` | Renda total bruta arrecadada da obra (R$) |
| `RAZAO_SOCIAL_DISTRIBUIDORA` | Razão social da empresa distribuidora |
| `REGISTRO_DISTRIBUIDORA` | Número de registro da distribuidora na ANCINE |
| `CNPJ_DISTRIBUIDORA` | Cadastro Nacional da Pessoa Jurídica da distribuidora |
---

## 🧹 Tratamento e Higienização da Base de Dados
Para garantir a consistência e confiabilidade dos dados antes da carga no Power BI, foram aplicadas consultas no SQL para identificação de inconsistências temporais, atributos com valor nulo/vazio, registros duplicados e anomalias em identificadores (`CPB_ROE`).

- **Fonte de Validação de Dados usada:** [Portal de Consulta de Obras Não Publicitárias da ANCINE](https://sad2.ancine.gov.br/obrasnaopublicitarias/consultarObraViaPortal/consultarObraViaPortal.seam)

> Ao todo, **3 registros foram removidos** e **14 registros foram modificados** (1 preenchimento de valor vazio e 13 substituições de valores).


### Principais Etapas Aplicadas:
### 1. Verificação de Inconsistências Temporais (Lançamentos Futuros)
Identificação da existência de registros com data de lançamento comercial da obra no Brasil posterior à data de coleta da base (17/07/2026):
```
-- Consultar os 100 primeiros registros
SELECT *
  FROM bd_ancine
 LIMIT 100;

-- Consultar registros após 17/07/2026
SELECT *
  FROM bd_ancine
 LIMIT 7;
```
### Diagnóstico e Ações:
- Como os dados estavam ordenados decrescentemente por data de lançamento comercial da obra no Brasil, identificou-se que 7 registros apresentavam datas posteriores ao dia da coleta e possuíam valores preenchidos de público e renda.
- Para a obra *IN THE GREY*, identificou-se via pesquisa na internet que seu lançamento foi em 14/05/2026, sendo decidido manter o registro e ajustar a data.
- Além disso, confirmou-se via pesquisa na internet que a data de lançamento das demais obras então corretas. Portanto, os valores de público e renda das obras correspondem aos valores globais (não só no Brasil) até 17/07/2026, sendo decidido manter todos esses registros.

### 2. Tratamento de Valores Nulos/Vazios:
Identificação de atributos em branco ou não preenchidos:
```
-- Consultar registros com valores vazios
SELECT *
  FROM bd_ancine
 WHERE DATA_LANCAMENTO_OBRA IS NULL
    OR TITULO_ORIGINAL IS NULL OR TRIM(TITULO_ORIGINAL) = ''
    OR CPB_ROE IS NULL OR TRIM(CPB_ROE) = ''
    OR TIPO_OBRA IS NULL OR TRIM(TIPO_OBRA) = ''
    OR PAIS_OBRA IS NULL OR TRIM(PAIS_OBRA) = ''
    OR PUBLICO_TOTAL IS NULL
    OR RENDA_TOTAL IS NULL
    OR RAZAO_SOCIAL_DISTRIBUIDORA IS NULL OR TRIM(RAZAO_SOCIAL_DISTRIBUIDORA) = ''
    OR REGISTRO_DISTRIBUIDORA IS NULL OR TRIM(REGISTRO_DISTRIBUIDORA) = ''
    OR CNPJ_DISTRIBUIDORA IS NULL OR TRIM(CNPJ_DISTRIBUIDORA) = '';
```
### Diagnóstico e Ações:
- O registro da obra *THE CHOSEN* estava com o atributo `PAIS_OBRA` vazio. Após pesquisa no portal da ANCINE, o país de origem da obra foi identificado e preenchido como *ESTADOS UNIDOS*.
- O registro com o atributo `Registro Distribuidora` vazio será mantido como esta. Essa decisão foi tomada pois o atributo usado para identificação de distribuidora será `RAZAO_SOCIAL_DISTRIBUIDORA`, ou seja, não impacta nas análises no Power BI.


### 3. Remoção de Duplicidades e Erros de Indexação:
Identificação de registros com a mesma data de lançamento, título original e distribuidora:
```
-- Consultar registros duplicados e erros de indexação
SELECT *
  FROM bd_ancine
 WHERE (DATA_LANCAMENTO_OBRA, TITULO_ORIGINAL, RAZAO_SOCIAL_DISTRIBUIDORA) IN (
     SELECT DATA_LANCAMENTO_OBRA,
            TITULO_ORIGINAL,
            RAZAO_SOCIAL_DISTRIBUIDORA
       FROM bd_ancine
      GROUP BY DATA_LANCAMENTO_OBRA,
               TITULO_ORIGINAL,
               RAZAO_SOCIAL_DISTRIBUIDORA
     HAVING COUNT(*) > 1
 )
 ORDER BY TITULO_ORIGINAL, DATA_LANCAMENTO_OBRA;
```
### Diagnóstico e Ações:
- Não foi identificado uma obra vinculada ao CPB/ROE *E1300000100000*, após pesquisa no portal da ANCINE.
- Após pesquisa no portal da ANCINE, identificou-se que a obra *FRIDAY THE 13TH* com CPB/ROE *E1500668200000* refere-se a obra de 1890, destoando do recorte da base (que se inicia em 2009).
- Além disso, esses 3 registros tem valores de público e renda inferiores aos outros registros, sendo decidido considerar os registro mencionados erros de indexação e remove-los.

### 4. Padronização e Correção do Identificador Único da Obra (`CPB_ROE`):
Avaliação da consistência dos identificadores únicos das obras:
```
-- Consultar quantidade de distribuidoras por obra
SELECT CPB_ROE,
       COUNT(*) AS frequencia
  FROM bd_ancine
 GROUP BY CPB_ROE
 ORDER BY frequencia DESC;

-- Consultar registros com 2+ distribuidoras
SELECT *
  FROM bd_ancine
 WHERE CPB_ROE IN (
     SELECT CPB_ROE
       FROM bd_ancine
      GROUP BY CPB_ROE
     HAVING COUNT(*) > 1
 )
 ORDER BY CPB_ROE;

-- Consultar anomalia no CPB/ROE E1300000100000
SELECT *
  FROM bd_ancine
 WHERE CPB_ROE == 'E1300000100000';
```
### Diagnóstico e Ações:
- Uma obra pode ter uma ou mais distribuidoras como verificado e confirmado pesquisando na internet.
- Anteriormente dois registros com o CPB/ROE *E1300000100000* foram identificados como registro duplicado. Verificando os outros reigistros vinculados a esse CPB/ROE observa-se que todos são vinculados há uma obra diferente. Portanto, confirmou-se que o CPB/ROE E1300000100000 é um identificador genérico.
- O CPB/ROE das obras *THE BIG FOUR* e *KARATE KID, THE*, após pesquisa no portal da ANCINE, foram identificados e preenchidos (*E1600633500000* e *E1600548700000*, respectivamente).
- Não foi possível identificar o CPB/ROE das outras obras, sendo decidido preencher o CPB/ROE seguindo o padrão `N/C <sequencial>` para preservar o histórico sem comprometer a unicidade desse atributo.


## Instrução de Instalação

### Pré Requsitos
- Instale o [R](https://www.r-project.org/) e o [RStudio](https://posit.co/downloads/).
   - xxx

## Instruções de Uso dos Arquivos
1. xxx.

## Interpretação

### Análises estatísticas:
### 1. Estatísticas sobre a Renda Total por Obra
- **Alta assimetria e presença de outliers extremos:** A média (R$ 4,70 milhões) é muito superior à mediana (R$ 145,97 mil). Isso indica que a maioria das obras fatura valores modestos (metade delas até ~R$ 146 mil), enquanto poucas obras com arrecadações astronômicas (máximo de R$ 444,47 milhões, obra *INSIDE OUT 2*) puxam a média para cima.
- **Extrema heterogeneidade:** O desvio padrão (R$ 17,77 milhões) é quase 4 vezes maior que a média, resultando em um CV de 377,78%. A base é extremamente dispersa e volátil.
- **Métrica recomendada de centro:** Mediana.

### 2. Estatísticas sobre o Público Total por Obra
- **Comportamento idêntico ao da renda (forte assimetria):** A média (324,94 mil pessoas) é cerca de 31 vezes maior que a mediana (10,37 mil pessoas). Isso mostra que 50% das obras têm público de até ~10,4 mil espectadores, mas sucessos massivos (máximo de 22,47 milhões, obra *INSIDE OUT 2*) inflam a média.
- **Altíssima dispersão:** O desvio padrão de 1,13 milhão gera um CV de 348,93%, confirmando um conjunto de dados muito desigual (típico do mercado de entretenimento: poucos blockbusters e muitas obras de nicho).
- **Métrica recomendada de centro:** Mediana. 

### 3. Estatísticas sobre a Quantidade de Obras por Ano
- **Comportamento equilibrado e simétrico:** A média (398,83) e a mediana (414) são muito próximas, indicando uma distribuição regular ao longo dos anos, sem distorções graves.
- **Dispersão moderada/baixa:** O desvio padrão é de 97,01 obras/ano, com um CV de 24%, apontando uma produção anual consistente e previsível (variando entre o mínimo de 198 em *2020* e o máximo de 570 obras em *2025*).
- **Métrica recomendada de centro:** Média.



### Informações obtidas
- xxx

## Contribuições e Contribuidores
Os dashboards foram desenvolvidos pelo gestor da informação, Felipe dos Santos de Oliveira (**[LinkedIn](https://www.linkedin.com/in/felipe-so/)**).

Caso tenha dúvidas ou sugestões, entre em contato através do GitHub ou envie um e-mail para: **[felipeoliveira.feol@gmail.com.br](mailto:forlok307@gmail.com.br)**.

