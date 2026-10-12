# Decisões técnicas e aprendizados

Documento complementar ao [README](README.md). Registra por que cada escolha foi feita e o que deu errado no caminho.

## 1. Decisões de arquitetura

| Decisão                                                        | Alternativa considerada               | Motivo                                                                                                                                          |
|----------------------------------------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
| **ELT** em vez de ETL                                          | Transformar antes de carregar         | Carrega-se o dado bruto no Bronze e transforma-se dentro do Databricks, mantendo o original disponível para reprocessamento                     |
| **ADLS Gen2** em vez de Blob Storage comum                     | Blob Storage                          | O namespace hierárquico dá suporte real a diretórios e é o padrão para data lakes com Delta                                                     |
| **Delta Lake** nas camadas Silver e Gold                       | CSV ou Parquet puro                   | Delta adiciona controle transacional (`_delta_log`) sobre Parquet                                                                               |
| **Compute serverless**                                         | Cluster manual                        | A conta Azure for Students tem quotas de VM que impediam criar clusters                                                                         |
| **Unity Catalog** com External Locations                       | Acesso direto por chave               | Governança centralizada e acesso ao ADLS por Access Connector, sem chaves no código                                                             |
| **Uma External Location por container** (bronze, silver, gold) | Uma única location na raiz do storage | Cada camada tem seu próprio caminho `abfss://` e pode ter permissões separadas, todas usando a mesma Storage Credential (`cred-adls-medallion`) |
| **ADF para orquestrar**                                        | Agendar só no Databricks              | Separa ingestão (ADF) de transformação (Databricks) e mostra integração entre serviços                                                          |
| **Job com 2 tarefas encadeadas**                               | Um notebook único                     | Cada etapa tem seu status, tempo e ponto de falha próprios                                                                                      |

## 2. Camadas Medallion

- **Bronze:** cópia fiel da fonte. Não se altera.
- **Silver:** dado confiável. Nulos tratados, duplicatas removidas, nomes em `snake_case`, textos padronizados.
- **Gold:** dado de negócio, já agregado. Três tabelas (mensal, por produto, por cliente).

## 3. Problemas e como foram resolvidos

### 3.1 Job lento no Databricks (12 min contra 1m30s)

**Sintoma:** o Job levava de 5 a 13 minutos. A leitura do CSV (3 MB) levava 3m39s a 4m20s dentro do Job. O mesmo notebook, rodado à mão, levava 2 s nessa célula.

**Investigação (hipóteses testadas):**
1. *Mais arquivos no Bronze?* Não: só um `sales.csv` de 3,16 MiB.
2. *Aviso de versões de ambiente diferentes entre as tarefas?* Não explicou a lentidão.
3. *Código ou acesso ao storage lentos?* Não: notebook manual rodou a leitura em 2 s, e uma célula `spark.range(10).count()` levou 1 s.
4. *Modo de execução do Job:* o modo normal do serverless tem tempo de inicialização parecido com o de infraestrutura clássica; o modo de *desempenho otimizado* tem partida mais rápida e confiável.

**Resultado:** com o desempenho otimizado ativado, o Job passou a levar 1m16s (48 s + 27,8 s). O tempo perdido estava na inicialização da computação, e não no código.

**Execução de referência (final):** `bronze_to_silver` em 26,4 s e `silver_to_gold` em 24,6 s, somando ~51 s. O ADF registrou 1m18s para a atividade do Job, porque conta também o disparo e a partida da computação. A diferença de ~27 s é overhead de inicialização, e não código.

**Lição:** quando o mesmo código é rápido manualmente e lento em Job, o gargalo costuma estar na computação, e não na lógica. Medir uma célula trivial separa as duas coisas.

### 3.2 ADF sem autorização no Databricks (erro 9512)

**Sintoma:** a identidade gerenciada do ADF recebeu "User not authorized".

**Tentativa:** permissão de Contribuidor no recurso do workspace para a identidade gerenciada. Não bastou.

**Solução adotada:** token de acesso pessoal com escopo `jobs` no linked service (validade de 30 dias).

**Limitação:** o token expira; uma solução de longo prazo seria migrar a autenticação ou guardar o token no Key Vault.

### 3.3 Storage Credential apontando para o conector antigo

O workspace foi recriado. Metastore, Storage Credential e External Locations sobrevivem à exclusão do workspace, então a credential continuou apontando para o Access Connector antigo. Foi preciso atualizá-la ("Forçar atualização") com o ID do novo conector e garantir o papel *Storage Blob Data Contributor* no storage.

### 3.4 External Locations com eventos de arquivo

Locations criadas com file events em "Automático" falham no teste de eventos por falta de permissões (EventGrid e filas), mas a leitura e a escrita Delta funcionam. Locations com file events habilitados não podem ser editadas e precisam ser apagadas e recriadas.

### 3.5 Outros

- O tier Standard do Databricks foi descontinuado: novos workspaces exigem Premium.
- A tradução automática do navegador pode alterar visualmente nomes técnicos no portal do Azure.
- Durante testes, vale desligar *novas tentativas* em Jobs para não pagar execuções repetidas.

## 4. Custo

- Conta Azure for Students, com orçamento mensal de US$ 15 e alertas em 25/50/75/90/100%.
- Gasto total em outubro de 2026: US$ 10,72 (≈ R$ 54), cerca de 11% do crédito de US$ 100, incluindo as execuções de depuração.
- Por serviço: Databricks US$ 10,69; Data Factory US$ 0,02; Storage abaixo de US$ 0,01; Bandwidth US$ 0,00.
- O gasto se concentrou em 9 e 10 de outubro, na fase de desenvolvimento e depuração, e ficou estável depois, com o projeto parado.
- O Cost Analysis atrasa horas para mostrar o consumo: o valor subiu de US$ 1,46 para US$ 4,17 e depois para US$ 10,72 sem nenhuma execução nova. Vale conferir de novo 24 a 48 horas depois de rodar algo.
- O alerta de orçamento só avisa; não interrompe recursos.

## 5. Documentação e privacidade

- Os prints do README ficam em `docs/img/`, numerados na ordem em que aparecem.
- Antes de publicar, cada captura foi revisada para não expor informações sensíveis 
como ID de assinatura ou tokens. Telas que mostram o proprietário de recursos (como as External Locations) foram recortadas antes da coluna do e-mail, e na tela do gráfico do Job os caminhos dos notebooks, que continham parte do e-mail, foram borrados.

## 6. O que eu faria diferente

- Ativar o desempenho otimizado desde o início, em vez de desligá-lo para "economizar" nos testes.
- Medir o tempo por célula já na primeira execução do Job.
- Usar Key Vault para o token desde o começo.
