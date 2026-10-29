# Grupo01-Case-RocketLab

Projeto desenvolvido pelo Grupo 01 no Rocket Lab 2026, a partir do case de atendimento da V-Credit. Organizamos as bases em uma arquitetura medalhão para tratar as inconsistências, integrar as informações e preparar os dados para análise.

O processamento foi desenvolvido no Databricks Free Edition, com Python, PySpark, SQL e tabelas Delta. O período analisado é de janeiro de 2025 a julho de 2026.

## Organização do repositório

| Arquivo ou pasta | Conteúdo |
| --- | --- |
| [00_Landing_Inspecao.ipynb](00_Landing_Inspecao.ipynb) | Preparação e inspeção dos arquivos de entrada. |
| [01_Ingestao_Bronze.ipynb](01_Ingestao_Bronze.ipynb) | Leitura dos CSVs e gravação das oito tabelas Bronze. |
| [02_Bronze_To_Silver.ipynb](02_Bronze_To_Silver.ipynb) | Tratamentos e validações das oito tabelas Silver, reunidos em um notebook. |
| [03_Silver_To_Gold.ipynb](03_Silver_To_Gold.ipynb) | Construção das cinco dimensões e da fato de chamados. |
| [04_Views/04_Views.ipynb](04_Views/04_Views.ipynb) | View que reúne os dados da Gold para o painel e as simulações de custo. |
| [05_Teste_De_Idempotência.ipynb](05_Teste_De_Idempot%C3%AAncia.ipynb) | Comparação entre duas execuções completas do pipeline, com o resultado do teste. |
| [06_Jobs/job_vcredit_medallion.yml](06_Jobs/job_vcredit_medallion.yml) | Configuração do Job, dependências e agendamento. |
| [07_Evidências/](07_Evid%C3%AAncias/) | Capturas do fluxo, do histórico de execuções e do agendamento. |
| [04_Views/V-Credit Central de Atendimento.pdf](04_Views/V-Credit%20Central%20de%20Atendimento.pdf) | Exportação visual do painel de atendimento. |

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
- Schema `auditoria`: criado pelo notebook 05 para os controles de comparação.

Importe os notebooks no Workspace de destino. No `00_Landing_Inspecao`, execute primeiro as células de configuração e criação do catálogo, dos três schemas e do volume. Em seguida, envie os oito CSVs para o volume e execute as células de conferência e inspeção. Os arquivos devem ficar com estes nomes:

| Arquivo | Codificação usada na inspeção e na Bronze |
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
4. Execute `03_Silver_To_Gold`.
5. Execute o notebook `04_Views/04_Views.ipynb` para criar ou atualizar a definição da view.
6. Execute o notebook 05 separadamente, seguindo o procedimento de idempotência abaixo.

O Job recorrente executa somente **Bronze → Silver → Gold**. A preparação e inspeção da Landing, a criação da view e o teste de idempotência são executados separadamente. A ingestão dos CSVs nas tabelas já acontece na Bronze. A disponibilização de novos arquivos na Landing é manual; o Job não busca arquivos no computador dos integrantes.

As tabelas são gravadas por `overwrite`, considerando os CSVs como a entrada completa da carga. A solução não implementa ingestão incremental. Na Gold, o recorte continua sendo o período do case: de 01/01/2025 a 31/07/2026.

## Painel de atendimento

O notebook `04_Views/04_Views.ipynb` cria a view `vcredit_medallion.gold.vw_dashboard_chamados`, que reúne a fato e as dimensões para apoiar o painel e as simulações de custo. O arquivo [04_Views/V-Credit Central de Atendimento.pdf](04_Views/V-Credit%20Central%20de%20Atendimento.pdf) apresenta a versão visual do painel. O PDF não inclui a configuração editável dos gráficos.

As views comuns consultam os dados atuais das tabelas. Sua definição precisa ser executada na preparação do ambiente e quando o código da view mudar. A atualização visual do dashboard depende de uma atualização das consultas do próprio painel; ela não é uma tarefa do Job Bronze → Silver → Gold.

As premissas e limitações das simulações de custo estão documentadas no notebook de views.

## Decisões de tratamento

- A Bronze preserva os valores recebidos e adiciona `_ingerido_em` para identificar o horário da carga.
- A Gold adiciona `_gold_atualizado_em` para registrar o horário de atualização de cada tabela.
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

| Tarefa | Nome do notebook no Workspace utilizado | Arquivo correspondente no repositório |
| --- | --- | --- |
| `01_bronze` | `01_Ingestao_Bronze` | `01_Ingestao_Bronze.ipynb` |
| `02_silver` | `02_Bronze_To_Silver` | `02_Bronze_To_Silver.ipynb` |
| `03_gold` | `03_silver_to_gold_bruno` | `03_Silver_To_Gold.ipynb` |

A Silver depende do sucesso da Bronze, e a Gold depende do sucesso da Silver. O YAML está em [06_Jobs/job_vcredit_medallion.yml](06_Jobs/job_vcredit_medallion.yml).

No ambiente utilizado, os caminhos do YAML apontam para os notebooks da pasta `Case SP Final`. O nome `03_silver_to_gold_bruno` no Workspace pode ser diferente de `03_Silver_To_Gold.ipynb` no GitHub; o importante é que o caminho selecionado no Job exista e contenha a versão final do código. Renomear ou mover um arquivo no GitHub não altera esse caminho automaticamente.

O agendamento está ativo para execução diária às **08h**, no fuso **America/Sao_Paulo**, com a expressão Quartz `0 0 8 * * ?`.

Foi registrada uma execução iniciada pelo agendador em 29/10/2026, às 08h, concluída com sucesso nas três tarefas. As capturas estão na pasta `07_Evidências`.

### Ajuste em outro Workspace

A configuração usa `source: WORKSPACE`. Os campos `notebook_path` apontam para os notebooks armazenados no Databricks, e não para pastas do computador ou caminhos do GitHub.

Para recriar o fluxo em outro Workspace:

1. Prepare os arquivos e o ambiente descritos acima e importe os notebooks.
2. Crie o Job com as três tarefas e mantenha as dependências Bronze → Silver → Gold.
3. Em **Tasks**, abra cada tarefa, mantenha **Source = Workspace** e selecione o notebook correspondente no campo **Path**.
4. Salve as tarefas e configure o agendamento com o horário e o fuso desejados.

Os nomes das pastas podem variar entre os ambientes. O YAML registra a configuração utilizada no projeto; alterar somente o arquivo no GitHub não atualiza automaticamente o Job no Databricks.

## Teste de idempotência

O notebook `05_Teste_De_Idempotência` compara as oito tabelas Bronze, oito Silver e seis Gold entre duas execuções completas. Ele verifica os hashes SHA-256 dos oito CSVs, os nomes e tipos das colunas, as contagens e o conteúdo dos registros. A comparação usa `exceptAll` nos dois sentidos, considerando também a multiplicidade das linhas.

Para reproduzir:

1. Execute o Job completo e aguarde as três tarefas terminarem com sucesso.
2. Na primeira célula de código do notebook 05, defina `acao = "registrar_referencia"` e execute todas as células.
3. Execute novamente o Job completo, mantendo os mesmos arquivos e códigos.
4. Volte ao notebook 05, defina `acao = "comparar"` e execute todas as células.
5. Confira as 22 linhas do relatório: `aprovado`, `schema_igual` e `versao_avancou` devem ser `true`; as contagens devem ser iguais, e as diferenças devem ser zero.

A referência usa versões anteriores das tabelas Delta. Faça o teste enquanto o histórico e os arquivos necessários ao time travel estiverem disponíveis, sem executar outras cargas durante o registro ou a comparação. Uma nova referência substitui a anterior.

O relatório é salvo em `vcredit_medallion.auditoria.idem_resultados`. O avanço de versão é exigido porque os notebooks atuais gravam todas as tabelas por `overwrite` em cada execução.

### Premissa de auditoria

A coluna `_ingerido_em` registra o horário da ingestão na Bronze e é preservada na Silver. Na Gold, `_gold_atualizado_em` registra o horário da atualização. Esses horários podem mudar a cada execução, mesmo quando os arquivos de entrada são os mesmos.

Por isso, o teste desconsidera somente os valores de `_ingerido_em` na Bronze/Silver e de `_gold_atualizado_em` na Gold. A presença e o tipo dessas colunas continuam sendo verificados. Todas as demais colunas são comparadas normalmente, considerando também a quantidade de ocorrências de cada linha.

No ensaio realizado com a premissa atualizada, as **22 tabelas foram aprovadas**, com schemas iguais, versões avançadas, contagens iguais e zero linhas ausentes ou adicionais. Os hashes dos oito CSVs também coincidiram com a referência. O resultado se refere aos arquivos e à versão do código usados no teste. O notebook 05 fica separado do Job recorrente, pois uma atualização válida dos arquivos pode produzir novos resultados.

## Evidências

Na pasta `07_Evidências`, foram organizadas as capturas do fluxo Bronze → Silver → Gold, do histórico de execuções e do agendamento ativo. O relatório das 22 tabelas aprovadas está incluído nas saídas do notebook 05.

## Uso dos dados

O material do case é confidencial e destinado às atividades do Rocket Lab. O acesso ao repositório, ao Workspace e às bases deve ficar restrito às pessoas autorizadas no processo.

Os notebooks de Landing, Bronze e Silver foram disponibilizados sem as saídas das células, para evitar a exposição de nomes, CPFs e e-mails nas amostras. O notebook de idempotência mantém o relatório agregado das verificações.
