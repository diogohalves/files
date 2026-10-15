# Configuração do contrato Chameleon — `dataflowtemplate-dbrh-base-ing-degreed`

## Como funciona

O componente ingere **um endpoint por execução**. Para cada endpoint que se deseja
trazer, existe um contrato (YAML) próprio, com o nome do endpoint no parâmetro
`endpoint`. O job autentica na API do Degreed via OAuth, consome o endpoint e grava
o retorno **cru** em Avro no caminho informado em `prefixOut`. A seleção de campos e
a tipagem ficam no `bqSchema` do próprio contrato.

## Parâmetros

| Parâmetro | Obrigatório | Descrição |
|---|---|---|
| `endpoint` | sim | Nome do endpoint a ser ingerido. Ver tabela de endpoints abaixo. |
| `api_path` | sim | Endereço base da API. Valor: `https://api.degreed.com` |
| `oauth_url` | sim | Endereço do endpoint de autenticação. Valor: `https://degreed.com/oauth/token` |
| `scope` | sim | Escopos OAuth necessários para o endpoint, separados por espaço. Ver tabela abaixo. |
| `dias_delta` | sim | Quantidade de dias consultados retroativamente. Ver seção de janela. |
| `secret_manager_project_id` | sim | ID numérico do projeto do Secret Manager. |
| `client_id` | sim | Nome do segredo com o client id da API do Degreed. |
| `client_secret` | sim | Nome do segredo com o client secret da API do Degreed. |
| `prefixOut` | sim | Pasta no GCS onde os arquivos Avro são gravados. |

## Endpoints disponíveis

| `endpoint` | `scope` | Limite de 7 dias |
|---|---|---|
| `users` | `users:read` | não |
| `content` | `content:read` | não |
| `pathways` | `pathways:read` | não |
| `groups` | `groups:read` | não |
| `accomplishments` | `accomplishments:read` | não |
| `certificates` | `certificates:read` | não |
| `organization_skills` | `skills:read` | não |
| `skill_ratings` | `skills:read` | não |
| `user_skills` | `skills:read` | não |
| `logins` | `logins:read` | **sim** |
| `completions` | `completions:read` | **sim** |
| `required_learning` | `required_learning:read` | **sim** |

Um `endpoint` fora dessa lista faz o job falhar na inicialização, de propósito — é
preferível a gravar um arquivo vazio.

## Janela de consulta e `dias_delta`

Três endpoints (`logins`, `completions`, `required_learning`) têm limite imposto pela
própria API do Degreed: **no máximo 7 dias por requisição**. Isso é tratado dentro do
componente, não precisa de configuração.

O `dias_delta` informa quantos dias retroativos trazer, e o pipeline fatia o período
em janelas de 7 dias automaticamente. Exemplos:

- `dias_delta: "7"` → 1 requisição
- `dias_delta: "30"` → 5 requisições encadeadas
- `dias_delta: "90"` → 13 requisições encadeadas

Nos demais endpoints não há filtro de data: a API devolve a base completa e o
`dias_delta` não afeta o resultado (deve ser informado mesmo assim, por ser
obrigatório no template).

**Carga histórica:** não existe parâmetro separado. Um contrato de backfill é o mesmo
contrato com `dias_delta` maior, no mesmo modelo dos arquivos `_hist_N` já usados em
outras ingestões. Quanto maior o valor, mais requisições e maior o tempo de execução —
vale considerar `timeoutMinutos` folgado.

## Bloco `parametros` do contrato

```yaml
parametros:
  endpoint: "users"
  api_path: "https://api.degreed.com"
  oauth_url: "https://degreed.com/oauth/token"
  scope: "users:read"
  dias_delta: "7"
  secret_manager_project_id: "<id do projeto>"
  client_id: "<nome do segredo>"
  client_secret: "<nome do segredo>"
  prefixOut: "#{prefixOut}"
  temp_location: "gs://#{gcsLocation}/tmp/"
additionalExperiments:
  - enable_confidential_compute
```

## Observações para o `bqSchema`

Os campos são gravados **todos como string anulável**, com o nome saneado para o
formato aceito pelo Avro: caracteres inválidos viram `_`. Os campos da API vêm com
hífen, então `employee-id` é gravado como `employee_id`, `completed-at` como
`completed_at`, e assim por diante. O `bqSchema` do contrato deve usar os nomes com
underscore.

Além dos campos da API, cada registro traz dois metadados: `source_endpoint` (nome do
endpoint de origem) e `ingested_at` (timestamp da ingestão).

## Pendência

Os escopos da tabela são os que o componente solicita hoje. Confirmar com o time que
administra as credenciais do Degreed se todos estão liberados — em testes,
`certificates` e `required_learning` retornaram 403, o que é permissão da credencial,
não do componente.


###########

Opa! Queria registrar que fechamos as duas frentes de ingestão: Ludos e Degreed.

No Ludos, a ingestão do SQL Server para o BigQuery saiu via contratos do Chameleon — 7 configs cobrindo as tabelas da plataforma de gamificação no dbeducacao, já na convenção de produção.

No Degreed foi um trabalho mais longo, porque é API externa. Desenvolvi o componente dataflowtemplate-dbrh-base-ing-degreed, um template Dataflow em Apache Beam/Python que roda dentro do Confidential Space, autentica via OAuth com as credenciais no Secret Manager e aterrissa os dados crus em Avro para o Chameleon consumir. São 12 endpoints, com o Chameleon disparando um job por endpoint. Três deles (logins, completions e required-learning) têm limite de 7 dias por consulta na API, então o pipeline fatia o período automaticamente — o que também resolve o backfill sem intervenção manual.

O componente saiu com cobertura de testes unitários dentro do padrão dos demais templates do banco, seguindo a mesma estrutura e as mesmas convenções dos componentes já existentes.

Um ponto que veio dos testes e vale registrar: nas chamadas reais, a credencial do Degreed negou alguns escopos — certificates e required-learning retornaram 403. Não é problema do componente, é permissão do lado da credencial, mas foi justamente o desenvolvimento que deixou isso visível. Fica mapeado para ajuste com quem administra os acessos da API.

Vale dizer que boa parte do tempo nessa segunda frente foi a definição e aprovação da arquitetura — Confidential Space x Apigee, acessos, Secret Manager. Depois que o desenho fechou, o desenvolvimento andou rápido.

Com as duas ingestões em pé, as transformações dbt para a camada semântica seguem o fluxo normal daqui pra frente.

Foi um ciclo longo, e é bom ver as entreguas.
