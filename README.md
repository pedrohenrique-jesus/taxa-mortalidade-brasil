# Análise da Mortalidade no Brasil

## Evolução temporal, desigualdades regionais e perfil demográfico das principais causas de óbito

Este projeto apresenta uma análise da mortalidade no Brasil entre **1996 e 2019**, utilizando dados públicos do **Sistema de Informações sobre Mortalidade (SIM)**, disponibilizados pelo Ministério da Saúde e acessados por meio da Base dos Dados.

O objetivo foi investigar a evolução da mortalidade ao longo do tempo, identificar desigualdades territoriais, compreender as principais causas de óbito e analisar o perfil demográfico dos registros.

O projeto combina **SQL, Google BigQuery, Python e Looker Studio**, desde a consulta e preparação de mais de 22 milhões de registros até a construção de um dashboard interativo para comunicação dos resultados.

---

## Objetivo

A análise foi desenvolvida para responder a quatro questões principais:

- Como a mortalidade brasileira evoluiu entre 1996 e 2019?
- Existem diferenças relevantes entre regiões, estados e municípios?
- Quais grupos de causas concentram o maior número de óbitos e como sua participação mudou ao longo do tempo?
- Como os óbitos registrados se distribuem segundo idade, sexo e raça/cor?

Além da exploração dos dados, o projeto buscou transformar uma base de grande volume e alta granularidade em informações adequadas para análise e visualização, mantendo atenção à qualidade dos dados e às limitações dos indicadores utilizados.

---

## Fontes de dados

Foram utilizadas três bases públicas disponíveis no projeto `basedosdados` no Google BigQuery.

### Sistema de Informações sobre Mortalidade — SIM

Tabela:

`basedosdados.br_ms_sim.municipio_causa_idade_sexo_raca`

Principais campos utilizados:

`ano`, `sigla_uf`, `id_municipio`, `causa_basica`, `idade`, `sexo`, `raca_cor` e `numero_obitos`.

A base utilizada possui aproximadamente **22,6 milhões de registros**, correspondentes a **26.294.649 óbitos registrados entre 1996 e 2019**.

Sua granularidade original corresponde à combinação:

**ano × município × causa básica × idade × sexo × raça/cor**

### População municipal

Tabela:

`basedosdados.br_ibge_populacao.municipio`

Utilizada para incorporar a população dos municípios em cada ano e permitir o cálculo das taxas de mortalidade.

### Diretório de municípios

Tabela:

`basedosdados.br_bd_diretorios_brasil.municipio`

Utilizada para complementar os códigos municipais com informações territoriais, como nome do município, unidade federativa, região e centroide geográfico.

---

## Tecnologias utilizadas

- **Google BigQuery** — consulta, agregação, transformação e integração das bases
- **SQL** — preparação das tabelas analíticas e criação das métricas
- **Python**
- **Pandas**
- **NumPy**
- **Matplotlib**
- **Google Colab** — análise exploratória e validação
- **Looker Studio** — desenvolvimento do dashboard interativo
- **Git/GitHub** — versionamento e documentação do projeto

---

## Arquitetura do projeto

O fluxo de trabalho foi estruturado para evitar o carregamento desnecessário de milhões de registros no ambiente Python.

```text
Bases públicas
      │
      ▼
Google BigQuery
      │
      ├── Agregação
      ├── Tratamento
      ├── Integração
      └── Criação de variáveis
      │
      ▼
Tabelas analíticas
      │
      ├──────────────► Google Colab
      │                 │
      │                 └── EDA e validações
      │
      └──────────────► Looker Studio
                        │
                        └── Dashboard interativo
```

As bases públicas permaneceram em sua origem. Foram materializados no projeto apenas os conjuntos de dados necessários para análise e visualização.

---

## Preparação e modelagem dos dados

Foram criadas duas tabelas analíticas principais no BigQuery.

### `base_mortalidade_municipal`

Granularidade:

**ano × município**

Principais campos:

- ano
- UF
- município
- região
- número de óbitos
- população
- taxa de mortalidade por 100 mil habitantes
- centroide

Antes da integração com a população, os registros de mortalidade foram agregados ao nível município-ano.

Essa etapa é importante porque a base original possui diversas observações para o mesmo município e ano. Realizar o `JOIN` antes da agregação provocaria a repetição da população municipal e distorceria os indicadores.

A taxa bruta foi calculada como:

**Taxa de mortalidade = (número de óbitos / população) × 100.000**

A tabela final possui aproximadamente **132,9 mil registros município-ano**.

### `base_perfil_mortalidade`

Granularidade:

**ano × UF × causa × faixa etária × sexo × raça/cor**

A tabela foi desenvolvida para as análises de causas de mortalidade e perfil demográfico.

Entre as variáveis derivadas estão:

- faixa etária;
- categoria CID-10;
- grupo de causa de acordo com os capítulos da CID-10;
- descrição de sexo;
- descrição de raça/cor.

A tabela final possui aproximadamente **4,6 milhões de registros** e preserva o total de **26.294.649 óbitos** da base utilizada.

---

## Consultas SQL e validação dos dados

As consultas foram executadas no Google BigQuery para validar as tabelas analíticas e investigar a evolução temporal e a distribuição geográfica e demográfica da mortalidade no Brasil entre 1996 e 2019.

### 1\. Validação das tabelas analíticas

Para verificar a consistência das tabelas utilizadas nas análises, foi realizada uma consulta para contar os registros e comparar a soma de óbitos entre as duas tabelas materializadas.

```sql
SELECT
    'base_mortalidade_municipal' AS tabela,
    COUNT(*) AS total_linhas,
    SUM(numero_obitos) AS total_obitos
FROM `saude-sim-brasil.mortalidade_brasil.base_mortalidade_municipal`

UNION ALL

SELECT
    'base_perfil_mortalidade' AS tabela,
    COUNT(*) AS total_linhas,
    SUM(numero_obitos) AS total_obitos
FROM `saude-sim-brasil.mortalidade_brasil.base_perfil_mortalidade`;
```

**Resultados da validação:**

**Resultados observados:**

| Tabela analítica | Registros | Total de óbitos |
|---|---:|---:|
| `base_mortalidade_municipal` | 132.868 | 26.294.649 |
| `base_perfil_mortalidade` | 4.626.654 | 26.294.649 |

As duas tabelas apresentaram o mesmo total agregado de óbitos. Essa verificação fornece evidência de consistência entre os totais das tabelas, embora não substitua a validação da granularidade e da ausência de duplicidades em cada dimensão.

### 2\. Evolução temporal da mortalidade

**Pergunta de negócio:** Como evoluíram o número de óbitos registrados e a taxa bruta de mortalidade no Brasil entre 1996 e 2019?

Para analisar a evolução anual, foi utilizada uma consulta com `SUM()`, `SAFE_DIVIDE()`, `ROUND()`, `GROUP BY` e `ORDER BY`. A taxa bruta de mortalidade foi calculada por 100 mil habitantes, considerando os registros com população disponível.

```sql
SELECT
    ano,
    SUM(numero_obitos) AS total_obitos,
    SUM(populacao) AS populacao_total,
    ROUND(
        100000 * SAFE_DIVIDE(
            SUM(numero_obitos),
            SUM(populacao)
        ),
        2
    ) AS taxa_mortalidade_100k
FROM `saude-sim-brasil.mortalidade_brasil.base_mortalidade_municipal`
WHERE populacao IS NOT NULL
GROUP BY ano
ORDER BY ano;
```

**Resultados observados:**

| Indicador | 1996 | 2019 |
|---|---:|---:|
| Óbitos registrados | 849.354 | 1.347.263 |
| Taxa por 100 mil habitantes | 541,71 | 641,10 |

No período analisado, o número de óbitos registrados aumentou aproximadamente 58,62%, enquanto a taxa bruta de mortalidade cresceu aproximadamente 18,35%. Esses indicadores descrevem a evolução observada nos dados e não estabelecem, isoladamente, as causas dessas mudanças.

### 3\. Outras perguntas investigadas

As consultas do notebook também exploraram:

-   **Distribuição municipal:** comparação das taxas de mortalidade entre 5.570 municípios em 2019 e identificação de valores extremos.
    
-   **Desigualdades territoriais:** comparação das taxas brutas entre regiões e unidades federativas.
    
-   **Causas de óbito:** participação dos grupos de causas classificados pela CID-10 e sua evolução temporal.
    
-   **Perfil demográfico:** distribuição dos óbitos por faixa etária, sexo e raça/cor.
    
-   **Qualidade dos dados:** avaliação da completude dos registros de raça/cor, com recorte de 2014 a 2019 para as análises dessa variável.
    

Os resultados detalhados, as demais consultas e as visualizações estão documentados no notebook de análise. As interpretações devem considerar as limitações das bases e a diferença entre a composição dos óbitos registrados e as taxas de mortalidade específicas.

## Tratamento dos dados

Durante a preparação foram verificadas inconsistências, valores ausentes e compatibilidade entre as diferentes fontes.

### Idade

Os registros foram agrupados nas seguintes faixas:

- 0–14 anos
- 15–24 anos
- 25–44 anos
- 45–64 anos
- 65–79 anos
- 80 anos ou mais
- Não informado

Registros sem idade disponível foram preservados como **Não informado**, evitando a imputação de valores sem evidência.

### Sexo

Os códigos foram convertidos para categorias interpretáveis:

- Masculino
- Feminino
- Não informado

### Raça/cor

Os registros foram padronizados nas categorias:

- Branca
- Preta
- Amarela
- Parda
- Indígena
- Não informado

A qualidade dessa variável apresentou forte evolução ao longo da série histórica.

Em 1996, aproximadamente **96,7% dos óbitos não possuíam informação de raça/cor**. Em 2019, essa proporção havia caído para aproximadamente **2,5%**.

Por esse motivo, análises específicas de composição por raça/cor foram restritas ao período **2014–2019**, quando o preenchimento da variável permaneceu acima de 95%.

---

## Classificação das causas de óbito

A causa básica dos óbitos foi utilizada para construir dois níveis de análise.

O primeiro corresponde à **categoria CID-10**, obtida a partir dos três primeiros caracteres do código.

O segundo corresponde aos **grupos de causa**, definidos de acordo com os capítulos da Classificação Internacional de Doenças.

Entre os grupos analisados estão:

- doenças do aparelho circulatório;
- neoplasias;
- causas externas de morbidade e mortalidade;
- doenças do aparelho respiratório;
- doenças endócrinas, nutricionais e metabólicas;
- doenças do aparelho digestivo;
- doenças infecciosas e parasitárias;
- entre outros.

Essa classificação permitiu analisar tanto grandes grupos de mortalidade quanto categorias CID específicas.

---

# Análise Exploratória dos Dados

A EDA foi realizada no Google Colab a partir de consultas agregadas no BigQuery.

Essa estratégia permitiu explorar os dados sem transferir os milhões de registros da base analítica para a memória do notebook.

**Pergunta de negócio:** Como evoluíram o número de óbitos registrados e a taxa bruta de mortalidade no Brasil entre 1996 e 2019?

**Resultado:** A consulta SQL realizada no BigQuery identificou um aumento de 849.354 óbitos em 1996 para 1.347.263 em 2019, enquanto a taxa bruta de mortalidade passou de 541,71 para 641,10 óbitos por 100 mil habitantes. Isso representa aumentos aproximados de 58,62% no número de óbitos e 18,35% na taxa bruta.

**Limitação:** Os resultados descrevem a evolução dos indicadores registrados no período, mas não permitem identificar isoladamente as causas do aumento. A taxa apresentada é bruta, não ajustada por idade, e deve ser interpretada considerando as características e limitações das bases de dados utilizadas.

## Evolução da mortalidade

Entre 1996 e 2019, o número absoluto de óbitos registrados apresentou crescimento de aproximadamente **58,6%**.

A taxa bruta de mortalidade passou de aproximadamente:

**541,7 óbitos por 100 mil habitantes em 1996**

para:

**641,1 óbitos por 100 mil habitantes em 2019**

um aumento de aproximadamente **18,4%**.

Ao longo do período analisado, a taxa média foi de aproximadamente **580,7 óbitos por 100 mil habitantes**, com mediana de **567,7**.

---

## Distribuição municipal e outliers

Para 2019, foram analisadas as taxas de mortalidade de aproximadamente **5.570 municípios**.

A distribuição apresentou:

- média: 675,3
- mediana: 664,2
- desvio-padrão: 197,1
- mínimo: 98,0
- máximo: 2.060,1 óbitos por 100 mil habitantes

Utilizando o método do intervalo interquartil, foram identificados **110 municípios como outliers**, aproximadamente 2% do total.

A investigação mostrou que **86,4% desses municípios possuíam até 10 mil habitantes**.

Isso evidencia um problema importante de interpretação: taxas municipais podem apresentar grande volatilidade quando calculadas sobre populações pequenas.

Por esse motivo, o ranking municipal apresentado no dashboard considera apenas municípios com **10 mil habitantes ou mais**. Os demais registros permanecem preservados na base analítica.

---

## Desigualdades regionais

Em 2019, as taxas brutas regionais foram:

| Região | Taxa por 100 mil habitantes |
|---|---:|
| Sudeste | 695,70 |
| Sul | 687,08 |
| Nordeste | 616,98 |
| Centro-Oeste | 545,40 |
| Norte | 463,91 |

A diferença entre Sudeste e Norte foi de aproximadamente **232 óbitos por 100 mil habitantes**.

Entre as unidades federativas, as maiores taxas brutas observadas em 2019 ocorreram no **Rio de Janeiro** e no **Rio Grande do Sul**, com aproximadamente **832,0** e **784,4 óbitos por 100 mil habitantes**, respectivamente.

Esses valores representam **taxas brutas**, sem padronização por idade. Portanto, diferenças na estrutura etária das populações precisam ser consideradas antes de interpretar essas taxas como diferenças de risco ou desempenho dos sistemas de saúde.

---

## Principais grupos de causas

No período completo, as doenças do aparelho circulatório representaram o principal grupo de causas de óbito.

| Grupo de causa | Óbitos | Participação |
|---|---:|---:|
| Doenças do aparelho circulatório | 7.359.092 | 27,99% |
| Neoplasias | 3.955.347 | 15,04% |
| Causas externas | 3.238.259 | 12,32% |
| Doenças do aparelho respiratório | 2.778.279 | 10,57% |
| Sintomas, sinais e achados anormais | 2.379.961 | 9,05% |

Além da concentração, a composição das causas mudou ao longo do período.

Entre 1996 e 2019:

- a participação das **neoplasias aumentou aproximadamente 6,1 pontos percentuais**;
- as doenças respiratórias aumentaram aproximadamente 2,3 p.p.;
- as doenças circulatórias permaneceram próximas de 27% do total;
- as causas externas diminuíram aproximadamente 2,6 p.p.;
- sintomas, sinais e achados anormais diminuíram aproximadamente 9,5 p.p.

Esses resultados descrevem a evolução observada nos registros. A análise não atribui causalidade às mudanças.

---

## Perfil etário

A mortalidade registrada apresentou forte concentração nas faixas etárias mais elevadas.

No período completo:

| Faixa etária | Participação nos óbitos |
|---|---:|
| 65–79 anos | 28,80% |
| 80 anos ou mais | 24,54% |
| 45–64 anos | 24,09% |
| 25–44 anos | 11,75% |
| 0–14 anos | 6,05% |
| 15–24 anos | 4,25% |
| Não informado | 0,52% |

Aproximadamente **77,4% dos óbitos registrados ocorreram entre pessoas com 45 anos ou mais**.

---

## Sexo e faixa etária

No período completo, aproximadamente **57,1% dos óbitos foram masculinos** e **42,8% femininos**.

Entretanto, essa composição varia significativamente conforme a idade.

Entre 15 e 24 anos, aproximadamente **80,6% dos óbitos registrados foram masculinos**.

Com o avanço da idade, essa diferença diminui. Na faixa de 80 anos ou mais, o padrão se inverte: aproximadamente **57,0% dos óbitos registrados foram femininos**.

Esses percentuais representam a **composição dos óbitos registrados**, e não taxas específicas de mortalidade por sexo, pois não foram utilizados denominadores populacionais por sexo e faixa etária.

---

## Raça/cor

Considerando exclusivamente **2014–2019**, período selecionado devido à maior completude da variável, a composição dos óbitos foi aproximadamente:

| Raça/cor | Participação |
|---|---:|
| Branca | 50,94% |
| Parda | 37,00% |
| Preta | 7,63% |
| Não informado | 3,57% |
| Amarela | 0,56% |
| Indígena | 0,30% |

Esses valores também representam **composição dos registros**, e não risco de mortalidade por raça/cor.

Para comparar riscos entre os grupos seriam necessários denominadores populacionais compatíveis por raça/cor.

---

# Dashboard

Os resultados foram consolidados em um dashboard interativo desenvolvido no **Looker Studio**.

**[Acessar o dashboard interativo no Looker Studio](https://datastudio.google.com/u/0/reporting/a75f25a1-5983-4c63-b6fc-42275595ca9b/page/ETY9F)**

O painel é disponibilizado por meio de um link online; por isso, o repositório contém o notebook, este README e o relatório analítico, sem um arquivo local do dashboard.

O dashboard foi organizado em quatro páginas.

### 1. Panorama Nacional

Apresenta os principais indicadores de mortalidade, evolução histórica da taxa e diferenças regionais.

Principais elementos:

- total de óbitos;
- população;
- taxa bruta por 100 mil habitantes;
- número de municípios;
- evolução histórica;
- comparação entre regiões.

### 2. Análise Territorial

Permite aprofundar a análise espacial por região, estado e município.

Inclui:

- ranking das taxas estaduais;
- ranking municipal;
- filtros territoriais;
- critério mínimo de 10 mil habitantes para o ranking municipal.

### 3. Causas de Mortalidade

Explora os grupos e categorias CID-10 associados aos óbitos.

Inclui:

- ranking dos principais grupos de causas;
- categorias CID-10;
- filtro por grupo de causa;
- evolução histórica dos principais grupos.

### 4. Perfil Demográfico

Apresenta a composição dos óbitos segundo:

- faixa etária;
- sexo e faixa etária;
- raça/cor.

A análise de raça/cor utiliza exclusivamente o período **2014–2019** devido à qualidade histórica da variável.

---

# Principais insights

A análise permitiu identificar quatro resultados centrais:

**1. A mortalidade registrada cresceu em termos absolutos e também em relação à população.**  
Entre 1996 e 2019, os óbitos aumentaram 58,6%, enquanto a taxa bruta aumentou aproximadamente 18,4%.

**2. Existem diferenças territoriais relevantes.**  
As taxas brutas variam consideravelmente entre regiões, estados e municípios. Entretanto, taxas municipais extremas estão fortemente associadas a populações pequenas, exigindo cautela na interpretação.

**3. As doenças do aparelho circulatório permanecem como principal grupo de causas.**  
Ao mesmo tempo, a participação das neoplasias aumentou ao longo da série histórica e a composição das principais causas apresentou mudanças relevantes.

**4. O perfil dos óbitos varia significativamente segundo idade e sexo.**  
A maior parte dos registros concentra-se a partir dos 45 anos, enquanto a participação masculina é particularmente elevada entre os mais jovens e a feminina passa a predominar a partir dos 80 anos.

---

# Cuidados metodológicos

Os resultados devem ser interpretados considerando algumas limitações.

As taxas apresentadas são **taxas brutas de mortalidade**, sem padronização por idade. Portanto, diferenças territoriais podem ser influenciadas pela estrutura demográfica de cada população.

Taxas calculadas para municípios pequenos podem apresentar grande volatilidade. Por esse motivo, o ranking municipal do dashboard utiliza um limite mínimo de 10 mil habitantes.

As análises de sexo, idade e raça/cor apresentadas como percentuais representam **composição dos óbitos**, não risco individual de mortalidade.

A variável raça/cor apresentou elevada incompletude nos primeiros anos da série. Sua análise foi, portanto, limitada a 2014–2019.

Além disso, os resultados são observacionais e descritivos. Associações identificadas nos dados não devem ser interpretadas automaticamente como relações causais.

---

# Conclusão

O projeto demonstrou como bases públicas de grande volume podem ser transformadas em informações úteis por meio de um pipeline que integra **SQL, BigQuery, Python e Business Intelligence**.

A análise revelou crescimento da taxa bruta de mortalidade ao longo do período, importantes diferenças territoriais, predominância das doenças do aparelho circulatório e mudanças na composição das causas de óbito.

Também evidenciou a importância de considerar fatores metodológicos durante a interpretação dos indicadores. Populações pequenas, ausência de padronização etária e qualidade desigual de determinadas variáveis podem produzir conclusões equivocadas quando analisadas sem contexto.

Mais do que apresentar números, o projeto buscou construir uma análise **reproduzível, transparente e orientada à interpretação responsável dos dados**.

---

## Possíveis extensões

O projeto pode ser aprofundado futuramente por meio da:

- padronização das taxas de mortalidade por idade;
- incorporação de denominadores populacionais por sexo, idade e raça/cor;
- análise específica de categorias CID-10;
- investigação das diferenças entre capitais, regiões metropolitanas e interior;
- análise espacial e temporal de causas específicas;
- incorporação de indicadores socioeconômicos e de infraestrutura de saúde.

---

## Autor

**Pedro Henrique**

Projeto desenvolvido como parte da formação em **Análise de Dados**, com foco em SQL, Python, BigQuery, análise exploratória e visualização de dados.
