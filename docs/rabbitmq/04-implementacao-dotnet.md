# RabbitMQ: Implementação Prática em .NET 10 (C# & RabbitMQ.Client 7.x)

Este guia apresenta a implementação prática de mensageria assíncrona em **.NET 10** utilizando o pacote oficial **`RabbitMQ.Client` v7.x** (totalmente assíncrono), demonstrando os padrões de **Publisher**, **Consumer em BackgroundService** e **Injeção de Dependências** aplicados no ecossistema **FIAP Cloud Games**.

---

## 📑 Sumário

- [1. A Nova API Assíncrona do `RabbitMQ.Client` (v7.x)](#1-a-nova-api-assíncrona-do-rabbitmqclient-v7x)
- [2. Configuração de Ambiente e Papéis no `appsettings.json`](#2-configuração-de-ambiente-e-papéis-no-appsettingsjson)
- [3. O Publicador de Eventos e a Idempotência do `ExchangeDeclareAsync`](#3-o-publicador-de-eventos-e-a-idempotência-do-exchangedeclareasync)
- [4. O Consumidor em Segundo Plano (`BackgroundService`)](#4-o-consumidor-em-segundo-plano-backgroundservice)
- [5. Resolução de Escopo com `IServiceScopeFactory`](#5-resolução-de-escopo-com-iservicescopefactory)
- [6. Múltiplos Exchanges e Múltiplas Filas no Mesmo Microsserviço](#6-múltiplos-exchanges-e-múltiplas-filas-no-mesmo-microsserviço)
- [7. Caso Real do Ecossistema: Fan-Out de Pagamentos](#7-caso-real-do-ecossistema-fan-out-de-pagamentos)
- [8. Registro no Container de Injeção de Dependências (`Program.cs`)](#8-registro-no-container-de-injeção-de-dependências-programcs)

---

## 1. A Nova API Assíncrona do `RabbitMQ.Client` (v7.x)

A versão 7.x do `RabbitMQ.Client` introduziu uma grande modernização arquitetural:
- **Assincronismo Total:** Todos os métodos que realizam I/O de rede são nativamente assíncronos (`CreateConnectionAsync`, `CreateChannelAsync`, `BasicPublishAsync`, `BasicConsumeAsync`, etc.).
- **Zero-Allocation:** Utiliza `ReadOnlyMemory<byte>` no envio de payloads em vez de `byte[]`, reduzindo drasticamente alocações de memória na *Garbage Collection* sob alta carga.

### Instalação do Pacote via CLI:
```powershell
dotnet add package RabbitMQ.Client --version 7.2.2
```

---

## 2. Configuração de Ambiente e Papéis no `appsettings.json`

Em microsserviços orientados a eventos, cada serviço possui responsabilidades bem definidas em relação à mensageria:

- **Para o Publicador (Producer):** Precisa conhecer apenas a **`Exchange`** de destino e a **`RoutingKey`** de saída. O publicador **não conhece nem precisa saber da existência de filas**.
- **Para o Consumidor (Consumer):** Precisa conhecer a sua **`Queue`** (sua caixa de entrada), a **`Exchange`** da qual deseja receber dados e a **`RoutingKey`** (ou padrão curinga como `payment.*`) utilizada para criar a amarração (**Binding**) entre a exchange e a fila.

Exemplo de configuração básica em um microsserviço consumidor (`appsettings.json`):

```json
{
  "RabbitMQ": {
    "HostName": "localhost",
    "Port": 5672,
    "UserName": "guest",
    "Password": "guest",
    "Exchange": "fcg.users.events",
    "Queue": "notifications.user-created",
    "RoutingKey": "user.created"
  }
}
```

---

## 3. O Publicador de Eventos e a Idempotência do `ExchangeDeclareAsync`

O publicador é registrado como **Singleton** ou **Scoped**. Ele é responsável por serializar objetos em JSON, garantir a existência da Exchange e despachar os bytes para o RabbitMQ com persistência ativada.

### 💡 Como funciona o `channel.ExchangeDeclareAsync`?
O método `ExchangeDeclareAsync` é uma operação **estritamente idempotente**:
1. **Se a Exchange NÃO existir:** O RabbitMQ a cria imediatamente com os parâmetros solicitados (`type`, `durable`, `autoDelete`).
2. **Se a Exchange JÁ existir com os mesmos parâmetros:** O RabbitMQ **não faz nada** (operação *no-op*) e retorna sucesso instantaneamente.
3. **Se houver conflito de parâmetros:** Se a exchange já existir com propriedades divergentes (por exemplo: criada anteriormente como `Fanout` e você tentar declarar como `Topic`, ou criada com `durable: false` e você tentar declarar `durable: true`), o RabbitMQ recusa a operação e encerra o canal lançando uma exceção de rede: **`406 PRECONDITION_FAILED`**.

```csharp
using System.Text;
using System.Text.Json;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using RabbitMQ.Client;

namespace FCG02.Shared.Messaging;

public interface IEventPublisher
{
    Task PublishAsync<T>(string exchange, string routingKey, T message, CancellationToken cancellationToken = default);
}

public class RabbitMqEventPublisher : IEventPublisher
{
    private readonly ConnectionFactory _factory;
    private readonly ILogger<RabbitMqEventPublisher> _logger;

    public RabbitMqEventPublisher(IConfiguration configuration, ILogger<RabbitMqEventPublisher> logger)
    {
        _logger = logger;
        _factory = new ConnectionFactory
        {
            HostName = configuration["RabbitMQ:HostName"] ?? "localhost",
            Port = int.Parse(configuration["RabbitMQ:Port"] ?? "5672"),
            UserName = configuration["RabbitMQ:UserName"] ?? "guest",
            Password = configuration["RabbitMQ:Password"] ?? "guest"
        };
    }

    public async Task PublishAsync<T>(string exchange, string routingKey, T message, CancellationToken cancellationToken = default)
    {
        // 1. Cria a conexão e o canal de comunicação assíncronos
        await using var connection = await _factory.CreateConnectionAsync(cancellationToken);
        await using var channel = await connection.CreateChannelAsync(cancellationToken: cancellationToken);

        // 2. Garante que a Exchange do tipo Topic existe (idempotente)
        await channel.ExchangeDeclareAsync(
            exchange: exchange,
            type: ExchangeType.Topic,
            durable: true,
            autoDelete: false,
            cancellationToken: cancellationToken);

        // 3. Serializa o payload em JSON UTF-8
        var json = JsonSerializer.Serialize(message, new JsonSerializerOptions { PropertyNamingPolicy = JsonNamingPolicy.CamelCase });
        var body = Encoding.UTF8.GetBytes(json);

        // 4. Configura propriedades da mensagem (Persistente no disco do RabbitMQ)
        var properties = new BasicProperties
        {
            Persistent = true,
            ContentType = "application/json",
            MessageId = Guid.NewGuid().ToString(),
            Timestamp = new AmqpTimestamp(DateTimeOffset.UtcNow.ToUnixTimeSeconds())
        };

        // 5. Publica os bytes na Exchange com a Routing Key informada
        await channel.BasicPublishAsync(
            exchange: exchange,
            routingKey: routingKey,
            mandatory: false,
            basicProperties: properties,
            body: body,
            cancellationToken: cancellationToken);

        _logger.LogInformation(" [RabbitMQ] Evento publicado com sucesso. Exchange: {Exchange} | Key: {Key}", exchange, routingKey);
    }
}
```

---

## 4. O Consumidor em Segundo Plano (`BackgroundService`)

O consumidor opera como um worker assíncrono de longa duração (`BackgroundService`). Ele se conecta ao broker, declara a fila, configura o prefetch (QoS) e escuta eventos de forma reativa:

```csharp
using System.Text;
using System.Text.Json;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;

namespace FCG02.NotificationsAPI.Services;

public class RabbitMqConsumerService : BackgroundService
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<RabbitMqConsumerService> _logger;
    private IConnection? _connection;
    private IChannel? _channel;

    public RabbitMqConsumerService(IConfiguration configuration, ILogger<RabbitMqConsumerService> logger)
    {
        _configuration = configuration;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var hostName = _configuration["RabbitMQ:HostName"] ?? "localhost";
        var port = int.Parse(_configuration["RabbitMQ:Port"] ?? "5672");
        var exchange = _configuration["RabbitMQ:Exchange"] ?? "fcg.users.events";
        var queue = _configuration["RabbitMQ:Queue"] ?? "notifications.user-created";
        var routingKey = _configuration["RabbitMQ:RoutingKey"] ?? "user.created";

        var factory = new ConnectionFactory
        {
            HostName = hostName,
            Port = port,
            UserName = _configuration["RabbitMQ:UserName"] ?? "guest",
            Password = _configuration["RabbitMQ:Password"] ?? "guest"
        };

        // 1. Abre a conexão e o canal
        _connection = await factory.CreateConnectionAsync(stoppingToken);
        _channel = await _connection.CreateChannelAsync(cancellationToken: stoppingToken);

        // 2. Declara Exchange e Fila Durável
        await _channel.ExchangeDeclareAsync(exchange, ExchangeType.Topic, durable: true, cancellationToken: stoppingToken);
        await _channel.QueueDeclareAsync(queue, durable: true, exclusive: false, autoDelete: false, cancellationToken: stoppingToken);

        // 3. Cria a ligação (Binding)
        await _channel.QueueBindAsync(queue, exchange, routingKey, cancellationToken: stoppingToken);

        // 4. Configura Fair Dispatch (Não sobrecarregar memória do worker)
        await _channel.BasicQosAsync(prefetchSize: 0, prefetchCount: 1, global: false, cancellationToken: stoppingToken);

        // 5. Configura o Consumidor Assíncrono Reativo
        var consumer = new AsyncEventingBasicConsumer(_channel);
        consumer.ReceivedAsync += async (model, ea) =>
        {
            try
            {
                var body = ea.Body.ToArray();
                var json = Encoding.UTF8.GetString(body);

                _logger.LogInformation(" [RabbitMQ] Mensagem recebida na fila {Queue}: {Payload}", queue, json);

                // Executa o processamento de negócio (ex: disparar e-mail)
                await ProcessarEventoAsync(json);

                // Confirmação Manual de Sucesso (Remove da fila)
                await _channel.BasicAckAsync(deliveryTag: ea.DeliveryTag, multiple: false);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, " [RabbitMQ] Falha no processamento da mensagem.");

                // Rejeita a mensagem sem requeue (ou encaminha para a DLQ)
                await _channel.BasicNackAsync(deliveryTag: ea.DeliveryTag, multiple: false, requeue: false);
            }
        };

        // Inicia o consumo em modo Push
        await _channel.BasicConsumeAsync(queue: queue, autoAck: false, consumer: consumer, cancellationToken: stoppingToken);

        _logger.LogInformation(" [RabbitMQ] Worker ativo e escutando na fila: {Queue}", queue);

        // Mantém o worker vivo até o cancelamento do container
        await Task.Delay(Timeout.Infinite, stoppingToken);
    }

    private Task ProcessarEventoAsync(string json)
    {
        // Lógica de negócio: Formatação e disparo do e-mail
        return Task.CompletedTask;
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        if (_channel != null) await _channel.CloseAsync(cancellationToken);
        if (_connection != null) await _connection.CloseAsync(cancellationToken);
        await base.StopAsync(cancellationToken);
    }
}
```

---

## 5. Resolução de Escopo com `IServiceScopeFactory`

Um `BackgroundService` em .NET é registrado como **Singleton**. No entanto, a maioria dos serviços de negócio e DbContexts do Entity Framework Core são **Scoped** (vivem pelo tempo de uma requisição).

> ⚠️ **Atenção:** Você **NÃO PODE** injetar diretamente um serviço `Scoped` (como um `DbContext` ou `IUserRepository`) no construtor de um `BackgroundService`. Fazer isso resulta em violação de escopo e vazamento de memória!

### Como resolver corretamente:
Injete `IServiceScopeFactory` e crie um escopo temporário para cada mensagem recebida:

```csharp
consumer.ReceivedAsync += async (model, ea) =>
{
    // Cria um escopo sob demanda para a mensagem
    using (var scope = _serviceScopeFactory.CreateScope())
    {
        var dbContext = scope.ServiceProvider.GetRequiredService<PaymentsDbContext>();
        var paymentService = scope.ServiceProvider.GetRequiredService<IPaymentService>();

        await paymentService.ProcessOrderAsync(pedido);
    }

    await _channel.BasicAckAsync(ea.DeliveryTag, false);
};
```

---

## 6. Múltiplos Exchanges e Múltiplas Filas no Mesmo Microsserviço

Em arquiteturas reais de microsserviços (como no próprio ecossistema **FIAP Cloud Games**), é comum que um serviço precise:
1. Publicar eventos em uma Exchange (ex: a `CatalogAPI` publicando pedidos em `fcg.catalog.events`).
2. Consumir eventos vindos de outra Exchange (ex: a `CatalogAPI` consumindo resultados de pagamento vindos de `fcg.payments.events`).
3. Ou até mesmo consumir de múltiplas filas com propósitos de negócio distintos.

### 6.1. Como Estruturar o `appsettings.json` para Múltiplos Exchanges?

Em vez de uma única propriedade genérica, declaramos chaves semânticas para cada finalidade:

```json
{
  "RabbitMQ": {
    "HostName": "localhost",
    "Port": 5672,
    "UserName": "guest",
    "Password": "guest",
    
    // 1. Exchange para onde este microsserviço PUBLICA pedidos
    "Exchange": "fcg.catalog.events",

    // 2. Exchange de onde este microsserviço CONSOME pagamentos
    "PaymentsExchange": "fcg.payments.events",
    "PaymentProcessedQueue": "catalog.payment-processed",
    "PaymentProcessedRoutingKey": "payment.*"
  }
}
```

---

### 6.2. Como o Publicador Lida com Múltiplos Exchanges?

A nossa interface `IEventPublisher` foi projetada exatamente para ser **desacoplada de um Exchange fixo**:

```csharp
public interface IEventPublisher
{
    Task PublishAsync<T>(string exchange, string routingKey, T message, CancellationToken cancellationToken = default);
}
```

Como o método recebe a `exchange` por parâmetro dinâmico, uma única instância injetada de `IEventPublisher` pode publicar em quantos Exchanges forem necessários:

```csharp
// Publica um pedido no Exchange de catálogo
await _eventPublisher.PublishAsync("fcg.catalog.events", "order.placed", orderPlacedEvent);

// Publica um registro de auditoria em outro Exchange
await _eventPublisher.PublishAsync("fcg.audit.events", "audit.catalog.created", auditLogEvent);
```

---

### 6.3. Como Consumir de Múltiplas Filas ou Múltiplos Exchanges?

Para o consumo, existem duas abordagens arquiteturais consagradas:

#### Abordagem A: Múltiplos `BackgroundService` Dedicados (Padrão Ouro / Recomendado)
Cada fila possui a sua própria classe `BackgroundService` independente, com seu próprio canal, prefetch, ciclo de vida e logs:
- `OrderPlacedConsumerService : BackgroundService` (consome pedidos).
- `PaymentProcessedConsumerService : BackgroundService` (consome pagamentos).

No `Program.cs`, basta registrar ambos:
```csharp
builder.Services.AddHostedService<OrderPlacedConsumerService>();
builder.Services.AddHostedService<PaymentProcessedConsumerService>();
```
- **Vantagem:** Princípio da Responsabilidade Única (SRP). Se um consumidor tiver lentidão ou for reiniciado, o outro segue processando normalmente.

#### Abordagem B: Múltiplos Bindings na MESMA Fila
Se a aplicação precisa receber eventos de tipos ou Exchanges diferentes na **mesma fila**, basta executar múltiplos `channel.QueueBindAsync`:
```csharp
// Uma única fila 'minha-fila' ligada a duas exchanges diferentes:
await channel.QueueBindAsync("minha-fila", "fcg.users.events", "user.*");
await channel.QueueBindAsync("minha-fila", "fcg.payments.events", "payment.*");
```
O worker recebe mensagens de ambas as fontes e inspeciona o `ea.RoutingKey` para decidir qual caso de uso acionar.

---

## 7. Caso Real do Ecossistema: Fan-Out de Pagamentos

No **FIAP Cloud Games**, a `PaymentsAPI` publica um único evento quando uma transação é concluída:

```csharp
// Na PaymentsAPI:
var paymentEvent = new PaymentProcessedIntegrationEvent(
    PaymentId: Guid.NewGuid(),
    OrderId: orderId,
    UserId: userId,
    GameId: gameId,
    Amount: 199.90m,
    Status: "Approved",
    Reason: "Transação autorizada",
    ProcessedAt: DateTime.UtcNow
);

// Se aprovado, usa a key "payment.approved"
await _eventPublisher.PublishAsync("fcg.payments.events", "payment.approved", paymentEvent);
```

### Como múltiplos serviços consomem simultaneamente sem se conhecerem?
1. **CatalogAPI:**
   - Cria sua fila: `catalog.payment-processed`.
   - Faz o binding com padrão curinga: `channel.QueueBindAsync("catalog.payment-processed", "fcg.payments.events", "payment.*")`.
   - Recebe a mensagem e **adiciona o jogo à biblioteca**.
2. **NotificationsAPI:**
   - Cria sua fila: `notifications.payment-processed`.
   - Faz o binding com padrão curinga: `channel.QueueBindAsync("notifications.payment-processed", "fcg.payments.events", "payment.*")`.
   - Recebe a mensagem e **dispara o e-mail transacional**.

O RabbitMQ duplica o pacote em nível de rede e entrega para ambas as filas de forma 100% independente!

---

## 8. Registro no Container de Injeção de Dependências (`Program.cs`)

No arquivo `Program.cs` de cada microsserviço:

```csharp
// Registro do Publicador como Singleton
builder.Services.AddSingleton<IEventPublisher, RabbitMqEventPublisher>();

// Registro do Consumidor em Segundo Plano (Hosted Service)
builder.Services.AddHostedService<RabbitMqConsumerService>();
```
