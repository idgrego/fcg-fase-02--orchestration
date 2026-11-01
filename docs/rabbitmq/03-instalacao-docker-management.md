# RabbitMQ: Instalação via Docker & Guia do Painel de Gerenciamento

Este guia orienta sobre como executar o **RabbitMQ** localmente utilizando **Docker** e **Docker Compose**, o mapeamento das portas essenciais e como operar a interface web de gestão (**Management UI**) para inspecionar filas, exchanges e mensagens do ecossistema **FIAP Cloud Games**.

---

## 📑 Sumário

- [1. Imagens Oficiais do Docker: Por que a tag `-management`?](#1-imagens-oficiais-do-docker-por-que-a-tag--management)
- [2. Mapeamento de Portas Essenciais](#2-mapeamento-de-portas-essenciais)
- [3. Execução Isolada via Docker CLI (`docker run`)](#3-execução-isolada-via-docker-cli-docker-run)
- [4. Execução Integrada no Docker Compose (`docker-compose.yml`)](#4-execução-integrada-no-docker-compose-docker-composeyml)
- [5. Guia Prático da Interface Web de Gestão (Management UI)](#5-guia-prático-da-interface-web-de-gestão-management-ui)
  - [5.1. Acessando a Interface](#51-acessando-a-interface)
  - [5.2. Aba Overview (Visão Geral de Saúde)](#52-aba-overview-visão-geral-de-saúde)
  - [5.3. Aba Exchanges (Inspecionar e Testar Roteamento)](#53-aba-exchanges-inspecionar-e-testar-roteamento)
  - [5.4. Aba Queues (Monitorar e Obter Mensagens)](#54-aba-queues-monitorar-e-obter-mensagens)
- [6. Comandos CLI Essenciais dentro do Contêiner (`rabbitmqctl`)](#6-comandos-cli-essenciais-dentro-do-contêiner-rabbitmqctl)

---

## 1. Imagens Oficiais do Docker: Por que a tag `-management`?

O repositório oficial do RabbitMQ no Docker Hub oferece duas variantes principais de imagem:

1. **Imagem Padrão (ex: `rabbitmq:3.13` ou `rabbitmq:4.0`):**
   - Contém apenas o daemon principal do broker AMQP.
   - Não possui interface web gráfica.
   - Mais leve (~100 MB), indicada para ambientes de produção altamente restritos onde o gerenciamento é feito via CLI ou agentes de telemetria.

2. **Imagem Management (ex: `rabbitmq:3-management` ou `rabbitmq:4-management`):**
   - Vem com o plugin oficial **`rabbitmq_management` pré-ativado de fábrica**.
   - Inclui um servidor HTTP embutido que serve uma interface web rica e uma API REST completa para administração.
   - **Recomendada para Desenvolvimento e utilizada no FIAP Cloud Games.**

---

## 2. Mapeamento de Portas Essenciais

Ao containerizar o RabbitMQ, duas portas são mandatórias:

| Porta | Protocolo | Finalidade | Quem Consome? |
|---|---|---|---|
| **`5672`** | **AMQP 0-9-1** | Conexão binária de alta velocidade para publicação e consumo | Nossos microsserviços em .NET (`UsersAPI`, `CatalogAPI`, `PaymentsAPI`, `NotificationsAPI`) |
| **`15672`** | **HTTP / Web UI** | Painel visual de administração e API REST administrativa | Desenvolvedores via navegador Web, Postman ou scripts de automação |

*(Outras portas opcionais em cenários avançados: `5671` para AMQP com TLS/SSL e `15692` para métricas do Prometheus).*

---

## 3. Execução Isolada via Docker CLI (`docker run`)

Se você desejar subir uma instância isolada do RabbitMQ para testes rápidos sem o Compose:

```powershell
docker run -d `
  --name fcg-rabbitmq `
  -p 5672:5672 `
  -p 15672:15672 `
  -e RABBITMQ_DEFAULT_USER=guest `
  -e RABBITMQ_DEFAULT_PASS=guest `
  -v rabbitmq_data:/var/lib/rabbitmq `
  rabbitmq:3-management
```

---

## 4. Execução Integrada no Docker Compose (`docker-compose.yml`)

No projeto **`fcg-fase-02--orchestration`**, o RabbitMQ já está declarado de forma resiliente, incluindo volume persistente e verificação contínua de saúde (**Healthcheck**):

```yaml
  # Broker de mensageria RabbitMQ
  rabbitmq:
    image: rabbitmq:3-management
    container_name: fcg-rabbitmq
    environment:
      - RABBITMQ_DEFAULT_USER=guest
      - RABBITMQ_DEFAULT_PASS=guest
    ports:
      - "5672:5672"   # Protocolo AMQP
      - "15672:15672" # Painel Web Management
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - fcg-network
    healthcheck:
      test: [ "CMD", "rabbitmq-diagnostics", "-q", "ping" ]
      interval: 10s
      timeout: 5s
      retries: 4
    restart: unless-stopped
```

### Por que o Healthcheck com `rabbitmq-diagnostics` é crucial?
O RabbitMQ é uma aplicação Erlang que leva alguns segundos para inicializar os subsistemas de rede e armazenamento. O healthcheck garante que os microsserviços (`users-api`, `catalog-api`, etc.) configurados com `condition: service_healthy` só inicializem quando o broker estiver **efetivamente pronto para aceitar conexões TCP**, evitando erros de *Connection Refused* no boot da máquina.

---

## 5. Guia Prático da Interface Web de Gestão (Management UI)

### 5.1. Acessando a Interface

1. Abra seu navegador em: **`http://localhost:15672`**
2. Credenciais padrão de desenvolvimento:
   - **Username:** `guest`
   - **Password:** `guest`

---

### 5.2. Aba Overview (Visão Geral de Saúde)

Na tela inicial (**Overview**), você encontra:
- **Taxas em Tempo Real:** Gráficos de mensagens publicadas por segundo (*Publish*), mensagens confirmadas (*Ack*) e mensagens entregues (*Deliver*).
- **Consumo de Recursos:** Uso de memória RAM do nó Erlang e espaço disponível no disco rígido.
- **Alarmes:** Indicadores visuais caso a máquina host atinja níveis críticos de memória (quando o RabbitMQ ativa o *Memory Alarm* e bloqueia publicações para evitar que o processo morra).

---

### 5.3. Aba Exchanges (Inspecionar e Testar Roteamento)

Na aba **Exchanges**, você verá as exchanges padrão do AMQP (`amq.direct`, `amq.topic`, etc.) e as exchanges criadas pela nossa aplicação:
- **`fcg.users.events`** (Topic)
- **`fcg.catalog.events`** (Topic)
- **`fcg.payments.events`** (Topic)

#### 🧪 Como publicar uma mensagem de teste diretamente pela UI:
1. Clique sobre a exchange desejada (ex: `fcg.users.events`).
2. Expanda o accordion **"Publish message"**.
3. No campo **Routing key**, informe: `user.created`.
4. No campo **Payload**, cole o JSON:
   ```json
   {
     "userId": "11111111-2222-3333-4444-555555555555",
     "name": "Jogador Teste",
     "email": "jogador.teste@fiap.com.br",
     "role": "Gamer",
     "createdAt": "2026-10-29T12:00:00Z"
   }
   ```
5. Clique no botão **"Publish message"**.
6. Vá até o terminal e veja a `NotificationsAPI` consumindo o evento e imprimindo o e-mail em tempo real!

---

### 5.4. Aba Queues (Monitorar e Obter Mensagens)

Na aba **Queues**, você visualiza o estado de cada fila declarada:
- **`notifications.user-created`**
- **`payments.order-placed`**
- **`catalog.payment-processed`**
- **`notifications.payment-processed`**

#### Indicadores das Colunas:
- **Ready:** Quantidade de mensagens esperando na fila para serem entregues aos consumidores.
- **Unacked:** Quantidade de mensagens que já foram entregues a um consumidor, mas o consumidor ainda não emitiu o `BasicAck` ou `BasicNack`.
- **Total:** Soma de Ready + Unacked.

#### 🔍 Como ler mensagens da fila sem deletá-las (*Get Message(s)*):
1. Clique sobre uma fila (ex: `payments.order-placed`).
2. Expanda o accordion **"Get messages"**.
3. Em **Requeue**, selecione **"Yes"** (para que a mensagem permaneça na fila após você lê-la).
4. Clique em **"Get Message(s)"**.
5. O painel exibirá o payload original, headers e propriedades completas da mensagem!

---

## 6. Comandos CLI Essenciais dentro do Contêiner (`rabbitmqctl`)

Você pode executar comandos administrativos diretamente via terminal:

```powershell
# Listar todas as filas e contagem de mensagens
docker exec -it fcg-rabbitmq rabbitmqctl list_queues name messages messages_ready messages_unacknowledged

# Listar todas as exchanges e seus tipos
docker exec -it fcg-rabbitmq rabbitmqctl list_exchanges name type

# Listar todas as conexões ativas de microsserviços
docker exec -it fcg-rabbitmq rabbitmqctl list_connections peer_address peer_port connected_at

# Testar o status de saúde do nó
docker exec -it fcg-rabbitmq rabbitmq-diagnostics ping
```
