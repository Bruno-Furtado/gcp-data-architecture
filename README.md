# GCP Data Architecture

Documento que apresenta uma arquitetura de dados, baseada em GCP, para disponibilizar informações recebidas de diversas fontes de forma eficiente, tanto para consumo rápido quanto para uso analítico.

A solução foi pensada para operar em larga escala, com alto volume de dados, mas pode ser simplificada para cenários com menor carga ou complexidade.

<p align="center">
  <img src="https://raw.githubusercontent.com/Bruno-Furtado/gcp-data-architecture/refs/heads/main/architecture.webp" alt="architecture">
</p>

## 1. Envio de dados 🏌️‍♂️

Três são as fontes externas que escrevem dados na solução:

- **Mobile App**: utilizado pelo motorista (envia e consome dados)
- **Backoffice**: utilizado por supervisores (envia e consome dados)
- **Other**: integrações ou scripts (apenas enviam dados)

Todas as fontes se comunicam com a API hospedada em [**Cloud Run**](https://cloud.google.com/run), via chamadas HTTP seguindo o padrão REST, utilizando o método adequado para cada tipo de operação:

| Método   | Objetivo                 | Exemplo               |
| -------- | ------------------------ | --------------------- |
| `POST`   | Inserção de registros    | `POST /motoristas`    |
| `PUT`    | Atualização de registros | `PUT /viagens/123`    |
| `DELETE` | Remoção de registros     | `DELETE /clientes/12` |

```http
PUT /viagens/123
Content-Type: application/json

{
  "status": "concluída",
  "data_fim": "2025-04-03T18:00:00Z"
}
```

### Metadados

Cada requisição carrega metadados, que serão usados nas etapas seguintes:

| Metadado    | Origem          | Utilidade                                              |
| ----------- | --------------- | ------------------------------------------------------ |
| `timestamp` | gerado pela API | Para ordenação e auditoria                             |
| `auditid`   | gerado pela API | Id único da requisição, usado no ciclo de vida do dado |
| `source`    | header ou token | Identifica a origem (ex: app, backoffice)              |
| `operation` | método HTTP     | Define se é insert/update/delete                       |
| `entity`    | path da URL     | Define a tabela/entidade                               |
| `userid`    | path da URL     | Identifica quem gerou o registro                       |

Ao receber uma requisição de qualquer uma das fontes externas, a API em **Cloud Run** realiza duas ações:

- **Log da requisição**: o conteúdo recebido, os headers, o status HTTP e o tempo de resposta são registrados automaticamente no [**Cloud Logging**](https://cloud.google.com/logging), garantindo rastreabilidade.

- **Encaminhamento dos dados**: após o processamento inicial, o dado é bifurcado, persistindo em um banco relacional (utilizado para leituras rápidas) e publicando no [**Pub/Sub**](https://cloud.google.com/pubsub) (onde será processado de forma assíncrona).

> Os logs gerados podem ser analisados via [**Log Analytics**](https://cloud.google.com/logging/docs/log-analytics), possibilitando a criação de alertas e dashboards que ajudam a identificar comportamentos inesperados.



## 2. Persistência no PostgreSQL 💾

O dado é processado de acordo com a regra de negócio e salvo no banco relacional PostgreSQL em instância primária, fazendo uso do [**Cloud SQL**](https://cloud.google.com/sql).

Consultas operacionais e analíticas são realizadas por meio de uma instância de réplica, que serve de apoio para relatórios que exigem menor latência, sem sobrecarregar a instância primária.

Caso ocorra algum erro na escrita, a transação é revertida e a resposta HTTP reflete a falha.



## 3. Publicação no Pub/Sub 📬

Após o recebimento da requisição, a API publica um evento no tópico do [**Pub/Sub**](https://cloud.google.com/pubsub) e a mensagem é entregue ao pipeline do **Dataflow**. O **Pub/Sub** garante a entrega da mensagem por meio de tentativas automáticas:
- Cada mensagem pode ser reenviada até máximo de tentativas (ex: 5 vezes)
- O consumidor deve confirmar o recebimento com um `ack`
- Se falhar após as tentativas, a mensagem é enviada para o [**DLQ**](https://cloud.google.com/pubsub/docs/handling-failures)

### Mensagens na DLQ

A **DLQ** é implementada como um segundo tópico **Pub/Sub**, recebendo mensagens que excederam o máximo de tentativas para subscription principal:
- O tópico da **DLQ** é integrado ao **BigQuery**, utilizando a [funcionalidade nativa de ingestão](https://cloud.google.com/pubsub/docs/bigquery) via **Pub/Sub**.
- Toda mensagem publicada na **DLQ** é inserida em uma tabela, sem necessidade de pipeline adicional.

```json
{
  "timestamp": "2025-04-03T19:15:32.123Z",
  "auditid": "req-123abc",
  "userid": "abc321",
  "source": "backoffice",
  "operation": "update",
  "entity": "viagens",
  "id": "abc123",
  "data": {
    "id": "abc123",
    "status": "concluída",
    "data_fim": "2025-04-03T18:00:00Z"
  },
  "messageid": "abc123",
  "publishtime": "2025-04-03T19:15:33.543Z"
}
```



## 4. Processamento com Dataflow 🚴‍♂️

O pipeline do [**Dataflow**](https://cloud.google.com/products/dataflow) é responsável por consumir os eventos publicados no tópico do Pub/Sub e gravá-los no BigQuery da forma mais simples possível.

Como o fluxo foi projetado para operar em **modo batch**, o Dataflow agrupa eventos em janelas de tempo configuráveis (ex: a cada 5 minutos), garantindo redução de custos:
- Leitura direta do evento JSON recebido do **Pub/Sub**
- Separação e roteamento dos eventos de acordo com a entidade (ex: viagens, motoristas, clientes)
- Persistência em tabelas separadas dentro do dataset `raw`, uma para cada entidade
- Encaminhamento de falhas internas do **Dataflow** (ex: parsing inválido) para tabelas de erro específicas no **BigQuery**

> Desenho com suporte a esquemas flexíveis, utilizando o tipo `data:STRING` no BigQuery. Isso permite armazenar os dados recebidos independentemente de mudanças na estrutura do payload.



## 5. Armazenamento no BigQuery 🗃️

### Dataset `raw`

No [**BigQuery**](https://cloud.google.com/bigquery), os eventos processados pelo **Dataflow** são armazenados em um dataset chamado `raw`. Neste dataset, cada entidade possui sua própria tabela. Essa abordagem traz maior organização, performance e facilidade de evolução do schema de forma independente.

- `raw.viagens`
- `raw.motoristas`
- `raw.clientes`
- `raw_errors.viagens` (para eventos malformados)

#### Estrutura esperada para tabelas no `raw`

| Coluna              | Tipo        | Descrição                                          |
| ------------------- | ----------- | -------------------------------------------------- |
| `timestamp`         | TIMESTAMP   | Momento em que o dado foi gerado                   |
| `auditid`           | STRING      | ID único da requisição                             |
| `userid`            | STRING      | ID único do usuário                                |
| `operation`         | STRING      | Tipo de operação: insert, update ou delete         |
| `source`            | STRING      | Origem da requisição (app, backoffice etc)         |
| `id`                | STRING      | Identificador único do registro                    |
| `data`              | STRING      | Payload original enviado                           |
| `messageid`         | STRING      | ID único da mensagem no Pub/Sub                    |
| `publishtime`       | TIMESTAMP   | Momento em que a mensagem foi publicada no Pub/Sub |
| `ingestiondatetime` | TIMESTAMP   | Momento em que o dado foi gravado no BigQuery      |

> As tabelas são particionadas por data, no caso `DATE(timestamp)` e podem ser clusterizadas por `id` e `operation` para melhorar o desempenho nas consultas.
>
> Eventos inválidos ou malformados são armazenados em tabelas espelhadas no dataset `raw_errors`, com a mesma estrutura da `raw`.



## 6. Transformações com Dataform 🧞‍♂️

O [**Dataform**](https://cloud.google.com/dataform) é responsável por organizar os dados armazenados nas tabelas do dataset `raw`, aplicando regras de negócio, validações adicionais e modelagens necessárias para disponibilizar os dados prontos para análise.

### Staging

Nesta etapa, cada tabela do dataset `raw` possui sua respectiva tabela no dataset `staging`. Essas tabelas são atualizadas com base em transformações agendadas, geralmente via DAGs ou jobs do próprio Dataform.

As tabelas realizam:
- Leitura dos dados brutos (`raw.*`)
- Extração de campos do JSON `data`
- Padronização de nomes e formatos
- Remoção de duplicidades
- Filtro de registros inválidos ou desnecessários

#### `staging.viagens`

| Coluna     | Tipo      | Descrição                         |
| ---------- | --------- | --------------------------------- |
| `id`       | STRING    | Identificador único da viagem     |
| `status`   | STRING    | Status atual da viagem            |
| `data_fim` | TIMESTAMP | Data e hora de conclusão da viagem |
| `lasttimestamp` | TIMESTAMP | Data e hora da última request que gerou o registro |
| `lastuserid` | STRING | Último usuário que gerou o registro |
| `lastoperation` | TIMESTAMP | Última operação realizada no registro |
| `ingestiondatetime` | TIMESTAMP | Momento em que o dado foi gravado no BigQuery |

```sql
config {
  type: "table",
  partition_by: "DATE(ingestiondatetime)",
  schema: "staging",
  name: "viagens"
}

SELECT
  id,
  MAX_BY(data.status, timestamp) AS status,
  MAX_BY(data.data_fim, timestamp) AS data_fim,
  MAX(timestamp) AS lasttimestamp,
  MAX_BY(userid, timestamp) AS lastuserid,
  MAX_BY(operation, timestamp) AS lastoperation,
  CURRENT_TIMESTAMP() AS ingestiondatetime
FROM raw.viagens
WHERE operation != 'delete'
GROUP BY id
```

> Esta tabela é particionada por `DATE(ingestiondatetime)`, o que permite melhorar a performance de leitura e reduzir os custos nas consultas analíticas.
>
> Como padrão, utilizamos `MAX_BY(campo, timestamp)` para reconstruir o estado completo do registro quando os eventos são parciais. Isso garante que cada campo traga o valor mais recente, mesmo que as atualizações venham em diferentes momentos e com apenas parte dos dados.

### Gold

Representa dados prontos para consumo por ferramentas de BI, como o **Looker Studio**. Os modelos são construídos a partir das tabelas da `staging`, geralmente por meio de jobs agendados no próprio **Dataform**. Essa camada costuma conter dados já enriquecidos, validados e organizados de acordo com a necessidade de visualizações e relatórios de negócio.

#### `gold.problemas_motoristas_24h`

| Coluna                  | Tipo      | Descrição                                        |
|-------------------------|-----------|--------------------------------------------------|
| `motorista_nome`        | STRING    | Nome completo do motorista                       |
| `motorista_telefone`    | STRING    | Telefone de contato do motorista                 |
| `problema_nome`         | STRING    | Tipo de problema identificado                    |
| `problema_oficina`      | STRING    | Nome da oficina associada ao problema            |
| `problema_oficina_contato` | STRING | Contato da oficina para encaminhamento           |
| `problema_horario`      | TIMESTAMP | Horário em que o problema foi registrado         |

```sql
config {
  type: "table",
  schema: "gold",
  name: "problemas_motoristas_24h",
  partition_by: "DATE(horario)"
}

WITH problemas_motoristas AS (
  SELECT
    id_problema,
    id_motorista,
    horario
  FROM staging.problemas_motoristas
  WHERE horario >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
)

SELECT
  m.nome AS motorista_nome,
  m.telefone AS motorista_telefone,
  p.nome AS problema_nome,
  p.oficina AS problema_oficina,
  p.oficina_contato AS problema_oficina_contato,
  pm.horario AS problema_horario
FROM problemas_motoristas AS pm
LEFT JOIN staging.motoristas AS m ON pm.id_motorista = m.id
LEFT JOIN staging.problemas AS p ON pm.id_problema = p.id
```
> Tabela re-gerada uma vez ao dia, mas poderiamos trabalhar com casos de carga incremental afim de ter o histórico dos dados (estratégia de carga a partir do max counter da última ingestão).



## 7. Consumo dos dados 🍫

Após as transformações e modelagens nas camadas `staging` e `gold`, os dados ficam disponíveis para consumo por diferentes canais, de acordo com a necessidade da aplicação ou do time de negócio.

### Looker Studio

O [**Looker Studio**](https://lookerstudio.google.com/) pode se conectar tanto ao **BigQuery** quanto ao **PostgreSQL** (réplica). A escolha da fonte depende do tipo de dado e da necessidade de latência:
- Quando conectado ao **BigQuery**, pode-se fazer uso de mecanismos de cache para evitar a criação de novos jobs a cada acesso e diminuir o tempo de resposta.
- Quando conectado ao **PostgreSQL** (réplica), o **Looker Studio** permite consultas com menor latência, ideais para dados operacionais ou dashboards em tempo quase real.

### Processos analíticos e modelos de IA

Os dados da camada `staging` e `gold` também podem ser utilizados como insumo para:
- Treinamento de modelos de machine learning
- Execução de rotinas batch analíticas
- Previsões operacionais



## Notas ✍️

### Custos

- Embora não seja uma ferramenta de fácil uso, a Google disponibiliza uma [calculadora](https://cloud.google.com/products/calculator) para auxiliar com relação aos custos. Também é possível mensurar consultando a documentação de cada um dos recursos utilizados.

### Sofisticação

- É possível criar uma **Cloud Run** e conecta-la ao **BigQuery** para chama-la durante a execução de uma query ([mais detalhes](https://cloud.google.com/bigquery/docs/connections-api-intro)), possibilitando por exemplo, informar os motoristas por meio de alertas.

- É possível criar uma conexão federada no **BigQuery** para que ele acesse o banco relacional ([mais detalhes](https://cloud.google.com/bigquery/docs/federated-queries-intro)), possibilitando enriquecer os relatórios.

### Adaptações

- Na etapa 2, caso não haja uma frequência de envio de dados muito alta, o uso da replica no Cloud SQL pode ser deixado de lado.

- As etapas 3 e 4 podem ser removidas, sendo possível enviar dados por meio do [**BigQuery Write API**](https://cloud.google.com/bigquery/docs/write-api) pela **Cloud Run**.

- A etapa 4 pode ser removida, enviando dados diretamente do **Pub/Sub** para o **BigQuery** ([mais detalhes](https://cloud.google.com/pubsub/docs/bigquery)).

- Caso o **Dataform** seja uma opção muito robusta, ele pode ser substituido por [**Scheduled Queries**](https://cloud.google.com/bigquery/docs/scheduling-queries).

### Melhorias

- Arquitetura proposta com fins didáticos, passível de customizações conforme o contexto e o volume de dados de cada aplicação.

---

<p align="center">Made with ❤️ in Curitiba 🌳 ☔️</p>
