# Arquitetura e Fluxo Global da Solução Distribuída - FIAP Cloud Games

Este documento descreve a visão sistêmica, o fluxo de ponta a ponta e a interação assíncrona entre os microsserviços do ecossistema **FIAP Cloud Games (Fase 02)**.

---

## 📑 Sumário

- [1. Visão Geral da Topologia de Microsserviços e Contêineres](#1-visão-geral-da-topologia-de-microsserviços-e-contêineres)
- [2. Diagrama de Sequência Global do Ecossistema](#2-diagrama-de-sequência-global-do-ecossistema)
- [3. Fluxo 1: Cadastro de Usuário e Notificação de Boas-Vindas](#3-fluxo-1-cadastro-de-usuário-e-notificação-de-boas-vindas)
  - [3.1. Narrativa do Fluxo](#31-narrativa-do-fluxo)
  - [3.2. Contratos e Payloads (Ida e Volta)](#32-contratos-e-payloads-ida-e-volta)
- [4. Fluxo 2: Iniciação de Compra de Jogo no Catálogo](#4-fluxo-2-iniciação-de-compra-de-jogo-no-catálogo)
  - [4.1. Narrativa do Fluxo](#41-narrativa-do-fluxo)
  - [4.2. Contratos e Payloads (Ida e Volta)](#42-contratos-e-payloads-ida-e-volta)
- [5. Fluxo 3: Processamento do Pagamento e Auditoria](#5-fluxo-3-processamento-do-pagamento-e-auditoria)
  - [5.1. Narrativa do Fluxo](#51-narrativa-do-fluxo)
  - [5.2. Contratos e Payloads (Ida e Volta)](#52-contratos-e-payloads-ida-e-volta)
- [6. Fluxo 4: Finalização da Compra e Fan-Out Assíncrono](#6-fluxo-4-finalização-da-compra-e-fan-out-assíncrono)
  - [6.1. Cenário A: Pagamento Aprovado (Liberação na Biblioteca e Confirmação)](#61-cenário-a-pagamento-aprovado-liberação-na-biblioteca-e-confirmação)
  - [6.2. Cenário B: Pagamento Rejeitado (Alerta ao Usuário)](#62-cenário-b-pagamento-rejeitado-alerta-ao-usuário)
- [7. Matriz de Mensageria e Topologia do RabbitMQ](#7-matriz-de-mensageria-e-topologia-do-rabbitmq)

---

## 1. Visão Geral da Topologia de Microsserviços e Contêineres

A solução adota rigorosamente os princípios de **Clean Architecture**, **Domain-Driven Design (DDD)**, o padrão arquitetural **Database-per-Service** (cada serviço possui persistência isolada e autônoma) e comunicação desacoplada via mensageria assíncrona com **RabbitMQ**.

Todos os componentes operam conectados sob a mesma rede virtual bridge do Docker (**`fcg-network`**). 

A visualização abaixo divide o ecossistema em **dois grandes quadros independentes**:
- **Quadro da Esquerda (⬅️):** O ciclo isolado de cadastro de usuário e disparo do e-mail de boas-vindas.
- **Quadro da Direita (➡️):** A jornada completa de compra, iniciando pela autenticação do jogador com token JWT, geração de pedido, processamento financeiro e entrega assíncrona do jogo na biblioteca e recibo por e-mail.

```mermaid
graph LR
    subgraph PainelEsquerdo ["⬅️ FLUXO 1: REGISTRO DE USUÁRIO & BOAS-VINDAS"]
        direction TB
        GamerReg["👤 Gamer / Visitante"]
        UsersReg["👤 UsersAPI (:8080)<br/><i>Container: fcg-ms-users-api</i>"]
        DBUsersReg[("☁️ Azure Database<br/>Database: fcg-02-usersdb (:1433)<br/><i>Server: fcg-server.database.windows.net</i>")]
        BrokerReg[("🐇 RabbitMQ Broker (:5672)<br/><i>Container: fcg-rabbitmq</i><br/>Exchange: fcg.users.events<br/>Key: user.created")]
        QueueNotifReg[["📬 Fila: notifications.user-created"]]
        NotifReg["🔔 NotificationsAPI (:8081)<br/><i>Container: fcg-ms-notifications-api</i>"]
        EmailReg["📧 E-mail de Boas-Vindas<br/><i>(Log no Console)</i>"]

        GamerReg -->|"1. POST /api/v1/auth/register"| UsersReg
        UsersReg -->|"2. Grava na tabela 'users'"| DBUsersReg
        UsersReg -.->|"3. Publica UserCreatedEvent"| BrokerReg
        BrokerReg -->|"4. Roteia mensagem"| QueueNotifReg
        QueueNotifReg -->|"5. Consome (Worker)"| NotifReg
        NotifReg -->|"6. Envia e emite BasicAck"| EmailReg
        UsersReg -->|"7. Resposta HTTP 201 Created"| GamerReg
    end

    subgraph PainelDireito ["➡️ FLUXO 2: LOGIN & COMPRA DE JOGOS (AUTENTICADO)"]
        direction TB
        GamerBuy["👤 Gamer Autenticado"]
        UsersAuth["👤 UsersAPI (:8080)<br/><i>Emissão de Token JWT</i>"]
        CatalogAPI["🎮 CatalogAPI (:8082)<br/><i>Container: fcg-ms-catalog-api</i>"]
        DBCatalog[("☁️ Azure Database<br/>Database: fcg-02-catalogdb (:1433)<br/><i>Server: fcg-server.database.windows.net</i>")]
        BrokerCatalog[("🐇 RabbitMQ Broker (:5672)<br/>Exchange: fcg.catalog.events<br/>Key: order.placed")]
        QueuePayments[["📬 Fila: payments.order-placed"]]
        PaymentsAPI["💳 PaymentsAPI (:8083)<br/><i>Container: fcg-ms-payments-api</i>"]
        DBPayments[("☁️ Azure Database<br/>Database: fcg-02-paymentsdb (:1433)<br/><i>Server: fcg-server.database.windows.net</i>")]
        BrokerPayments[("🐇 RabbitMQ Broker (:5672)<br/>Exchange: fcg.payments.events<br/>Key: payment.*")]
        QueueCatalogPay[["📬 Fila: catalog.payment-processed"]]
        QueueNotifPay[["📬 Fila: notifications.payment-processed"]]
        NotifPay["🔔 NotificationsAPI (:8081)<br/><i>Container: fcg-ms-notifications-api</i>"]
        EmailBuy["📧 E-mail de Comprovante ou Recusa<br/><i>(Log no Console)</i>"]

        GamerBuy -->|"1. POST /api/v1/auth/login"| UsersAuth
        UsersAuth -->|"2. Retorna Bearer JWT Token"| GamerBuy
        GamerBuy -->|"3. POST /api/v1/orders (Bearer JWT)"| CatalogAPI
        CatalogAPI -->|"4. Valida jogo na tabela 'Games'"| DBCatalog
        CatalogAPI -->|"5. Resposta HTTP 202 Accepted"| GamerBuy
        CatalogAPI -.->|"6. Publica OrderPlacedEvent"| BrokerCatalog
        BrokerCatalog -->|"7. Roteia mensagem"| QueuePayments
        QueuePayments -->|"8. Consome (Worker)"| PaymentsAPI
        PaymentsAPI -->|"9. Simula e grava auditoria"| DBPayments
        PaymentsAPI -.->|"10. Publica PaymentProcessedEvent"| BrokerPayments

        BrokerPayments -->|"11a. payment.approved"| QueueCatalogPay
        QueueCatalogPay -->|"12a. Adiciona jogo à biblioteca"| CatalogAPI
        CatalogAPI -->|"13a. Atualiza 'UserLibraries'"| DBCatalog

        BrokerPayments -->|"11b. payment.approved / rejected"| QueueNotifPay
        QueueNotifPay -->|"12b. Consome (Worker)"| NotifPay
        NotifPay -->|"13b. Dispara e-mail e emite BasicAck"| EmailBuy
    end

    %% Garante ordem estrita: PainelEsquerdo à esquerda e PainelDireito à direita
    PainelEsquerdo ~~~ PainelDireito
```

---

## 2. Diagrama de Sequência Global do Ecossistema

O diagrama abaixo apresenta o ciclo de vida completo: do registro do jogador à aquisição e entrega do jogo digital em sua biblioteca.

```mermaid
sequenceDiagram
    autonumber
    actor Gamer as 👤 Gamer / Cliente
    participant Users as 👤 UsersAPI<br/>(Porta 8080)
    participant DBUsers as ☁️ Azure DB Users<br/>(fcg-02-usersdb)
    participant Broker as 🐇 RabbitMQ Broker<br/>(fcg-rabbitmq)
    participant Notif as 🔔 NotificationsAPI<br/>(Porta 8081)
    participant Catalog as 🎮 CatalogAPI<br/>(Porta 8082)
    participant DBCatalog as ☁️ Azure DB Catalog<br/>(fcg-02-catalogdb)
    participant Payments as 💳 PaymentsAPI<br/>(Porta 8083)
    participant DBPayments as ☁️ Azure DB Payments<br/>(fcg-02-paymentsdb)

    %% ETAPA 1: CADASTRO
    rect rgb(240, 248, 255)
        Note over Gamer,Notif: ETAPA 1: Cadastro de Usuário e Notificação de Boas-Vindas
        Gamer->>Users: POST /api/v1/auth/register (Nome, Email, Senha, Role)
        Users->>Users: Valida Email (Value Object) e gera hash BCrypt
        Users->>DBUsers: INSERT INTO users ...
        DBUsers-->>Users: Usuário persistido
        Users-)Broker: Publica UserCreatedEvent<br/>Exchange: fcg.users.events | Key: user.created
        Users-->>Gamer: 201 Created (UserResponseDto com Id, Nome, Email)
        Broker-)Notif: Entrega UserCreatedEvent<br/>Fila: notifications.user-created
        Notif->>Notif: Monta e despacha E-mail de Boas-Vindas (Log em Console)
        Notif--)Broker: BasicAck (Mensagem finalizada com sucesso)
    end

    %% ETAPA 2: COMPRA DO JOGO
    rect rgb(255, 250, 240)
        Note over Gamer,Payments: ETAPA 2: Iniciação de Compra de Jogo
        Gamer->>Catalog: POST /api/v1/orders (GameId, Preço) [Bearer JWT Token]
        Catalog->>Catalog: Valida autenticação JWT e extrai UserId
        Catalog->>DBCatalog: Consulta jogo na collection 'games' e valida duplicidade
        DBCatalog-->>Catalog: Jogo disponível
        Catalog-)Broker: Publica OrderPlacedEvent<br/>Exchange: fcg.catalog.events | Key: order.placed
        Catalog-->>Gamer: 202 Accepted (OrderResponseDto: OrderId, Status: PendingPayment)
    end

    %% ETAPA 3: PROCESSAMENTO DE PAGAMENTO
    rect rgb(245, 255, 245)
        Note over Broker,DBPayments: ETAPA 3: Processamento Financeiro e Auditoria
        Broker-)Payments: Entrega OrderPlacedEvent<br/>Fila: payments.order-placed
        Payments->>Payments: Executa PaymentGatewaySimulator (regras determinísticas)
        Payments->>DBPayments: INSERT INTO payment_transactions (OrderId, Amount, Status, Reason...)
        DBPayments-->>Payments: Transação financeira registrada
        Payments--)Broker: BasicAck (Pedido processado)
    end

    %% ETAPA 4: FAN-OUT E FINALIZAÇÃO
    rect rgb(255, 245, 245)
        alt Pagamento Aprovado (Status: Approved)
            Payments-)Broker: Publica PaymentProcessedEvent<br/>Exchange: fcg.payments.events | Key: payment.approved
            
            par Liberação na Biblioteca (CatalogAPI)
                Broker-)Catalog: Entrega PaymentProcessedEvent (Key: payment.*)<br/>Fila: catalog.payment-processed
                Catalog->>DBCatalog: Atualiza 'user_libraries' adicionando GameId ao UserId
                DBCatalog-->>Catalog: Jogo adicionado à biblioteca do jogador
                Catalog--)Broker: BasicAck
            and Envio de Notificação de Compra (NotificationsAPI)
                Broker-)Notif: Entrega PaymentProcessedEvent (Key: payment.*)<br/>Fila: notifications.payment-processed
                Notif->>Notif: Dispara E-mail com Recibo da Compra e Link de Download
                Notif--)Broker: BasicAck
            end

        else Pagamento Rejeitado (Status: Rejected)
            Payments-)Broker: Publica PaymentProcessedEvent<br/>Exchange: fcg.payments.events | Key: payment.rejected
            Broker-)Notif: Entrega PaymentProcessedEvent<br/>Fila: notifications.payment-processed
            Notif->>Notif: Dispara E-mail informando motivo da recusa (ex: limite excedido)
            Notif--)Broker: BasicAck
        end
    end
```

---

## 3. Fluxo 1: Cadastro de Usuário e Notificação de Boas-Vindas

### 3.1. Narrativa do Fluxo

1. **Requisição do Cliente:** O usuário envia uma requisição `POST /api/v1/auth/register` para a **`UsersAPI`** (porta host `8080`, container `fcg-ms-users-api`).
2. **Validação de Domínio:** A camada de domínio valida o formato do e-mail através do Value Object `Email` (utilizando Expressão Regular compilada) e a senha é criptografada com algoritmo de salt adaptativo via `BCrypt`.
3. **Persistência Relacional:** O usuário é gravado na tabela `users` do banco de dados relacional **`fcg-02-usersdb`** hospedado no **Azure Database** (`fcg-server.database.windows.net`, porta `1433`).
4. **Resposta Síncrona:** A API retorna imediatamente ao cliente o status `201 Created` contendo o `UserResponseDto`, sem reter a conexão aguardando envio de e-mails.
5. **Publicação do Evento:** Em segundo plano, a `UsersAPI` publica o evento assíncrono **`UserCreatedEvent`** no broker **RabbitMQ** (container `fcg-rabbitmq`), na Topic Exchange **`fcg.users.events`** com a routing key **`user.created`**.
6. **Consumo e Notificação:** O microsserviço **`NotificationsAPI`** (container `fcg-ms-notifications-api`), através do seu serviço em segundo plano `RabbitMqConsumerService`, consome a mensagem da fila **`notifications.user-created`**.
7. **Disparo Transacional:** O e-mail de boas-vindas é gerado pelo serviço `ConsoleEmailSender` e o registro de auditoria é impresso no console. Ao final, é emitido um **`BasicAck`** manual para o RabbitMQ remover a mensagem da fila.

### 3.2. Contratos e Payloads (Ida e Volta)

#### Requisição HTTP de Cadastro (`RegisterUserDto`):
```json
POST /api/v1/auth/register
Content-Type: application/json

{
  "name": "Bruce Wayne",
  "email": "bruce.wayne@waynecorp.com",
  "password": "SenhaUltraForte123!",
  "role": "Gamer"
}
```

#### Resposta HTTP de Sucesso (`UserResponseDto`):
```json
HTTP/1.1 201 Created
Location: /api/v1/users/550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Bruce Wayne",
  "email": "bruce.wayne@waynecorp.com",
  "role": "Gamer",
  "createdAt": "2026-10-29T10:00:00Z"
}
```

#### Evento Publicado no RabbitMQ (`UserCreatedEvent`):
- **Exchange:** `fcg.users.events`
- **Routing Key:** `user.created`
- **Fila Destino:** `notifications.user-created`
```json
{
  "eventId": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
  "occurredOn": "2026-10-29T10:00:00.125Z",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Bruce Wayne",
  "email": "bruce.wayne@waynecorp.com",
  "role": "Gamer",
  "createdAt": "2026-10-29T10:00:00Z"
}
```

---

## 4. Fluxo 2: Iniciação de Compra de Jogo no Catálogo

### 4.1. Narrativa do Fluxo

1. **Requisição de Compra:** O jogador autenticado envia `POST /api/v1/orders` para a **`CatalogAPI`** (porta host `8082`, container `fcg-ms-catalog-api`), anexando no cabeçalho o token JWT emitido pela `UsersAPI` (`Authorization: Bearer <token>`).
2. **Autenticação e Extração de Claims:** O middleware do ASP.NET Core valida a assinatura do token JWT e extrai o `UserId` diretamente das claims do usuário logado.
3. **Consulta no Catálogo Relacional:** A `CatalogAPI` consulta a tabela `Games` no banco relacional **`fcg-02-catalogdb`** (Azure Database) para validar a existência do jogo, seu preço atual e checar se o jogador já não o possui em sua biblioteca (`UserLibraries`).
4. **Geração do Pedido Provisório:** Um identificador de pedido (`OrderId`) é gerado com status pendente (`PendingPayment`).
5. **Publicação do Evento de Pedido:** A `CatalogAPI` publica o evento assíncrono **`OrderPlacedEvent`** no RabbitMQ (container `fcg-rabbitmq`), na Topic Exchange **`fcg.catalog.events`** com a routing key **`order.placed`**.
6. **Resposta Síncrona Imediata:** A `CatalogAPI` responde ao cliente com **`202 Accepted`**, informando que a ordem de compra foi recebida com sucesso e está sendo processada de forma assíncrona.

### 4.2. Contratos e Payloads (Ida e Volta)

#### Requisição HTTP de Compra (`CreateOrderDto`):
```json
POST /api/v1/orders
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "gameId": "67210b3a4e9b8c0012e45678",
  "price": 199.90
}
```

#### Resposta HTTP (`OrderResponseDto`):
```json
HTTP/1.1 202 Accepted
Location: /api/v1/orders/770e8400-e29b-41d4-a716-446655440111
Content-Type: application/json

{
  "orderId": "770e8400-e29b-41d4-a716-446655440111",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "gameId": "67210b3a4e9b8c0012e45678",
  "price": 199.90,
  "status": "PendingPayment",
  "createdAt": "2026-10-29T10:05:00Z"
}
```

#### Evento Publicado no RabbitMQ (`OrderPlacedEvent`):
- **Exchange:** `fcg.catalog.events`
- **Routing Key:** `order.placed`
- **Fila Destino:** `payments.order-placed`
```json
{
  "eventId": "f8a7b6c5-d4e3-2a1b-0c9d-8e7f6a5b4c3d",
  "occurredOn": "2026-10-29T10:05:00.050Z",
  "orderId": "770e8400-e29b-41d4-a716-446655440111",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "gameId": "67210b3a4e9b8c0012e45678",
  "price": 199.90
}
```

---

## 5. Fluxo 3: Processamento do Pagamento e Auditoria

### 5.1. Narrativa do Fluxo

1. **Consumo do Pedido:** O microsserviço **`PaymentsAPI`** (porta host `8083`, container `fcg-ms-payments-api`) mantém um consumidor ativo (`OrderPlacedConsumerService`) escutando na fila **`payments.order-placed`** (ligada à exchange `fcg.catalog.events` pela chave `order.placed`).
2. **Processamento no Simulador de Gateway:** O serviço `PaymentGatewaySimulator` avalia a transação de forma determinística:
   - Valores menores ou iguais a zero ou superiores ao teto de risco de **R$ 5.000,00** resultam em status **`Rejected`** (com o respectivo motivo, e.g., *"Valor excede limite máximo de transação permitido"*).
   - Valores positivos e dentro dos limites resultam em status **`Approved`**.
3. **Persistência de Auditoria:** A entidade rica `PaymentTransaction` é gravada via Entity Framework Core na tabela `payment_transactions` do banco relacional **`fcg-02-paymentsdb`** (Azure Database, porta `1433`).
4. **Confirmação do Consumo:** A `PaymentsAPI` emite um `BasicAck` para a fila `payments.order-placed`, confirmando que o pedido foi devidamente processado e armazenado.
5. **Publicação do Resultado:** A `PaymentsAPI` publica o evento de integração **`PaymentProcessedEvent`** na Topic Exchange **`fcg.payments.events`**, direcionando para as routing keys:
   - **`payment.approved`** caso aprovado;
   - **`payment.rejected`** caso recusado.

### 5.2. Contratos e Payloads (Ida e Volta)

#### Objeto Persistido na Tabela `payment_transactions` (`fcg-02-paymentsdb`):
| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | `uniqueidentifier` | Chave primária da transação financeira |
| `order_id` | `uniqueidentifier` | Chave de correlação do pedido gerado no catálogo |
| `user_id` | `uniqueidentifier` | Identificador do jogador |
| `game_id` | `nvarchar(max)` | Identificador do jogo |
| `amount` | `decimal(18,2)` | Valor cobrado |
| `payment_method` | `nvarchar(50)` | Método utilizado (ex: `CreditCard`, `Pix`) |
| `status` | `nvarchar(50)` | `Pending`, `Approved` ou `Rejected` |
| `reason` | `nvarchar(max)` | Justificativa do status ou motivo de eventual recusa |
| `processed_at` | `datetime2` | Carimbo de data/hora da conclusão do processamento |

#### Evento Publicado no RabbitMQ (`PaymentProcessedEvent`):
- **Exchange:** `fcg.payments.events`
- **Routing Key:** `payment.approved` (ou `payment.rejected`)
```json
{
  "eventId": "12345678-abcd-ef01-2345-6789abcdef01",
  "occurredOn": "2026-10-29T10:05:01.320Z",
  "paymentId": "990e8400-e29b-41d4-a716-446655440222",
  "orderId": "770e8400-e29b-41d4-a716-446655440111",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "gameId": "67210b3a4e9b8c0012e45678",
  "amount": 199.90,
  "status": "Approved",
  "reason": "Transação autorizada com sucesso pelo gateway simulado.",
  "processedAt": "2026-10-29T10:05:01.300Z"
}
```

---

## 6. Fluxo 4: Finalização da Compra e Fan-Out Assíncrono

A exchange Topic **`fcg.payments.events`** opera um padrão **Fan-Out seletivo**, entregando o evento simultaneamente a múltiplos consumidores que possuem binding pattern `payment.*`.

### 6.1. Cenário A: Pagamento Aprovado (Liberação na Biblioteca e Confirmação)

Quando a chave é `payment.approved`:

1. **Reação da CatalogAPI (`catalog.payment-processed`):**
   - O worker de segundo plano da **`CatalogAPI`** recebe a mensagem.
   - Atualiza de forma idempotente as tabelas `UserLibraries` e `LibraryItems` no **Azure Database (`fcg-02-catalogdb`)**, inserindo o `GameId` na biblioteca pessoal do `UserId`.
   - A partir deste momento, o jogador pode consultar `GET /api/v1/library` e o jogo constará como adquirido e liberado para download/execução em nuvem.
   - Emite `BasicAck` ao RabbitMQ.

2. **Reação da NotificationsAPI (`notifications.payment-processed`):**
   - O worker de segundo plano da **`NotificationsAPI`** recebe a mensagem.
   - Monta um e-mail transacional de comprovante de compra:
     - **Destinatário:** E-mail do jogador (recuperado ou correlacionado).
     - **Assunto:** *"🎮 Pagamento Confirmado! Seu jogo já está disponível na biblioteca!"*
     - **Corpo:** Detalhes do `OrderId`, valor pago (`R$ 199.90`) e instruções de instalação.
   - Imprime o log formatado no console do contêiner e emite `BasicAck`.

### 6.2. Cenário B: Pagamento Rejeitado (Alerta ao Usuário)

Quando a chave é `payment.rejected`:

1. **Reação da CatalogAPI (`catalog.payment-processed`):**
   - A `CatalogAPI` recebe o evento com `Status: Rejected`.
   - O jogo **NÃO** é inserido na biblioteca do jogador. O pedido temporário é marcado como cancelado.
   - Emite `BasicAck`.

2. **Reação da NotificationsAPI (`notifications.payment-processed`):**
   - A `NotificationsAPI` recebe o evento com `Status: Rejected`.
   - Monta um e-mail transacional de notificação de erro:
     - **Assunto:** *"⚠️ Falha no processamento do pagamento do seu pedido"*
     - **Corpo:** Exibe o `OrderId`, o motivo da recusa (`Reason`) e orientações para troca de forma de pagamento.
   - Imprime o log no console e emite `BasicAck`.

---

## 7. Matriz de Mensageria e Topologia do RabbitMQ

A tabela a seguir consolida todas as exchanges, tópicos, filas e os respectivos microsserviços produtores e consumidores:

| Exchange | Tipo | Routing Key | Fila de Destino | Serviço Produtor | Serviço(s) Consumidor(es) | Contrato / Payload |
|---|---|---|---|---|---|---|
| **`fcg.users.events`** | `Topic` | `user.created` | `notifications.user-created` | `fcg-ms-users-api` | `fcg-ms-notifications-api` | `UserCreatedEvent` (Id, Nome, Email, Role) |
| **`fcg.catalog.events`** | `Topic` | `order.placed` | `payments.order-placed` | `fcg-ms-catalog-api` | `fcg-ms-payments-api` | `OrderPlacedEvent` (OrderId, UserId, GameId, Price) |
| **`fcg.payments.events`** | `Topic` | `payment.approved` | `catalog.payment-processed` | `fcg-ms-payments-api` | `fcg-ms-catalog-api` | `PaymentProcessedEvent` (Status: Approved) |
| **`fcg.payments.events`** | `Topic` | `payment.approved` | `notifications.payment-processed` | `fcg-ms-payments-api` | `fcg-ms-notifications-api` | `PaymentProcessedEvent` (Status: Approved) |
| **`fcg.payments.events`** | `Topic` | `payment.rejected` | `notifications.payment-processed` | `fcg-ms-payments-api` | `fcg-ms-notifications-api` | `PaymentProcessedEvent` (Status: Rejected, Reason) |

---

## 8. Considerações de Resiliência e Confiabilidade

1. **Desacoplamento Temporal:** Se a `NotificationsAPI` estiver temporariamente fora do ar durante uma compra, as mensagens acumulam na fila `notifications.payment-processed` e são processadas automaticamente assim que o contêiner reiniciar, sem perda de dados.
2. **Idempotência:** A adição de jogos na biblioteca do catálogo utiliza índices compostos exclusivos (`UserId` + `GameId`), garantindo que o reprocessamento de uma mensagem nunca duplique o item na conta do jogador.
3. **Manual Acknowledgement (Ack):** Nenhum microsserviço utiliza auto-ack. A mensagem só sai da fila do RabbitMQ após a conclusão bem-sucedida da gravação no banco ou do envio da notificação.
