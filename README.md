# Análise de Emprego e Estabelecimentos (RAIS) em Aglomerações Urbanas do Estado de São Paulo

## Sobre o projeto

Este projeto organiza dados da **Relação Anual de Informações Sociais (RAIS)** em arquivos CSV para estudar os vínculos empregatícios formais e os estabelecimentos de quatro aglomerações urbanas do estado de São Paulo.

Os dados são consultados na **Base dos Dados**, que disponibiliza tabelas da RAIS no BigQuery. Os notebooks são executados no Google Colab e salvam os arquivos resultantes em uma pasta do Google Drive.

A RAIS cobre empregos formais registrados. Ela não representa todo o mercado de trabalho: ocupações informais e outras relações não registradas podem não aparecer nos dados.

## Áreas estudadas

Cada aglomeração reúne os municípios definidos no código. Os totais por aglomeração correspondem à soma dos municípios que a compõem.

| Aglomeração | Municípios incluídos |
|---|---|
| Bauru | Bauru, Piratininga |
| São José do Rio Preto | Bady Bassitt, Cedral, Guapiaçu, Mirassol, São José do Rio Preto |
| São José dos Campos | Caçapava, Jacareí, São José dos Campos |
| Sorocaba | Alumínio, Araçoiaba da Serra, Sorocaba, Votorantim |

A coluna `id_municipio` permite identificar cada município individualmente. Os códigos municipais são obtidos no diretório de municípios da Base dos Dados, filtrando o estado de São Paulo.

## Conceitos importantes

- **Vínculo empregatício:** uma relação formal de trabalho. Uma pessoa pode ter mais de um vínculo.
- **Vínculos ativos em 31 de dezembro:** o código de vínculos configura `SOMENTE_ATIVOS_3112 = True` e, quando a coluna correspondente existe na tabela, filtra os vínculos ativos em 31 de dezembro.
- **Estabelecimento:** unidade registrada na RAIS. Um mesmo negócio pode ter mais de um estabelecimento.
- **CNAE:** Classificação Nacional de Atividades Econômicas. O código procura usar a CNAE 2, em nível de subclasse, com sete dígitos.
- **Descrição detalhada:** descrição específica da subclasse CNAE.
- **Descrição da seção:** classificação ampla da atividade, derivada da divisão do código CNAE em uma das seções de A a U. A coluna é acrescentada pelo script de atualização de descrições.
- **Tamanho do estabelecimento:** faixa de tamanho codificada na RAIS. Na tabela agrupada de vínculos, os códigos reconhecidos são apresentados como faixas textuais.
- **Setores econômicos:** para o resumo de vínculos, as divisões CNAE são agrupadas da seguinte forma:
  - **Primário:** divisões 01–03.
  - **Secundário:** divisões 05–43.
  - **Terciário:** divisões 45–99.
  - **Não classificado:** vínculos sem CNAE que possa ser classificada em um dos grupos anteriores.

## Como os códigos funcionam

Os arquivos enviados contêm dois scripts principais, executados no Google Colab, além de etapas de tratamento incluídas nos notebooks.

### 1. Vínculos empregatícios

O código de vínculos:

1. Instala as bibliotecas necessárias e autentica o acesso ao Google Cloud e ao Google Drive.
2. Usa o `PROJECT_ID` configurado para acessar o BigQuery.
3. Localiza os códigos IBGE dos 14 municípios definidos no dicionário `APS`.
4. Consulta quais anos estão disponíveis na tabela `basedosdados.br_me_rais.microdados_vinculos`.
5. Consulta os anos solicitados que estiverem disponíveis: **2000, 2010, 2014, 2019, 2022 e 2025**.
6. Quando a coluna correspondente existe, filtra vínculos ativos em 31 de dezembro.
7. Usa a CNAE 2 quando ela está disponível. Para registros sem CNAE 2, tenta converter o CNAE antigo por meio do código CNAE 2 mais frequente associado a cada CNAE antigo nos dados de 2008 a 2010.
8. Acrescenta a descrição detalhada da CNAE e indica o método usado na coluna `metodo_cnae2`.
9. Salva um CSV de vínculos por aglomeração.
10. Gera também um resumo por setor e aglomeração e uma tabela agrupada com a contagem de vínculos por município, ano, atividade e tamanho do estabelecimento.

Na tabela individual, cada linha representa um vínculo extraído da RAIS. Na tabela agrupada, cada linha representa uma combinação de município, ano, CNAE e faixa de tamanho, acompanhada da contagem de vínculos.

### 2. Estabelecimentos

O código de estabelecimentos:

1. Instala as bibliotecas necessárias e autentica o acesso ao Google Cloud e ao Google Drive.
2. Consulta os estabelecimentos dos municípios definidos em `APS`, nos anos **2014, 2019, 2022 e 2025** que estiverem disponíveis.
3. Recupera informações como município, quantidade de vínculos ativos, tamanho do estabelecimento, tipo, CNAE e CEP.
4. Mapeia os códigos numéricos do tipo de estabelecimento para `CNPJ` ou `CPF`, conforme o dicionário configurado no script.
5. Padroniza o CNAE 2 e, quando necessário, tenta converter o CNAE antigo usando as associações mais frequentes observadas entre 2008 e 2010.
6. Consulta serviços de geocodificação por CEP para obter latitude e longitude. O script tenta primeiro a AwesomeAPI e depois a BrasilAPI.
7. Guarda as coordenadas no arquivo `cache_cep_latlon.json`. O cache permite reaproveitar coordenadas já consultadas se a execução for interrompida.
8. Salva um CSV por aglomeração.

A geocodificação pode demorar, dependendo da quantidade de CEPs e da disponibilidade dos serviços externos. CEPs sem coordenadas encontradas podem ficar sem latitude e longitude.

### 3. Atualização das descrições da CNAE

O script de atualização percorre os CSVs de vínculos e estabelecimentos, renomeia `descricao_cnae` para `descricao_cnae_detalhada` quando necessário e acrescenta `descricao_cnae_secao`.

**Execute essa etapa depois de gerar os CSVs principais.** Se os CSVs forem gerados novamente, a coluna de seção pode ser sobrescrita ou removida; nesse caso, execute a atualização outra vez.

## Arquivos gerados

Os arquivos são salvos na pasta indicada pela variável `PASTA`. No código fornecido, o valor inicial é:

```python
PASTA = "/content/drive/MyDrive/mc/rais/tabelas"
```

É possível alterar esse caminho para outra pasta do Google Drive.

| Arquivo | Conteúdo |
|---|---|
| `vinculos_bauru.csv` | Vínculos da aglomeração de Bauru |
| `vinculos_sao_jose_do_rio_preto.csv` | Vínculos da aglomeração de São José do Rio Preto |
| `vinculos_sao_jose_dos_campos.csv` | Vínculos da aglomeração de São José dos Campos |
| `vinculos_sorocaba.csv` | Vínculos da aglomeração de Sorocaba |
| `vinculos_setores_por_ap.csv` | Contagem e participação percentual dos vínculos por aglomeração, ano e setor |
| `vinculos_agrupado_municipios.csv` | Contagem de vínculos por município, ano, atividade CNAE e tamanho do estabelecimento |
| `estabelecimentos_bauru.csv` | Estabelecimentos da aglomeração de Bauru |
| `estabelecimentos_sao_jose_do_rio_preto.csv` | Estabelecimentos da aglomeração de São José do Rio Preto |
| `estabelecimentos_sao_jose_dos_campos.csv` | Estabelecimentos da aglomeração de São José dos Campos |
| `estabelecimentos_sorocaba.csv` | Estabelecimentos da aglomeração de Sorocaba |
| `cache_cep_latlon.json` | Cache interno das coordenadas consultadas por CEP; não é uma tabela analítica |

Os CSVs usam **ponto e vírgula (`;`)** como separador e codificação UTF-8 com BOM (`utf-8-sig`), para facilitar a abertura em programas de planilha.

## Estrutura das tabelas

### Tabelas de vínculos (`vinculos_*.csv`)

Cada linha representa um vínculo empregatício retornado pela consulta.

| Coluna | Significado |
|---|---|
| `ap` | Aglomeração urbana |
| `ano` | Ano da RAIS |
| `id_municipio` | Código IBGE do município |
| `nome_municipio` | Nome do município |
| `codigo_cnae2` | Código CNAE 2 usado no tratamento |
| `descricao_cnae_detalhada` | Descrição específica da atividade; é preenchida/renomeada na etapa de atualização |
| `descricao_cnae_secao` | Descrição ampla da seção CNAE; é acrescentada na etapa de atualização |
| `tamanho_estabelecimento` | Código/faixa de tamanho disponível na tabela da RAIS |
| `cnae_1_original` | CNAE antigo quando usado como origem da conversão |
| `metodo_cnae2` | Método usado para obter a CNAE 2 |

Para contar os vínculos, conte as linhas da tabela, respeitando os filtros de município, ano ou atividade desejados. O código consulta apenas os municípios e anos disponíveis configurados e, quando possível, filtra vínculos ativos em 31 de dezembro.

### Resumo por setor (`vinculos_setores_por_ap.csv`)

Cada linha representa uma combinação de aglomeração, ano e setor.

| Coluna | Significado |
|---|---|
| `ap` | Aglomeração urbana |
| `ano` | Ano da RAIS |
| `setor` | Primário, Secundário, Terciário ou, quando houver, Não classificado |
| `total_vinculos` | Número de vínculos no setor |
| `total_vinculos_ap_ano` | Total de vínculos da aglomeração naquele ano, somando os setores |
| `percentual` | Participação percentual do setor no total da aglomeração e do ano |

O resumo é calculado a partir da tabela de vínculos em memória, sem uma nova consulta à RAIS. As categorias são definidas pelas divisões do código CNAE: 01–03 para Primário, 05–43 para Secundário e 45–99 para Terciário. A categoria “Não classificado” é mantida quando há vínculos sem classificação.

### Tabela agrupada (`vinculos_agrupado_municipios.csv`)

Cada linha representa uma combinação de município, ano, CNAE 2 e faixa de tamanho do estabelecimento. O arquivo contém a contagem de vínculos pronta, sem listar individualmente cada vínculo.

| Coluna | Significado |
|---|---|
| `ano` | Ano da RAIS |
| `nome_municipio` | Nome do município |
| `id_municipio` | Código IBGE do município |
| `codigo_cnae2` | Código CNAE 2 |
| `descricao_cnae_secao` | Descrição ampla da seção CNAE |
| `descricao_cnae_detalhada` | Descrição específica da atividade |
| `tamanho_estabelecimento` | Faixa de tamanho do estabelecimento em texto |
| `total_vinculos` | Número de vínculos naquela combinação |

O script traduz os códigos de tamanho reconhecidos para estas faixas:

| Código | Faixa |
|---:|---|
| 1 | Zero |
| 2 | Até 4 |
| 3 | De 5 a 9 |
| 4 | De 10 a 19 |
| 5 | De 20 a 49 |
| 6 | De 50 a 99 |
| 7 | De 100 a 249 |
| 8 | De 250 a 499 |
| 9 | De 500 a 999 |
| 10 | 1000 ou mais |

O mesmo CNAE pode aparecer em várias linhas do mesmo município e ano, pois cada faixa de tamanho forma uma combinação diferente. Para obter o total de uma atividade sem separar por tamanho, some `total_vinculos` para os códigos e filtros desejados.

A célula que gera essa tabela compara a soma de `total_vinculos` com o número de linhas da tabela de vínculos em memória. Essa verificação ajuda a detectar perda de registros durante o agrupamento. Linhas sem CNAE ou sem tamanho são mantidas no agrupamento (`dropna=False`), para que os vínculos não sejam descartados.

### Tabelas de estabelecimentos (`estabelecimentos_*.csv`)

Cada linha representa um registro de estabelecimento consultado na RAIS.

| Coluna | Significado |
|---|---|
| `ap` | Aglomeração urbana |
| `ano` | Ano da RAIS |
| `sigla_uf` | Sigla da unidade federativa |
| `estado` | Nome do estado |
| `cidade` | Município do estabelecimento |
| `id_municipio` | Código IBGE do município |
| `quantidade` | Quantidade de vínculos informada para o estabelecimento |
| `tamanho_estabelecimento` | Faixa de tamanho informada pela RAIS |
| `tipo` | Tipo identificado pelo mapeamento configurado no código (`CNPJ`, `CPF` ou `não identificado`) |
| `cnae_2` | Código CNAE 2 utilizado |
| `descricao_cnae_detalhada` | Descrição específica da atividade; é renomeada na etapa de atualização |
| `descricao_cnae_secao` | Descrição ampla da seção CNAE; é acrescentada na etapa de atualização |
| `cnae_1_original` | CNAE antigo quando usado como origem da conversão |
| `metodo_cnae2` | Método usado para obter a CNAE 2 |
| `cep` | CEP do estabelecimento |
| `latitude` / `longitude` | Coordenadas aproximadas obtidas por CEP |

O script usa o mapeamento de tipo `{"1": "CNPJ", "2": "CPF", "3": "CPF"}`. **Esse mapeamento precisa ser conferido no dicionário oficial da RAIS/Base dos Dados antes de ser usado como interpretação definitiva.**

### Valores de `metodo_cnae2`

- `original`: a CNAE 2 veio diretamente da RAIS.
- `convertido_de_cnae1`: a CNAE 2 foi estimada a partir do CNAE antigo, usando a associação mais frequente nos dados de 2008 a 2010.
- `sem_correspondencia`: não foi possível obter uma CNAE 2 pelo método disponível.

## Limitações e cuidados

1. **Cobertura do mercado de trabalho:** os dados se referem aos registros formais cobertos pela RAIS e não devem ser interpretados como medida de todo o emprego existente.
2. **Disponibilidade dos anos:** o código verifica quais anos existem na tabela e avisa quando algum ano solicitado não está disponível. A ausência de um ano no CSV pode significar que ele ainda não foi publicado na fonte consultada.
3. **Conversão do CNAE antigo:** a conversão para CNAE 2 é aproximada. O código escolhe a CNAE 2 mais frequente associada a cada código antigo nos dados de 2008 a 2010; isso não garante que a atividade original de cada registro tenha sido recuperada corretamente.
4. **Coordenadas por CEP:** latitude e longitude representam uma localização aproximada associada ao CEP, não necessariamente o endereço exato do estabelecimento. Estabelecimentos com o mesmo CEP podem receber as mesmas coordenadas, e alguns CEPs podem não ser geocodificados.
5. **Cache de coordenadas:** `cache_cep_latlon.json` guarda resultados para evitar repetir consultas. Se a execução for interrompida, o arquivo permite reaproveitar os resultados já salvos. Não é necessário editá-lo manualmente.
6. **Tipo CPF/CNPJ:** a correspondência entre os códigos numéricos e os rótulos foi definida no código e precisa ser verificada no dicionário da RAIS.
7. **Descrições ausentes:** códigos CNAE sem correspondência no diretório consultado podem ficar sem descrição detalhada ou sem descrição de seção.
8. **Faixas de tamanho:** a tabela agrupada traduz apenas os códigos listados no dicionário `FAIXAS`. O código avisa quando encontra códigos numéricos não mapeados; valores ausentes continuam sem faixa.
9. **Definição de setores:** o resumo usa os intervalos de divisões CNAE descritos neste README. Se a pesquisa adotar outra definição de setor econômico, será necessário ajustar a função `setor`.
10. **Tamanho dos arquivos:** as tabelas individuais de vínculos podem ser grandes, pois cada linha representa um vínculo. Para análise de grandes volumes, prefira ferramentas que suportem arquivos extensos, como Python, R ou bancos de dados, em vez de depender apenas de planilhas.

## Como executar

1. Abra o notebook de vínculos ou o notebook de estabelecimentos no Google Colab.
2. Confirme o `PROJECT_ID` do projeto Google Cloud que será usado para consultar o BigQuery.
3. Confirme o caminho definido em `PASTA` e se o Google Drive está montado.
4. Execute as células na ordem em que aparecem no notebook.
5. Verifique as mensagens sobre anos disponíveis e ausentes.
6. Aguarde a consulta e a geração dos CSVs. No notebook de estabelecimentos, a geocodificação por CEP pode aumentar o tempo de execução.
7. Depois de gerar ou regenerar as tabelas principais, execute a etapa de atualização das descrições da CNAE para acrescentar `descricao_cnae_secao` e padronizar o nome `descricao_cnae_detalhada`.
8. Confira as mensagens de saída, especialmente as contagens de linhas, os códigos de tamanho sem correspondência e a verificação do total da tabela agrupada.

### Dependências

Os notebooks instalam as bibliotecas necessárias por meio de comandos `pip`, incluindo `google-cloud-bigquery`, `db-dtypes` e `pandas`. O notebook de estabelecimentos também usa `requests` e `tqdm`.

É necessário autenticar uma conta Google com acesso ao projeto do BigQuery e ao Google Drive usado para salvar os arquivos. O acesso ao BigQuery pode estar sujeito às permissões e às condições de uso da Base dos Dados e do Google Cloud.

## Fontes

- **RAIS:** Ministério do Trabalho e Emprego.
- **Dados e tabelas consultadas:** [Base dos Dados](https://basedosdados.org/), incluindo as tabelas `basedosdados.br_me_rais.microdados_vinculos`, `basedosdados.br_me_rais.microdados_estabelecimentos` e diretórios de municípios, CNAE 2 e CEP.
- **Classificação econômica:** Classificação Nacional de Atividades Econômicas (CNAE).
