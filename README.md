# 🎬 Análise de Lançamentos Comerciais de Obras Audiovisuais no Brasil: Dados Abertos ANCINE (Power BI + SQL)
[![licensebuttons by](https://licensebuttons.net/l/by/3.0/88x31.png)](https://creativecommons.org/licenses/by/4.0)

## 📌 Descrição do Repositório
Este repositório reúne um projeto de análise de dados abrangendo desde a coleta e tratamento dos dados até a construção de painéis visuais interativos. 
O objetivo do projeto é disponibilizar:
- Os conjuntos de dados originais e tratados;
- O processo de higienização, tratamento e validação de dados via **SQL**;
- Dashboards analíticos desenvolvidos no **Power BI**;
- Interpretação e insights extraídos a partir das informações obtidas.

## 📌 Descrição da Base de Dados
Disponibilizada pelo [Portal Brasileiro de Dados Abertos](https://dados.gov.br/dados/conjuntos-dados/lancamentos-comerciais-por-distribuidoras), a base de dados da Agência Nacional do Cinema ([ANCINE](https://www.gov.br/ancine/pt-br)) utilizada reúne informações de lançamentos comerciais de obras audiovisuais no Brasil. 
- **Arquivo Original:** `lancamentos-comerciais-por-distribuidoras.csv`
- **Data de Coleta/Atualização:** 17/07/2026

### Descrição dos Atributos da Base de Dados (Autoral)

| Atributo | Descrição |
| :--- | :--- |
| `DATA_LANCAMENTO_OBRA` | Data de lançamento comercial da obra no Brasil |
| `TITULO_ORIGINAL` | Nome original da obra |
| `CPB_ROE` | Certificado de Produto Brasileiro (CPB) ou Registro de Obra Estrangeira (ROE) - Identificador único da obra |
| `TIPO_OBRA` | Gênero da obra (ex.: Animação, Documentário, Ficção, etc.) |
| `PAIS_OBRA` | País de origem/produção da obra |
| `PUBLICO_TOTAL` | Total acumulado de espectadores da obra |
| `RENDA_TOTAL` | Renda total bruta arrecadada da obra (R$) |
| `RAZAO_SOCIAL_DISTRIBUIDORA` | Razão social da empresa distribuidora da obra |
| `REGISTRO_DISTRIBUIDORA` | Número de registro da distribuidora da obra na ANCINE |
| `CNPJ_DISTRIBUIDORA` | Cadastro Nacional da Pessoa Jurídica da distribuidora da obra |
---


## 🧹 Tratamento e Higienização da Base de Dados
Para garantir a consistência e confiabilidade dos dados antes da carga no Power BI, foram aplicadas consultas no SQL para identificação de inconsistências temporais, atributos com valor nulo/vazio, registros duplicados e anomalias em identificadores das obras (`CPB_ROE`).

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
- Apesar de dois registros recuperados estarem com o mesmo CPB/ROE, após pesquisa no portal da ANCINE, não foi identificado nenhuma obra vinculada ao CPB/ROE *E1300000100000*.
- Após pesquisa no portal da ANCINE, identificou-se que o registro da obra *FRIDAY THE 13TH* com CPB/ROE *E1500668200000* refere-se a obra de 1890, destoando do recorte da base, que se inicia em 2009.
- Além disso, esses 3 registros têm valores de público e renda inferiores aos outros registros, sendo decidido considerar os registros mencionados erros de indexação e remove-los.

### 4. Padronização e Correção do Identificador Único da Obra (`CPB_ROE`):
Avaliação da consistência dos identificadores únicos das obras:
```
-- Consultar quantidade de distribuidoras por obra
SELECT CPB_ROE,
       COUNT(*) AS frequencia
  FROM bd_ancine
 GROUP BY CPB_ROE
 ORDER BY frequencia DESC;

-- Consultar registros com 2ou+ distribuidoras
SELECT *
  FROM bd_ancine
 WHERE CPB_ROE IN (
     SELECT CPB_ROE
       FROM bd_ancine
      GROUP BY CPB_ROE
     HAVING COUNT(*) > 1
 )
 ORDER BY CPB_ROE;

-- Consultar anomalia, registros com CPB/ROE E1300000100000
SELECT *
  FROM bd_ancine
 WHERE CPB_ROE == 'E1300000100000';
```

### Diagnóstico e Ações:
- Uma obra pode ter uma ou mais distribuidoras como verificado e confirmado pesquisando na internet.
- Anteriormente dois registros com o CPB/ROE *E1300000100000* foram identificados como registro duplicado. Verificando os outros registros vinculados a esse CPB/ROE observa-se que todos são vinculados há uma obra diferente. Portanto, confirmou-se que o CPB/ROE E1300000100000 é um identificador genérico.
- O CPB/ROE das obras *THE BIG FOUR* e *KARATE KID, THE*, após pesquisa no portal da ANCINE, foram identificados e preenchidos (*E1600633500000* e *E1600548700000*, respectivamente).
- Não foi possível identificar o CPB/ROE das outras obras, sendo decidido preencher o CPB/ROE seguindo o padrão `N/C <sequencial>` para preservar o histórico sem comprometer a unicidade desse atributo.
### 5. Exportação da Base de Dados:
Modificação e remoção de registros da base de dados definida (versão final para exportação):
```
-- Alterar data e país
UPDATE bd_ancine SET DATA_LANCAMENTO_OBRA = '14/05/2026' WHERE (CPB_ROE, TITULO_ORIGINAL) = ('E2500270900000', 'IN THE GREY');
UPDATE bd_ancine SET PAIS_OBRA = 'ESTADOS UNIDOS'      WHERE (CPB_ROE, TITULO_ORIGINAL) = ('E2400142000000', 'THE CHOSEN');

-- Remover registros
DELETE FROM bd_ancine WHERE (CPB_ROE, TITULO_ORIGINAL) IN (
     ('E1300000100000','THE KING''S SPEECH'),
     ('E1300000100000','CARAMEL'),
     ('E1500668200000','FRIDAY THE 13TH'));

-- Alterar CPB_ROE
UPDATE bd_ancine SET CPB_ROE = CASE DATA_LANCAMENTO_OBRA
       WHEN '22/06/2010' THEN 'E1600633500000'
       WHEN '27/08/2010' THEN 'E1600548700000'
       WHEN '06/02/2009' THEN 'N/C 1'
       WHEN '20/03/2009' THEN 'N/C 2'
       WHEN '27/03/2009' THEN 'N/C 3'
       WHEN '05/06/2009' THEN 'N/C 4'
       WHEN '11/06/2009' THEN 'N/C 5'
       WHEN '31/07/2009' THEN 'N/C 6'
       WHEN '09/10/2009' THEN 'N/C 7'
       WHEN '19/02/2010' THEN 'N/C 8'
       WHEN '13/01/2014' THEN 'N/C 9'
       WHEN '25/10/2014' THEN 'N/C 10'
       ELSE CPB_ROE
   END
 WHERE CPB_ROE = 'E1300000100000';

-- Consultar registros (base modificada para exportar)
SELECT * 
  FROM bd_ancine;
```
---

## 📊 Análise e Interpretação dos Dashboards

A estrutura analítica no Power BI foi desenvolvida de forma modular, permitindo a exploração dos dados desde um panorama consolidado até análises específicas por segmentação.

### Estrutura e Navegação do Relatório

A partir da página de **Índice**, é possível acessar diretamente os quatro módulos principais do relatório:

* **Visão Geral:** Panorama consolidado com os principais indicadores (KPIs), séries temporais, decomposição dos maiores sucessos de bilheteria e tabela resumo por gênero.
 * **Visão por País:** Replica a estrutura analítica da Visão Geral, aplicando segmentação interativa por país de origem/produção da obra.
  * **Visão por Ano:** Foca no recorte temporal via segmentação por ano de lançamento. Nesta página, os gráficos de linhas temporais são substituídos por uma tabela com os 10 principais países/polos de produção.
  * **Visão por Gênero:** Permite filtrar os indicadores por tipo/gênero das obras. Nesta página, a tabela resumo por gênero dá lugar à tabela com os 10 principais países/polos de produção, facilitando a identificação dos países líderes em cada tipo de obra.


### Análise Detalhada: Visão Geral

A **Visão Geral** consolida as métricas históricas de lançamentos comerciais no Brasil registradas pela ANCINE:

#### 1. Principais Indicadores (KPIs Globais)
* **Total de Obras:** 6.921 obras
* **Público Total Acumulado:** ~ 2,33 bilhões de espectadores
* **Renda Total Bruta:** ~ R$ 33,77 bilhões

#### 2. Destaques de Público e Distribuição
* **Principais Obras por Público Acumulado (CPB/ROE):**
  1. `E2400179900000` (*Inside Out 2 / Divertida Mente 2*) - 22.473.802 espectadores
  2. `E1900107800000` (*Avengers: Endgame / Vingadores: Ultimato*) - 19.656.475 espectadores
  3. `E2100453300000` (*Spider-Man: No Way Home / Homem-Aranha: Sem Volta para Casa*) - 17.382.099 espectadores

> **Domínio de Produções Norte-Americanas:** O topo do ranking de público e renda é amplamente dominado por produções norte-americanas de grande apelo comercial, com destaque para a animação *Inside Out 2* (E2400179900000), que atingiu o recorde histórico de **22.473.802 espectadores** e **~ R$ 444,47 milhões em renda**.

* **Principais Distribuidoras por Arrecadação:**
  1. **The Walt Disney Company:** ~ R$ 7,95 bilhões
  2. **Warner Bros.:** ~ R$ 6,98 bilhões
  3. **Columbia TriStar:** ~ R$ 4,63 bilhões

> **Oligopólio na Distribuição:** Três distribuidoras multinacionais (*The Walt Disney Company*, *Warner Bros.* e *Columbia TriStar*) concentram sozinhas aproximadamente **~ R$ 19,56 bilhões** - correspondendo a quase **58% de toda a renda bruta histórica** registrada na base da ANCINE (~ R$ 33,77 bilhões).

#### 3. Evolução Histórica (Séries Temporais)
* **Volume Anual de Obras:** Crescimento contínuo a partir de 2010, atingindo picos próximos a 500 títulos lançados ao ano até 2019.
* **Impacto da Pandemia (2020):** Queda drástica tanto no público acumulado quanto no número de obras lançadas em 2020, seguida por uma retomada gradual nos anos seguintes.
* **Queda Tendencial no Público Acumulado:** Em contraste com a oferta de títulos, a linha de tendência para o público anual acumulado apresenta **inclinação descendente**. Esse descolamento sinaliza que o aumento na quantidade de obras lançadas não resultou em um crescimento proporcional de espectadores, indicando uma pulverização da audiência e mudanças nos hábitos de consumo dos espectadores.

#### 4. Tabela Resumo por Gênero da Obra
O comparativo entre gêneros das obras evidencia o modelo comercial predominante:
* **Predominância da Ficção:** O gênero Ficção responde pelo maior volume de lançamentos (76,91% do Total) e pela maior fatia do mercado (80,64% da renda Total).
* **Eficiência das Animações:** Apesar de corresponder a apenas 6,24% das obras lançadas, as animações capturam quase 19% de toda a renda e público gerados, exibindo o maior retorno médio por título exibido.
* **Documentários:** Apresentam relevância em volume de títulos (14,77% do Total), consolidando-se como uma produção expressiva em diversidade cultural, mas com circulação de nicho comercial.
---

### 📌 Diagnóstico Estatístico e Insights Conclusivos

A análise descritiva das métricas da ANCINE evidencia um comportamento de **extrema assimetria à direita**, alta dispersão e forte concentração comercial no mercado audiovisual:

| Indicador | Mínimo | Média | Mediana | Máximo | Desvio Padrão (DesvP) | Coef. de Variação (CV) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **Qtd. de Obras por Ano** | 197 | 387,89 | **401** | 520 | 90,26 | **23%** |
| **Público por Obra** | 1 | 334.112,38 | **11.398,50** | 22.473.802 | 1.150.801,06 | **344,44%** |
| **Renda Total por Obra (R$)** | R$ 1,00 | R$ 4.836.430,03 | **R$ 159.337,01** | R$ 444.469.144,86 | R$ 18.021.893,04 | **372,63%** |
### Diagnóstico:
#### 1. Disparidade Extrema e Inadequação da Média Aritmética
* **Dispersão Crítica:** Os Coeficientes de Variação de **344,44%** para público e **372,63%** para renda comprovam a altíssima heterogeneidade da base de dados.
* **Distorção por *Outliers*:** A média de público (~ 334,1 mil) é quase **30 vezes maior** que a mediana (~ 11,4 mil), e a média de arrecadação (~ R$ 4,84 milhões) é cerca de **30 vezes superior** à renda mediana (~ R$ 159,3 mil). Essa discrepância ocorre porque produções de apelo massivo puxam a média para cima de forma desproporcional.

#### 2. Casos Extremos
* **Extremo Inferior:**
  * **Público Mínimo (1 espectador):** Registrado por títulos como os documentários brasileiros *Zona Árida* (2020) e *Pipoca Moderna* (2019), além da ficção alemã *Metropolis* (2026).
  * **Renda Mínima (R$ 1,00):** Registrada pela ficção alemã *Metropolis* (2026).
* **Extremo Superior:**
  * **Recorde de Público e Renda:** A animação norte-americana *Inside Out 2* (2024) atingiu a marca histórica de **22.473.802 espectadores** e **R$ 444.469.144,86** em arrecadação bruta.
* **Volume Anual de Lançamentos:** O piso histórico ocorreu em **2020 (197 obras)** devido ao impacto pandêmico, enquanto o ápice de lançamentos foi atingido em **2025 (520 obras)**.

### Ação:
Optou-se deliberadamente por **não remover os outliers** da base de dados pelos seguintes motivos:
* **Fidelidade à Realidade do Setor:** O mercado audiovisual é estruturalmente regido pela distribuição de cauda longa e assimetria. Excluir os maiores sucessos de bilheteria descaracterizaria a dinâmica econômica real do mercado audiovisual.
* **Preservação de Obras de Nicho e Culturais:** Manter registros com bilheterias mínimas assegura a representatividade de produções independentes, regionais e experimentais que cumprem circuitos reduzidos de exibição comercial.

> **💡 Conclusão:** O mercado audiovisual brasileiro opera sob a dinâmica de distribuição de cauda longa e Pareto: a maioria dos títulos compõe um catálogo diverso com circulação restrita, enquanto uma fração mínima de produções (notadamente animações e franquias estrangeiras) concentra quase a totalidade da receita e do público pagante.
---
## 👤 Contribuições e Contribuidores
Os dashboards foram desenvolvidos pelo gestor da informação, Felipe dos Santos de Oliveira (**[LinkedIn](https://www.linkedin.com/in/felipe-so/)**).

Caso tenha dúvidas ou sugestões, entre em contato através do GitHub ou envie um e-mail para: **[felipeoliveira.feol@gmail.com.br](mailto:forlok307@gmail.com.br)**.
