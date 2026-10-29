# Grupo01-Case-RocketLab

Projeto desenvolvido pelo nosso Grupo 01 no Rocket Lab 2026, a partir do case de atendimento da V-Credit. Organizamos as bases em uma arquitetura medalhão para tratar as inconsistências, integrar as informações e preparar os dados para análise.

O processamento foi desenvolvido no Databricks Free Edition, com Python, PySpark, SQL e tabelas Delta. O período analisado é de janeiro de 2025 a julho de 2026.

## Organização do repositório

| Arquivo ou pasta | Conteúdo |
| --- | --- |
| [00_Landing_Inspecao.ipynb](00_Landing_Inspecao.ipynb) | Preparação e inspeção dos arquivos de entrada. |
| [01_Ingestao_Bronze.ipynb](01_Ingestao_Bronze.ipynb) | Leitura dos CSVs e gravação das oito tabelas Bronze. |
| [02_Bronze_To_Silver.ipynb](02_Bronze_To_Silver.ipynb) | Tratamentos e validações das oito tabelas Silver, reunidos em um notebook. |
| [03_silver_to_gold_bruno.ipynb](03_silver_to_gold_bruno.ipynb) | Construção das cinco dimensões e da fato de chamados. |
| [04_Views.ipynb](04_Views.ipynb) | Views para consulta e consumo dos dados. |
| [06_Teste_Idempotencia.ipynb](06_Teste_Idempotencia.ipynb) | Comparação entre duas execuções completas do pipeline, com o resultado do teste. |
| [jobs/job_vcredit_medallion.yml](jobs/job_vcredit_medallion.yml) | Configuração do Job, dependências e agendamento. |
| [evidencias/](evidencias/) | Capturas do fluxo, do histórico de execuções e do agendamento. |

## Arquitetura

| Etapa | Função |
| --- | --- |
| Landing | Armazenar os CSVs recebidos em um volume do Databricks. |
| Bronze | Preservar os campos de origem como texto e adicionar o horário de ingestão. |
| Silver | Padronizar e tipar os campos, remover repetições exatas nas bases aplicáveis, sinalizar inconsistências e verificar correspondências entre bases. |
| Gold | Consolidar os dados em um modelo dimensional, com uma linha por chamado na fato. |
| Views | Organizar as consultas para consumo e análise. |

A Gold contém `dim_cliente`, `dim_atendente`, `dim_motivo`, `dim_canal`, `dim_data` e `fato_chamados`. Os relacionamentos usam surrogate keys, e custos e pesquisas são agregados por chamado antes dos cruzamentos.

## Preparação do ambiente

Para executar o projeto, é necessário ter acesso a um Workspace do Databricks e permissão para criar e atualizar as tabelas e os controles do teste.

O ambiente utilizado tem:

- Catálogo: `vcredit_medallion`.
- Schemas do pipeline: `bronze`, `silver` e `gold`.
- Volume de entrada: `vcredit_medallion.bronze.landing`.
- Caminho dos arquivos: `/Volumes/vcredit_medallion/bronze/landing`.
- Schema `auditoria`: criado pelo notebook 06 para os controles de comparação.

Antes de executar a Bronze, prepare o catálogo, os três schemas do pipeline e o volume de entrada no Workspace de destino. Importe os notebooks e coloque no volume os oito arquivos abaixo, com estes nomes:

| Arquivo | Codificação usada na Bronze |
| --- | --- |
| `base_atendentes.csv` | UTF-8 |
| `base_motivos.csv` | UTF-8 |
| `canais.csv` | UTF-8 |
| `chamados.csv` | UTF-8 |
| `chamados_hora.csv` | ISO-8859-1 |
| `clientes.csv` | ISO-8859-1 |
| `custos.csv` | UTF-8 |
| `pesquisa_satisfacao.csv` | UTF-8 |

Os arquivos fornecidos com o case devem ser obtidos pelos canais do programa. As codificações são definidas por arquivo para preservar os textos e permitir a conversão dos horários, incluindo o trecho `às`.

## Ordem de execução

1. Use o notebook `00_Landing_Inspecao` para a preparação e inspeção dos arquivos.
2. Execute `01_Ingestao_Bronze`.
3. Execute `02_Bronze_To_Silver`.
4. Execute `03_silver_to_gold_bruno`.
5. Execute `04_Views` para criar ou atualizar as definições das views.
6. Execute o notebook 06 separadamente, seguindo o procedimento de idempotência abaixo.

O Job recorrente executa somente **Bronze → Silver → Gold**. A ingestão dos CSVs nas tabelas já acontece na Bronze. A disponibilização de novos arquivos na Landing é manual; o Job não busca arquivos no computador dos integrantes.

As tabelas são gravadas por `overwrite`, considerando os CSVs como a entrada completa da carga. A solução não implementa ingestão incremental. Na Gold, o recorte continua sendo o período do case: de 01/01/2025 a 31/07/2026.

## Decisões de tratamento

- A Bronze preserva os valores recebidos e adiciona `_ingerido_em` para identificar o horário da carga.
- A Silver mantém campos de origem e sinalizadores para permitir a conferência dos tratamentos.
- Repetições exatas de chamados e custos são removidas; registros conflitantes são verificados separadamente.
- Custos são convertidos para decimal. Valores negativos são preservados e sinalizados; ausência de custo não é tratada como custo zero.
- Notas de satisfação de 1 a 5 são consideradas válidas. Valores ausentes ou fora desse domínio são classificados e não recebem uma nota válida artificial.
- Os tempos usados na Gold vêm dos horários tratados da base `chamados_hora`. Divergências em relação aos tempos informados em `chamados` são sinalizadas na Silver.
- Intervalos com ordem temporal inválida não são usados como tempos válidos.
- A Gold prioriza o identificador de cliente de `chamados_hora`, recorrendo ao de `chamados` quando o primeiro está ausente.
- Os chamados fora do período do case permanecem sinalizados na Silver e são excluídos da fato Gold.

## Resultados das validações

| Verificação | Resultado observado |
| --- | --- |
| Chamados na Bronze | 134.474 registros |
| Chamados na Silver | 131.838 registros, após remover 2.636 repetições exatas |
| Custos na Silver | 130.288 registros, após remover 781 repetições exatas |
| Pesquisas na Silver | 81.021 registros |
| Horários na Silver | 131.838 registros |
| Chamados fora do período | 198 registros |
| Fato de chamados na Gold | 131.640 registros e 131.640 identificadores distintos |
| Chaves da fato nulas ou sem correspondência nas dimensões | Zero nas verificações executadas |
| Teste de idempotência | 22 tabelas aprovadas, sem linhas ausentes ou adicionais |

A soma dos custos passou de R$ 1.455.269,97 na origem para R$ 1.447.671,52 após a deduplicação. A diferença de R$ 7.598,45 corresponde aos lançamentos repetidos removidos. Esse ajuste corrige a informação e não representa uma economia operacional obtida por uma proposta.

Os alertas de qualidade, como notas ausentes, divergências de horários e clientes ou atendentes não identificados, continuam documentados e sinalizados nos notebooks. A aprovação das verificações não significa que todos os campos de origem estavam completos.

## Job e atualização recorrente

O Job tem três tarefas em sequência:

| Tarefa | Notebook |
| --- | --- |
| `01_bronze` | `01_Ingestao_Bronze` |
| `02_silver` | `02_Bronze_To_Silver`, após o sucesso da Bronze |
| `03_gold` | `03_silver_to_gold_bruno`, após o sucesso da Silver |

O agendamento está ativo para execução diária às **08h**, no fuso **America/Sao_Paulo**, com a expressão Quartz `0 0 8 * * ?`.

Foi registrada uma execução iniciada pelo agendador em 29/10/2026, às 08h, concluída com sucesso nas três tarefas. As capturas estão na pasta `evidencias`.

### Ajuste em outro Workspace

A configuração usa `source: WORKSPACE`. Os campos `notebook_path` apontam para os notebooks armazenados no Databricks, e não para pastas do computador ou caminhos do GitHub.

Para recriar o fluxo em outro Workspace:

1. Prepare os arquivos e o ambiente descritos acima e importe os notebooks.
2. Crie o Job com as três tarefas e mantenha as dependências Bronze → Silver → Gold.
3. Em **Tasks**, abra cada tarefa, mantenha **Source = Workspace** e selecione o notebook correspondente no campo **Path**.
4. Salve as tarefas e configure o agendamento com o horário e o fuso desejados.

Os nomes das pastas podem variar entre os ambientes. O YAML registra a configuração utilizada no projeto; alterar somente o arquivo no GitHub não atualiza automaticamente o Job no Databricks.

## Teste de idempotência

O notebook `06_Teste_Idempotencia` compara as oito tabelas Bronze, oito Silver e seis Gold entre duas execuções completas. Ele verifica os hashes SHA-256 dos oito CSVs, os nomes e tipos das colunas, as contagens e o conteúdo dos registros. A comparação usa `exceptAll` nos dois sentidos, considerando também a multiplicidade das linhas.

Para reproduzir:

1. Execute o Job completo e aguarde as três tarefas terminarem com sucesso.
2. Na primeira célula de código do notebook 06, defina `acao = "registrar_referencia"` e execute todas as células.
3. Execute novamente o Job completo, mantendo os mesmos arquivos e códigos.
4. Volte ao notebook 06, defina `acao = "comparar"` e execute todas as células.
5. Confira as 22 linhas do relatório: `aprovado`, `schema_igual` e `versao_avancou` devem ser `true`; as contagens devem ser iguais, e as diferenças devem ser zero.

A referência usa versões anteriores das tabelas Delta. Faça o teste enquanto o histórico e os arquivos necessários ao time travel estiverem disponíveis, sem executar outras cargas durante o registro ou a comparação. Uma nova referência substitui a anterior.

O relatório é salvo em `vcredit_medallion.auditoria.idem_resultados`. O avanço de versão é exigido porque os notebooks atuais gravam todas as tabelas por `overwrite` em cada execução.

### Premissa de auditoria

`_ingerido_em` registra o horário da carga na Bronze e é preservado na Silver. Como esse horário é gerado novamente em cada ingestão, seus valores podem mudar mesmo com os mesmos arquivos.

Por isso, somente os valores de `_ingerido_em` nas camadas Bronze e Silver são desconsiderados na comparação de conteúdo. A presença e o tipo da coluna continuam sendo verificados. Todas as demais colunas são comparadas, e nenhuma coluna da Gold é desconsiderada.

No ensaio realizado, as **22 tabelas foram aprovadas**, sem diferenças nos dados de negócio. O resultado se refere aos arquivos e à versão do código usados no teste. O notebook 06 fica separado do Job recorrente, pois uma atualização válida dos arquivos pode produzir novos resultados.

## Evidências

Na pasta `evidencias`, foram organizadas as capturas do fluxo Bronze → Silver → Gold, do histórico de execuções e do agendamento ativo. O relatório das 22 tabelas aprovadas está incluído nas saídas do notebook 06.

## Uso dos dados

O material do case é confidencial e destinado às atividades do Rocket Lab. O acesso ao repositório e às bases deve ficar restrito às pessoas autorizadas no processo. Na documentação e nas evidências, devem ser usados resultados agregados, preservando os dados pessoais dos clientes e atendentes.
