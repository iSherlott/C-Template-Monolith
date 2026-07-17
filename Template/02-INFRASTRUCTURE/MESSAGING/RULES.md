# RULES — Infrastructure / Messaging

> Herda todas as regras de [`00-PRINCIPLES/ARCHITECTURE-RULES.md`](../../00-PRINCIPLES/ARCHITECTURE-RULES.md). Este documento especializa, nunca substitui.

## 1. Missão

Esta camada fornece o **canal assíncrono** entre módulos (o segundo canal de
comunicação definido em `ARCHITECTURE-RULES.md` seção 4): publicação e
consumo de `IntegrationEvents` via RabbitMQ, com garantia de entrega através
do padrão **Transactional Outbox**.

Módulo nunca fala com `RabbitMQ.Client` diretamente. Todo publish passa por
`IEventBus`; todo consume passa por uma classe base fornecida aqui.

## 2. Por que Outbox — o problema que resolve

Sem Outbox, um Handler que grava estado no banco **e** publica um evento tem
uma janela de inconsistência: se o processo cair entre o `commit` do banco e
o `publish` no broker, o evento nunca é publicado — o resto do sistema nunca
fica sabendo que aquilo aconteceu, mesmo o dado já estando salvo.

O Outbox resolve isso transformando o "publish" em uma **escrita na mesma
transação de banco** do resto do fluxo. A publicação real no RabbitMQ
acontece depois, de forma assíncrona, lida por um processo que garante que
todo evento gravado será, eventualmente, publicado — mesmo que o processo
original tenha caído logo em seguida.

## 3. Estrutura de pastas

`Messaging` é uma subpasta dentro do projeto único `Infrastructure`
(`Infrastructure/Infrastructure.csproj`), assim como `Database` e `Cache`
(`DATABASE/RULES.md` seção 3) — namespace `Infrastructure.Messaging`.

```
Infrastructure/
└── Messaging/
    ├── Interface/
    │   └── IEventBus.cs              # contrato usado pelos Handlers de módulo
    ├── Implementation/
    │   └── EventBus.cs               # grava na tabela outbox — nunca publica direto no RabbitMQ (seção 5)
    ├── Extensions/
    │   └── MessagingExtensions.cs    # AddMessaging(IServiceCollection, IConfiguration)
    ├── IntegrationEvent.cs           # classe base abstrata de todo evento (ver seção 4) — fica na raiz, não é interface nem implementação de serviço
    ├── RabbitMq/                     # pasta de funcionalidade coesa — mantém organização própria, ver seção 3.1
    │   ├── RabbitMqConnectionFactory.cs
    │   ├── RabbitMqPublisher.cs      # implementação interna — nunca referenciada fora daqui
    │   ├── RabbitMqConsumerBase.cs   # classe base que Consumers de módulo herdam/implementam
    │   └── RabbitMqOptions.cs        # opções de configuração (bind de appsettings)
    └── Outbox/                       # pasta de funcionalidade coesa — mantém organização própria, ver seção 3.1
        ├── OutboxMessage.cs          # modelo da linha da tabela outbox_messages
        ├── IOutboxWriter.cs          # usado internamente por IEventBus (grava na transação do Handler)
        ├── OutboxPublisherService.cs # BackgroundService: faz polling e publica no RabbitMQ
        └── IOutboxSchemaRegistry.cs  # registro dos schemas de módulo que possuem tabela outbox (ver seção 6)
```

### 3.1 Por que `RabbitMq/` e `Outbox/` não seguem a divisão `Interface/Implementation/`

`Database` e `Cache` (`DATABASE/RULES.md` seção 3.1) dividem por **tipo de
arquivo** porque seus arquivos são poucos e heterogêneos soltos na raiz.
`RabbitMq/` e `Outbox/` já são, cada uma, uma **área de funcionalidade
coesa e pequena** (poucos arquivos, um propósito único e específico) — o
mesmo raciocínio da seção 3.1 de `DATABASE/RULES.md` sobre `Migrations/`.
Fragmentar `Outbox/` em `Outbox/Interface/IOutboxWriter.cs`,
`Outbox/Implementation/OutboxPublisherService.cs` só pra seguir a convenção
à risca adicionaria navegação sem ganho real de legibilidade, já que os 4
arquivos cabem numa única tela e o nome de cada um já deixa claro o que é.
Regra prática: a divisão por tipo (`Interface/`, `Implementation/`,
`Factory/`, `Map/`, `Extensions/`) se aplica à raiz de `Database`/`Messaging`/
`Cache`; uma subpasta de funcionalidade já coesa (`Migrations/`, `RabbitMq/`,
`Outbox/`) não é obrigada a replicá-la internamente.

## 4. `IntegrationEvent` — onde a classe base vive

**Regra importante de dependência:** a classe base `IntegrationEvent` vive
**aqui, em `Infrastructure/Messaging`** — não em `Modules/Shared/Kernel`.

Motivo: se a base vivesse em `Shared/Kernel` (dentro de `Modules`),
`Infrastructure` precisaria referenciar `Modules` para conhecer o tipo,
violando a direção de dependência (`Modules → Infrastructure`, nunca o
contrário). Como o envelope de evento é, na prática, um conceito técnico de
mensageria (precisa de `Id`, `OccurredOn`, serialização), ele pertence à
Infrastructure. Eventos concretos de negócio (ex: `PedidoCriadoEvent`) vivem
em `Modules/Shared/Contracts/IntegrationEvents/` (`CONTRACTS/RULES.md`
seção 5), herdando desta base — não dentro do módulo publicador, para que o
módulo consumidor possa referenciar o tipo do evento sem `ProjectReference`
para quem o publica.

```csharp
public abstract record IntegrationEvent
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public DateTime OccurredOn { get; init; } = DateTime.UtcNow;
    public string EventType => GetType().Name;
}
```

**Precisa ser `record` (não `class`).** Um evento concreto (`PedidoCriadoEvent`)
é, ele próprio, um `record` (`CONTRACTS/RULES.md` seção 5) — e um `record` só
pode herdar de outro `record`, nunca de uma `class`. Descoberto em runtime
(`error CS8864` no build) na primeira vez que um evento concreto tentou
herdar desta base como `class`.

## 5. `IEventBus` — contrato usado pelo Handler

```csharp
public interface IEventBus
{
    Task PublishAsync<TEvent>(TEvent @event, string schema, IUnitOfWork unitOfWork)
        where TEvent : IntegrationEvent;
}
```

- `PublishAsync` **não publica no RabbitMQ diretamente** — ele serializa o evento e grava uma linha na tabela `outbox_messages` do schema do módulo chamador, usando o `IUnitOfWork` já aberto pelo Handler (mesma transação da escrita de domínio — ver `DATABASE/RULES.md` seção 5, e `HANDLER/RULES.md` seção 5 para o padrão de reaproveitar o mesmo `IUnitOfWork` entre `Repository` e `IEventBus`).
- Isso significa: publicar um evento é sempre parte do mesmo bloco transacional de uma operação de escrita. Não existe `PublishAsync` sem um `IUnitOfWork` em andamento — se o fluxo não grava nada no banco, não há "commit atômico" para garantir, e o caso de uso provavelmente não deveria usar Outbox (avaliar se cabe um publish direto, caso realmente não exista nenhuma escrita associada — exceção rara, documentar quando ocorrer).

**❌ Errado — `Handler` publicando direto no RabbitMQ, pulando o Outbox:**

```csharp
await _pedidoRepository.InserirAsync(pedido, unitOfWork);
unitOfWork.Commit();

await _rabbitMqPublisher.PublishAsync("vendas.events", "PedidoCriado", evento); // ❌ se o processo cair aqui, o commit já aconteceu mas o evento nunca é publicado
```

**✅ Correto — evento gravado na mesma transação, via `IEventBus`:**

```csharp
await _pedidoRepository.InserirAsync(pedido, unitOfWork);
await _eventBus.PublishAsync(new PedidoCriadoEvent(pedido.Id), "vendas", unitOfWork); // grava na outbox, mesma transação
unitOfWork.Commit(); // domínio + evento confirmam juntos ou revertem juntos
```

## 6. Tabela `outbox_messages` — uma por schema de módulo

Cada módulo tem sua própria tabela `outbox_messages` **dentro do seu próprio
schema** (ex: `vendas.outbox_messages`), nunca uma tabela outbox centralizada
fora dos schemas de módulo. Isso mantém a escrita do evento na mesma
transação/schema da escrita de domínio, sem exigir transação cross-schema.

```sql
CREATE TABLE <schema>.outbox_messages (
    id              UNIQUEIDENTIFIER PRIMARY KEY,
    event_type      NVARCHAR(200)   NOT NULL,
    payload         NVARCHAR(MAX)   NOT NULL,
    occurred_on     DATETIME2       NOT NULL,
    processed_on    DATETIME2       NULL
);
```

**Como o `OutboxPublisherService` (genérico, sem conhecer módulos) sabe quais
schemas tem tabela outbox:** cada módulo se registra no `IOutboxSchemaRegistry`
durante o próprio `Install()` do seu `IModuleInstaller` (ex: `registry.Register("vendas")`).
Isso preserva a direção de dependência — é o módulo que depende de
Infrastructure para se anunciar, nunca Infrastructure sabendo o nome de um
módulo em código fixo.

## 7. `OutboxPublisherService` — polling e publicação

- `BackgroundService` registrado por `AddMessaging()`, roda em intervalo curto configurável (ex: a cada 2s).
- A cada ciclo, para cada schema registrado: `SELECT TOP N * FROM <schema>.outbox_messages WITH (UPDLOCK, READPAST) WHERE processed_on IS NULL ORDER BY occurred_on`. O hint `UPDLOCK, READPAST` evita que duas instâncias do serviço peguem a mesma linha para publicar quando há mais de um processo rodando.
- Para cada linha lida: publica no exchange do módulo (ver seção 8), depois marca `processed_on = GETUTCDATE()`. Linhas processadas são mantidas para auditoria — política de retenção/limpeza a definir quando houver necessidade operacional real (ex: job de limpeza mensal).
- Se a publicação falhar (broker fora do ar), a linha permanece com `processed_on IS NULL` e será tentada novamente no próximo ciclo — o Outbox garante *at-least-once delivery*, nunca *exactly-once*. Ver seção 9 sobre idempotência do lado do consumidor.

## 8. Topologia RabbitMQ — convenção de nomes

| Elemento | Convenção | Exemplo |
|---|---|---|
| Exchange | `<schema-publicador>.events`, tipo `topic` | `vendas.events` |
| Routing key | Nome do evento (`EventType`), PascalCase | `PedidoCriado` |
| Queue | `<schema-consumidor>.<descricao-curta>` | `estoque.pedido-criado` |
| Dead-letter queue | `<queue>.dlq` | `estoque.pedido-criado.dlq` |

- Um módulo publicador tem **um único exchange** para todos os seus eventos (`<schema>.events`), nunca um exchange por tipo de evento.
- Um módulo consumidor cria uma queue por propósito de consumo, com bind explícito no exchange do publicador usando a routing key do evento desejado.
- Após N tentativas de processamento com falha (configurável, default 5), a mensagem é movida para a DLQ correspondente — não fica reprocessando indefinidamente na fila principal. Mensagens em DLQ exigem investigação manual, não são republicadas automaticamente.

## 9. Consumo — idempotência é obrigatória

RabbitMQ com Outbox garante *at-least-once delivery*: o mesmo evento **pode**
ser entregue mais de uma vez. Todo `Consumer` de módulo é responsável por ser
idempotente.

- Padrão recomendado: tabela `processed_messages` (`event_id UNIQUEIDENTIFIER PRIMARY KEY`, `processed_on DATETIME2`) dentro do schema do módulo consumidor. Antes de processar, o Consumer verifica se o `event_id` já existe nessa tabela; se sim, descarta a mensagem (ack sem reprocessar).
- A inserção em `processed_messages` acontece **na mesma transação** da escrita de negócio disparada pelo evento — mesmo `IUnitOfWork` explícito já usado no restante da camada de Database (`DATABASE/RULES.md` seção 5).
- `Consumer` implementa/herda `RabbitMqConsumerBase` (fornecida aqui) e vive fisicamente em `Modules/<Nome>/Consumers/`, como classe `internal` (`CONSUMERS/RULES.md` seção 3) — a classe base cuida de deserialização, ack/nack e retry/DLQ; o código de módulo só implementa o que fazer com o evento já deserializado.

**❌ Errado — `Consumer` sem checagem de idempotência (implementação manual, ignorando a base):**

```csharp
protected override async Task HandleAsync(PedidoCriadoEvent @event, IUnitOfWork unitOfWork)
{
    var estoque = await _estoqueRepository.ObterPorProdutoAsync(@event.PedidoId, unitOfWork);
    estoque.Reservar(@event.Itens); // ❌ se a mensagem for entregue de novo (at-least-once), reserva duas vezes
    await _estoqueRepository.AtualizarAsync(estoque, unitOfWork);
}
```

**✅ Correto — herdar `RabbitMqConsumerBase<TEvent>` (seção 3) já resolve isso automaticamente**, checando/gravando em `processed_messages` antes de chamar `HandleAsync` — o código de módulo não reimplementa essa checagem, só confia que `HandleAsync` só roda uma vez por `event_id`.

## 10. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| Handler chamando `RabbitMqPublisher` diretamente, pulando `IEventBus`/Outbox | Reintroduz a janela de inconsistência que o Outbox existe para eliminar |
| `PublishAsync` chamado fora de uma transação de escrita | Quebra a garantia de atomicidade evento+estado |
| Tabela outbox centralizada fora dos schemas de módulo | Exigiria transação cross-schema, violando isolamento de módulo |
| Consumer sem checagem de idempotência | Reprocessamento duplicado (garantido de acontecer eventualmente com at-least-once) causa efeito colateral duplicado no domínio |
| Exchange por tipo de evento (em vez de um exchange por módulo) | Explosão de exchanges, dificulta descoberta de "o que um módulo publica" |
| Mensagem reprocessada indefinidamente sem ir para DLQ | Mascarar erro permanente como se fosse transitório |

## 11. Enforcement

- Code review bloqueia qualquer uso direto de `RabbitMqPublisher`/`RabbitMQ.Client` fora de `Infrastructure/Messaging`.
- Contract tests (ver `04-TEST/CONTRACT/RULES.md` — a definir) validam que todo `IntegrationEvent` publicado por um módulo continua compatível com o que os Consumers de outros módulos esperam.
- Todo módulo que publica evento precisa ter, na sua migration inicial, a criação da tabela `outbox_messages` no próprio schema — ausência dela é bloqueio de PR.
