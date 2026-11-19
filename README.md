# BFF (Backend for Frontend)

Serviço responsável por centralizar e orquestrar as chamadas aos microsserviços internos (`soat-upload-service` e `soat-fase5-report`), expondo uma API unificada para o cliente.


## Alunos

| Aluno | RM | Discord | LinkedIn |
|---|---|---|---|
| Felipe | 365154 | felipeoli7eira | [@felipeoli7eira](https://www.linkedin.com/in/felipeoli7eira) |
| Nicolas | 365746 | nic_hcm | [@Nicolas Martins](https://www.linkedin.com/in/nicolas-hcm) |
| William | 365973 | wllsistemas | [@William Francisco Leite](https://www.linkedin.com/in/williamfranciscoleite) |

---

## Descrição do Problema

O sistema tem como objetivo automatizar a análise de diagramas de arquitetura de software por meio de Inteligência Artificial. Equipes de engenharia submetem diagramas (JPG, JPEG, PNG ou PDF) e recebem, de forma assíncrona, um relatório com **componentes identificados**, **riscos** e **recomendações** de melhoria — eliminando a necessidade de revisão manual.

---

## Arquitetura Proposta

O sistema é composto por cinco microsserviços interligados:

| Serviço | Papel |
|---|---|
| **BFF** *(este serviço)* | Ponto de entrada unificado; orquestra chamadas aos serviços internos |
| **upload-service** | Recebe diagramas, armazena no Amazon S3, publica na fila `protocols` |
| **trigger-service** | Consome a fila, aciona a IA e persiste resultados no PostgreSQL |
| **report-service** | Consulta resultados e gera relatório em PDF |
| **RabbitMQ** | Broker de mensagens para comunicação assíncrona |

> Diagrama interativo: [FIAP - HACKATON FASE 5](https://www.tldraw.com/f/CPYtIC_xwtcfbSCH4vgkT?d=v567.4.3513.1667.page)

### Repositórios do projeto

| Repositório | O que faz | Stack |
|---|---|---|
| [soat-bff](https://github.com/felipeoli7eira/soat-bff) *(este)* | Porta de entrada do cliente; repassa para upload e relatório | PHP 8.4, Laravel 12, Nginx |
| [soat-upload-service](https://github.com/felipeoli7eira/soat-upload-service) | Valida o diagrama, cria o protocolo, grava no S3 e publica na fila | Laravel, Clean Architecture, PostgreSQL, RabbitMQ |
| [soat-fase5-trigger](https://github.com/nichcm/soat-fase5-trigger) | Consome a fila, chama a IA e guarda o status de cada protocolo | Laravel, PostgreSQL, RabbitMQ com DLQ |
| [soat-fase5-report](https://github.com/nichcm/soat-fase5-report) | Busca o resultado no trigger e gera o PDF | Laravel, Clean Architecture, DomPDF |
| [diagram-analyzer-service](https://github.com/GGroff15/iatd-soat-hackaton-diagram-analyzer-service) | Baixa o diagrama, chama o YOLO, roda OCR e publica o resultado na fila | Python 3.12, FastAPI, arquitetura hexagonal |
| [yolo-inference](https://github.com/GGroff15/iatd-soat-hackaton-yolo-inference) | Detecta componentes e setas no desenho | Python 3.12, FastAPI, Ultralytics YOLOv8 |
| [message-broker](https://github.com/felipeoli7eira/message-broker) | Sobe o RabbitMQ; tem o enunciado do hackathon e o diagrama de arquitetura | RabbitMQ |
| [observability](https://github.com/GGroff15/iatd-soat-hackaton-observability) | Traces, logs e métricas de todos os serviços | OpenTelemetry, Grafana, Tempo, Loki, Prometheus |

Todos rodam na rede Docker externa `soat-net` (`docker network create soat-net`).

---

## Fluxo da Solução

**Envio para análise:**
1. Usuário envia diagrama via `POST /api/upload` neste **BFF**
2. BFF repassa ao **upload-service**, que armazena no Amazon S3
3. **upload-service** publica `{ protocol_uuid, file_url, ... }` na fila `protocols` do RabbitMQ
4. **trigger-service** consome a mensagem e aciona o **Analysis Service (IA)**
5. IA processa e publica o resultado na fila `analysis_response`
6. **trigger-service** persiste o resultado no PostgreSQL (`SUCESSO` ou `ERRO`)

**Consulta do resultado:**
1. Usuário consulta status via `GET /api/status/{uuid}` neste **BFF**
2. Usuário solicita relatório via `GET /api/report/{uuid}` neste **BFF**
3. **report-service** busca os dados no **trigger-service** e gera o PDF

---

## Segurança

### Requisitos básicos adotados

- **Credenciais via variáveis de ambiente**: URLs e configurações dos serviços downstream são injetadas via `.env`, sem valores hardcoded
- **Superfície de ataque reduzida**: o BFF é o **único ponto de entrada público** do sistema — os demais serviços ficam acessíveis apenas na rede interna `soat-net`

### Validação e tratamento de entradas não confiáveis

- Validação de tipo MIME e extensão do arquivo (`image/jpeg`, `image/png`, `application/pdf`) antes de repassar ao upload-service
- Tamanho máximo de **2 MB** por arquivo
- `protocol_uuid` validado como UUID antes das chamadas de status/relatório
- Erros retornados pelos serviços downstream são tratados e normalizados antes de expor ao cliente — detalhes internos não são vazados nas respostas de erro

### Uso controlado do modelo de IA

- O BFF não se comunica diretamente com a IA — apenas orquestra chamadas ao **upload-service** e ao **report-service**
- O controle do escopo e do payload enviado à IA é responsabilidade do **trigger-service**

### Tratamento de falhas e comportamentos inesperados da IA

- Timeouts configuráveis (`UPLOAD_SERVICE_TIMEOUT_SECONDS`, `REPORT_SERVICE_TIMEOUT_SECONDS`) evitam que lentidão ou falha da IA trave requisições do cliente
- Erros `502` são retornados ao cliente quando serviços internos estão indisponíveis, sem expor detalhes do stack interno

### Comunicação entre serviços

- **Rede Docker interna** (`soat-net`): chamadas ao upload-service e report-service trafegam apenas na rede interna
- Timeouts HTTP configurados para todas as chamadas downstream
- O BFF atua como **facade**: o cliente nunca se comunica diretamente com os serviços internos

### Riscos e limitações conhecidos

| Risco | Impacto | Mitigação atual |
|---|---|---|
| Respostas de erro podem expor detalhes internos | Mensagens de stack ou nomes de serviços internos visíveis ao cliente | Erros normalizados antes de retornar ao cliente; `502` genérico para falhas downstream |
| Lentidão em um serviço downstream pode degradar a experiência | Requisições lentas acumulam no BFF | Timeouts configuráveis por serviço (`*_TIMEOUT_SECONDS`) |

---

## Setup via Docker Compose

A rede _default_ dos serviços é uma rede externa chamada `soat-net`. Crie-a antes de subir os containers:

```sh
docker network create soat-net
```

Em seguida, suba o serviço:

```sh
docker compose up -d
```

---

## Variáveis de Ambiente

| Variável | Descrição | Padrão |
|---|---|---|
| `UPLOAD_SERVICE_BASE_URL` | URL base do serviço de upload | — |
| `UPLOAD_SERVICE_TIMEOUT_SECONDS` | Timeout (em segundos) para chamadas ao serviço de upload | `10` |
| `REPORT_SERVICE_BASE_URL` | URL base do serviço de relatório | — |
| `REPORT_SERVICE_TIMEOUT_SECONDS` | Timeout (em segundos) para chamadas ao serviço de relatório | `10` |

---

## Endpoints

| Método | Rota | Descrição |
|---|---|---|
| `GET` | `/api/ping` | Health check do próprio BFF |
| `GET` | `/api/ping/upload` | Health check repassado ao serviço de upload |
| `GET` | `/api/ping/report` | Health check repassado ao serviço de relatório |
| `POST` | `/api/upload` | Envia um diagrama para análise (multipart/form-data, campo `diagram`) |
| `GET` | `/api/report/{uuid}` | Faz o download do relatório em PDF gerado para o protocolo informado |
| `GET` | `/api/report/{uuid}/status` | Consulta o status de processamento do relatório |
| `GET` | `/api/status/{uuid}` | Alias de `/api/report/{uuid}/status` |

### Upload — campo obrigatório

| Campo | Tipo | Regras |
|---|---|---|
| `diagram` | arquivo | JPG, JPEG, PNG ou PDF · máximo 2 MB |

---

## Testes

### Pré-requisitos

- Docker e Docker Compose
- Container `soat-bff` em execução (`docker compose up -d`)

### 1. Instalar dependências (incluindo dev)

```bash
docker exec soat-bff composer install
```

> Necessário apenas na primeira vez ou após alterações no `composer.json`.

### 2. Executar os testes

```bash
docker exec soat-bff vendor/bin/phpunit
```

### 3. Executar com relatório de cobertura HTML

```bash
docker exec soat-bff vendor/bin/phpunit --coverage-html var/coverage/html
```

O relatório estará disponível em `application/var/coverage/html/index.html`.

### Estrutura dos testes

| Suite   | Local                    | O que testa                              |
|---------|--------------------------|------------------------------------------|
| Feature | `tests/Feature/`         | Endpoints HTTP e integração com serviços externos (mockados via `Http::fake()`) |

## Equipe

### Integrantes IADT

| Nome | RM |
|---|---|
| Angelo Rossi | RM365902 |
| Carlos Eduardo | RM365213 |
| Felipe Goiabeira | RM365753 |
| Guilherme Groff | RM365281 |
| Rafael Lua | RM366254 |

### Integrantes SOAT

| Nome | RM |
|---|---|
| Felipe Alves de Oliveira | RM365154 |
| Nicolas Henrique Correa Martins | RM365746 |
| William Francisco Leite | RM365973 |
