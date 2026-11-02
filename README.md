# FIAP Cloud Games - Fase 02 - Orquestração & Ponto de Partida (`fcg-fase-02--orchestration`)

[![.NET 10](https://img.shields.io/badge/.NET-10.0-purple.svg)](https://dotnet.microsoft.com/)
[![Azure Database](https://img.shields.io/badge/Azure_Database-SQL-0089D6.svg)](https://azure.microsoft.com/)
[![RabbitMQ](https://img.shields.io/badge/RabbitMQ-3.13-orange.svg)](https://www.rabbitmq.com/)
[![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED.svg)](https://www.docker.com/)

> 🎓 **Projeto Acadêmico FIAP**  
> Este repositório centraliza a **orquestração local e a documentação geral** do projeto de conclusão de curso da **"Pós-Tech: Arquitetura de Sistemas .NET"** da **FIAP**.  
> O objetivo da solução é projetar e implementar um ecossistema distribuído de nuvem voltado para uma plataforma de jogos digitais (*Cloud Gaming*), aplicando os mais modernos padrões de arquitetura corporativa em **.NET 10**.

---

## 📑 Sumário

- [1. Visão Geral da Fase 02](#1-visão-geral-da-fase-02)
  - [1.1. Os 5 Repositórios do Ecossistema](#11-os-5-repositórios-do-ecossistema)
  - [1.2. Topologia de Pastas Recomendada](#12-topologia-de-pastas-recomendada)
- [2. Como Executar o Ecossistema Localmente](#2-como-executar-o-ecossistema-localmente)
  - [2.1. Pré-requisitos](#21-pré-requisitos)
  - [2.2. Execução Unificada via Docker Compose (Recomendado)](#22-execução-unificada-via-docker-compose-recomendado)
- [3. Visão Global da Solução & Fluxo Distribuído](#3-visão-global-da-solução--fluxo-distribuído)
- [4. Guia Oficial de Docker & Containerização](#4-guia-oficial-de-docker--containerização)
- [5. Guia Oficial de Kubernetes (K8s)](#5-guia-oficial-de-kubernetes-k8s)
- [6. Guia de Desenvolvimento .NET no VS Code](#6-guia-de-desenvolvimento-net-no-vs-code)
- [7. Guia Técnico de RabbitMQ](#7-guia-técnico-de-rabbitmq)
- [8. Guia Para Teste Integrado](#8-guia-para-teste-integrado)

---

## 1. Visão Geral da Fase 02

A Fase 02 do projeto **FIAP Cloud Games** é orientada a microsserviços desacoplados e distribuídos com **Database-per-Service** (bancos dedicados no Azure Database) e comunicação assíncrona orientada a eventos (**Event-Driven Architecture - EDA**) através do **RabbitMQ**.

### 1.1. Os 5 Repositórios do Ecossistema

O ecossistema é composto por **5 repositórios Git independentes**:

| Repositório |
|---|
| **`fcg-fase-02--orchestration`**<br/>**Orquestração Local (Este repositório):** Ponto de partida do ecossistema, contendo o Docker Compose unificado com a mensageria RabbitMQ, integração com o Azure Database e orquestração de todos os microsserviços.<br/>[github.com/idgrego/fcg-fase-02--orchestration](https://github.com/idgrego/fcg-fase-02--orchestration) |
| **`fcg-fase-02--users`**<br/>**UsersAPI:** Microsserviço de gestão de identidade, usuários, emissão de tokens JWT e controle de papéis (`Anonymous`, `Gamer`, `Admin`).<br/>[github.com/idgrego/fcg-fase-02--users](https://github.com/idgrego/fcg-fase-02--users) |
| **`fcg-fase-02--catalog`**<br/>**CatalogAPI:** Gestão do catálogo de jogos, biblioteca do jogador, controle de compras e consumo de confirmações de pagamento.<br/>[github.com/idgrego/fcg-fase-02--catalog](https://github.com/idgrego/fcg-fase-02--catalog) |
| **`fcg-fase-02--payments`**<br/>**PaymentsAPI:** Simulador de gateway financeiro de pagamentos, validação de transações e emissão de eventos de resultado.<br/>[github.com/idgrego/fcg-fase-02--payments](https://github.com/idgrego/fcg-fase-02--payments) |
| **`fcg-fase-02--notifications`**<br/>**NotificationsAPI:** Central assíncrona de envio de e-mails transacionais (boas-vindas, confirmação de compra e falhas).<br/>[github.com/idgrego/fcg-fase-02--notifications](https://github.com/idgrego/fcg-fase-02--notifications) |

### 1.2. Topologia de Pastas Recomendada

Para que os scripts de orquestração local (`docker-compose.yml`), caminhos relativos de build e redes funcionem de forma automática, **todos os 5 repositórios devem ser clonados lado a lado no mesmo diretório pai**:

```text
meu-diretorio-de-projetos/
├── fcg-fase-02--orchestration/     # Repositório de Orquestração (docker-compose) - Ponto de Partida
├── fcg-fase-02--users/             # Repositório da UsersAPI
├── fcg-fase-02--catalog/           # Repositório da CatalogAPI
├── fcg-fase-02--payments/          # Repositório da PaymentsAPI
└── fcg-fase-02--notifications/     # Repositório da NotificationsAPI
```

```powershell
# Comandos para clonar os 5 repositórios:
git clone https://github.com/idgrego/fcg-fase-02--orchestration.git
git clone https://github.com/idgrego/fcg-fase-02--users.git
git clone https://github.com/idgrego/fcg-fase-02--catalog.git
git clone https://github.com/idgrego/fcg-fase-02--payments.git
git clone https://github.com/idgrego/fcg-fase-02--notifications.git
```

---

## 2. Como Executar o Ecossistema Localmente

### 2.1. Pré-requisitos
- [.NET 10 SDK](https://dotnet.microsoft.com/) ou superior instalado.
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) ou Docker Engine instalado e em execução.
- Git instalado.

---

### 2.2. Execução Unificada via Docker Compose (Recomendado)

A forma mais simples e robusta de subir todo o ambiente de desenvolvimento é utilizando a orquestração centralizada:

1. Abra o terminal na pasta **`fcg-fase-02--orchestration`**:
   ```powershell
   cd fcg-fase-02--orchestration
   ```

2. Execute o comando para subir toda a infraestrutura e os microsserviços:
   ```powershell
   docker compose up -d --build
   ```

3. **O que este comando inicializa:**
   - **`rabbitmq` (RabbitMQ 3 Management):** Porta `5672` (protocolo AMQP) e `15672` (painel web de gerenciamento).
   - **`users-api` (.NET 10):** Porta host `8080`, conectada ao banco dedicado `fcg-02-usersdb` no Azure Database e ao broker.
   - **`notifications-api` (.NET 10):** Porta host `8081`, conectada ao RabbitMQ (stateless worker).
   - **`catalog-api` (.NET 10):** Porta host `8082`, conectada ao banco dedicado `fcg-02-catalogdb` no Azure Database e ao broker.
   - **`payments-api` (.NET 10):** Porta host `8083`, conectada ao banco dedicado `fcg-02-paymentsdb` no Azure Database e ao broker.

   *(Nota: A persistência relacional do ecossistema é hospedada no SGBD **Azure Database** na nuvem no servidor `fcg-server.database.windows.net`, onde cada microsserviço opera seu próprio banco de dados isolado).*

4. **Verificar os serviços em execução:**
   ```powershell
   docker compose ps
   ```

5. **Acompanhar os logs:**
   ```powershell
   # Logs de todos os serviços:
   docker compose logs -f

   # Ou logs de um serviço específico:
   docker compose logs -f users-api
   docker compose logs -f payments-api
   docker compose logs -f catalog-api
   docker compose logs -f notifications-api
   ```

6. **Derrubar os serviços mantendo os dados persistidos:**
   ```powershell
   docker compose down
   ```

---

### 2.3. Execução e Deploy no Kubernetes Local (`kubectl`)

Para implantar toda a solução no cluster Kubernetes (ex: Docker Desktop K8s, Minikube ou Kind):

1. **Aplicar a infraestrutura de mensageria (RabbitMQ):**
   ```powershell
   kubectl apply -f k8s/01-rabbitmq.yaml
   ```

2. **Aplicar os manifestos de cada microsserviço (ConfigMaps, Secrets, Deployments e Services):**
   ```powershell
   # UsersAPI
   kubectl apply -f ../fcg-fase-02--users/k8s/

   # NotificationsAPI
   kubectl apply -f ../fcg-fase-02--notifications/k8s/

   # CatalogAPI
   kubectl apply -f ../fcg-fase-02--catalog/k8s/

   # PaymentsAPI
   kubectl apply -f ../fcg-fase-02--payments/k8s/
   ```

3. **Verificar se todos os Pods e Serviços estão saudáveis e em execução:**
   ```powershell
   kubectl get pods -l 'app in (users-api, notifications-api, catalog-api, payments-api, rabbitmq)'
   kubectl get services
   ```

4. **Acessar os microsserviços via port-forward:**
   ```powershell
   kubectl port-forward svc/users-api-service 8080:80
   kubectl port-forward svc/catalog-api-service 8082:80
   kubectl port-forward svc/payments-api-service 8083:80
   ```

---

## 3. Visão Global da Solução & Fluxo Distribuído

Para entender como todos os microsserviços interagem de forma síncrona (HTTP REST) e assíncrona (Event-Driven com RabbitMQ), consulte a documentação técnica completa:

👉 **[📖 Documentação Completa: Arquitetura e Fluxo Global da Solução Distribuída](docs/fluxo-geral-solucao.md)**

### O que você encontrará no documento detalhado:
- **Topologia de Infraestrutura:** Mapeamento de todos os contêineres, bancos independentes (*Database-per-Service* no Azure Database) e RabbitMQ na rede `fcg-network`.
- **Diagrama de Sequência Global (Mermaid):** Rastreabilidade visual numerada de todas as etapas da jornada do usuário.
- **Narrativa Fluida e Detalhada dos 4 Fluxos de Negócio:**
  1. *Cadastro de Usuário e Notificação de Boas-Vindas:* `UsersAPI` -> Azure Database (`fcg-02-usersdb`) -> RabbitMQ (`fcg.users.events` / `user.created`) -> `NotificationsAPI`.
  2. *Iniciação de Compra no Catálogo:* `CatalogAPI` -> Azure Database (`fcg-02-catalogdb`) -> RabbitMQ (`fcg.catalog.events` / `order.placed`) -> Resposta `202 Accepted`.
  3. *Processamento de Pagamento e Auditoria:* `PaymentsAPI` -> Simulador de Gateway financeiro -> Azure Database (`fcg-02-paymentsdb`) -> RabbitMQ (`fcg.payments.events` / `payment.approved` ou `payment.rejected`).
  4. *Finalização e Fan-Out Assíncrono:* Atualização idempotente da biblioteca (`CatalogAPI` no Azure Database) e disparo de e-mails transacionais (`NotificationsAPI`).
- **Contratos e Dicionário de Payloads:** Exemplos completos em JSON de todas as requisições, respostas HTTP e eventos AMQP que transitam pelo ecossistema.
- **Matriz de Mensageria:** Relação consolidada de Exchanges, Routing Keys e Filas.

---

## 4. Guia Oficial de Docker & Containerização

Para um aprofundamento completo sobre containerização com Docker, anatomia do Dockerfile, boas práticas com `.dockerignore`, Multi-Stage Build em .NET 10, ciclo de vida de contêineres, volumes persistentes, redes virtuais e orquestração centralizada com Docker Compose:

👉 **[🐳 Guia Oficial de Docker & Containerização](docs/docker/index.md)**

---

## 5. Guia Oficial de Kubernetes (K8s)

Para dominar a orquestração em clusters Kubernetes, arquitetura (Control Plane e Worker Nodes), tipos de manifestos (`ConfigMap`, `Secret` com Base64, `Deployment`, `Service`), ciclo de vida e rollout sem downtime, túneis locais com `kubectl port-forward`, redes CoreDNS e diagnóstico de erros:

👉 **[☸️ Guia Oficial de Kubernetes (K8s)](docs/kubernetes/index.md)**

---

## 6. Guia de Desenvolvimento .NET no VS Code

Para dominar o desenvolvimento corporativo em .NET 10 no Visual Studio Code, extensões recomendadas, comandos de solução e projetos na .NET CLI, automação com `tasks.json` e `launch.json`, execução simultânea de múltiplos microsserviços via `compounds` e depuração profissional com breakpoints:

👉 **[💻 Guia de Desenvolvimento .NET no VS Code](docs/vscode/index.md)**

---

## 7. Guia Técnico de RabbitMQ

Para um aprofundamento completo sobre a mensageria do projeto (protocolo AMQP, tipos de exchange, modelos Push/Pull, estratégias de Ack/Nack, DLQ, painel Web Management e implementação com .NET 10):

👉 **[🐇 Guia Completo de RabbitMQ & Mensageria Distribuída](docs/rabbitmq/index.md)**

---

## 8. Guia Para Teste Integrado

Esse documento contém instruções para realização de testes em modo debug no VSCode.

👉 **[🐇 Guia Para Teste Integrado](docs/teste-integrado-debug-vscode.md)**

---
