# RULES — Modules / Consumers

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md`, `03-MODULES/RULES.md` e `02-INFRASTRUCTURE/MESSAGING/RULES.md`. Este documento especializa, nunca substitui.

## 1. Missão

`Consumer` é o ponto de entrada **assíncrono** de um módulo — o equivalente
ao `Controller`, mas reagindo a um `IntegrationEvent` publicado por outro
módulo em vez de a uma requisição HTTP.

## 2. Estrutura de pastas

```
Modules/<NomeModulo>/
└── Consumers/
    └── <EventoNoPassado>Consumer.cs   # ex: PedidoCriadoConsumer.cs (dentro do módulo Estoque, reagindo a evento de Vendas)
```

Um `Consumer` vive dentro do módulo que **reage** ao evento, não do módulo
que o publica. É normal (e esperado) que `Modules/Estoque/Consumers/` importe
`Shared.Contracts.IntegrationEvents.PedidoCriadoEvent` — o evento não fica
dentro do módulo Vendas, mora em `Modules/Shared/Contracts/IntegrationEvents/`
(`CONTRACTS/RULES.md` seção 5), exatamente para que o módulo Estoque possa
referenciá-lo sem precisar de `ProjectReference` para Vendas.

## 3. `RabbitMqConsumerBase` — o que já vem pronto de Infrastructure

A classe base, fornecida por `Infrastructure/Messaging`
(`MESSAGING/RULES.md` seção 3), cuida de tudo que é mecânico: consumir a
fila, deserializar o evento, checar idempotência, abrir/commitar a
transação, fazer ack/nack, e mover para DLQ após N tentativas. O `Consumer`
concreto do módulo implementa **só** o que fazer com o evento já
deserializado:

```csharp
public abstract class RabbitMqConsumerBase<TEvent> : BackgroundService
    where TEvent : IntegrationEvent
{
    protected RabbitMqConsumerBase(string queueName, string moduloSchema, IUnitOfWorkFactory unitOfWorkFactory);

    // implementado pela base: consome a fila, deserializa, verifica/grava em
    // <moduloSchema>.processed_messages usando um IUnitOfWork, chama HandleAsync,
    // commita, ack — em caso de exceção: Dispose() do IUnitOfWork reverte sozinho, nack, retry/DLQ
    protected abstract Task HandleAsync(TEvent @event, IUnitOfWork unitOfWork);
}
```

```csharp
internal class PedidoCriadoConsumer : RabbitMqConsumerBase<PedidoCriadoEvent>
{
    private readonly IServiceScopeFactory _scopeFactory;

    public PedidoCriadoConsumer(IUnitOfWorkFactory unitOfWorkFactory, IServiceScopeFactory scopeFactory)
        : base(queueName: "estoque.pedido-criado", moduloSchema: "estoque", unitOfWorkFactory)
    {
        _scopeFactory = scopeFactory;
    }

    protected override async Task HandleAsync(PedidoCriadoEvent @event, IUnitOfWork unitOfWork)
    {
        using var scope = _scopeFactory.CreateScope();
        var estoqueRepository = scope.ServiceProvider.GetRequiredService<IEstoqueRepository>();

        var estoque = await estoqueRepository.ObterPorProdutoAsync(@event.PedidoId, unitOfWork);
        estoque.Reservar(@event.Itens);
        await estoqueRepository.AtualizarAsync(estoque, unitOfWork);
    }
}
```

- **Visibilidade `internal`**: assim como o `Repository`/`Service` (implementação concreta), o `Consumer` concreto nunca é nomeado fora do próprio `Install()` do módulo — é `Activator`/DI que o instancia como `BackgroundService`, nenhum outro código o referencia por tipo (`ARCHITECTURE-RULES.md` seção 5.1). Só a classe base `RabbitMqConsumerBase<TEvent>`, em `Infrastructure`, precisa ser `public`.
- **Idempotência não é responsabilidade que o `Consumer` concreto implementa** — ela já é resolvida pela base, que recebe o `moduloSchema` e gerencia a tabela `processed_messages` genericamente (mesmo padrão do `IOutboxSchemaRegistry` em `MESSAGING/RULES.md` seção 6: o módulo se anuncia, a Infrastructure faz o trabalho mecânico).
- `HandleAsync` recebe o `IUnitOfWork` já aberto pela base — toda escrita feita dentro dele (via `Repository`/`Service`) usa esse mesmo `IUnitOfWork`, garantindo que a marca de "processado" e o efeito de negócio commitam atomicamente juntos.

### 3.1 `Consumer` é `Singleton` — dependências `Scoped` nunca vão no construtor

`RabbitMqConsumerBase<TEvent>` é um `BackgroundService`, registrado como
`Singleton` (`03-MODULES/RULES.md` seção 6). `Repository`, `Handler` e
`Service` são registrados como `Scoped`. O ASP.NET Core bloqueia (em tempo de
build, com validação de escopo ativa) um `Singleton` injetando um `Scoped`
diretamente no construtor — e mesmo sem essa validação ligada, o resultado
seria uma única instância do `Scoped` vivendo pelo tempo de vida inteiro da
aplicação, o que é exatamente o problema que `Scoped` existe para evitar.

**`IUnitOfWorkFactory` é exceção — pode ir direto no construtor do `Consumer`.**
Ele é `Singleton` (`DATABASE/RULES.md` seção 5: não tem estado próprio, só
fabrica objetos), então a base `RabbitMqConsumerBase` injeta e usa sem
nenhuma ginástica de escopo.

Para qualquer *outra* dependência `Scoped` — `Repository`, `Handler`
reaproveitado, `Service` — o `Consumer` concreto injeta `IServiceScopeFactory`
no construtor e cria um `scope` novo a cada mensagem processada, dentro de
`HandleAsync`, resolvendo a dependência a partir desse `scope`:

```csharp
using var scope = _scopeFactory.CreateScope();
var repository = scope.ServiceProvider.GetRequiredService<IAlgumRepository>();
```

**❌ Errado — `Repository` (`Scoped`) injetado direto no construtor do `Consumer` (`Singleton`):**

```csharp
internal class PedidoCriadoConsumer : RabbitMqConsumerBase<PedidoCriadoEvent>
{
    private readonly IEstoqueRepository _estoqueRepository; // ❌ Scoped preso num Singleton pra sempre

    public PedidoCriadoConsumer(IUnitOfWorkFactory unitOfWorkFactory, IEstoqueRepository estoqueRepository)
        : base("estoque.pedido-criado", "estoque", unitOfWorkFactory)
    {
        _estoqueRepository = estoqueRepository; // ❌ falha na validação de escopo do ASP.NET Core, ou pior, "funciona" com uma única instância vazando entre mensagens
    }
}
```

**✅ Correto — `IServiceScopeFactory` no construtor, escopo novo por mensagem (exemplo completo já na seção 3):**

```csharp
public PedidoCriadoConsumer(IUnitOfWorkFactory unitOfWorkFactory, IServiceScopeFactory scopeFactory)
    : base("estoque.pedido-criado", "estoque", unitOfWorkFactory) => _scopeFactory = scopeFactory;

protected override async Task HandleAsync(PedidoCriadoEvent @event, IUnitOfWork unitOfWork)
{
    using var scope = _scopeFactory.CreateScope();
    var estoqueRepository = scope.ServiceProvider.GetRequiredService<IEstoqueRepository>();
    // ...
}
```

## 4. Consumer é fino — delega para Handler ou Service

Mesma filosofia do `Controller` (`CONTROLLER/RULES.md`): `Consumer` traduz,
não decide. Duas opções, nesta ordem de preferência:

1. **Reaproveitar um `Handler` existente** — se reagir ao evento é
   equivalente a executar um `Command` que já existe no módulo, o `Consumer`
   monta esse `Command` a partir do evento e chama o `Handler`, exatamente
   como um `Controller` faria.
2. **Chamar um `Service` diretamente** — se a reação ao evento não se encaixa
   em nenhum `Command`/`Handler` já existente (é uma lógica exclusiva de
   reação a evento, sem equivalente síncrono), o `Consumer` chama um
   `Service` (`SERVICES/RULES.md`) que implementa essa lógica.

Em nenhum dos dois casos o `Consumer` implementa regra de negócio
diretamente dentro de `HandleAsync` além da tradução evento → chamada.

## 5. Nomenclatura

`<EventoNoPassado>Consumer` — mesmo nome do evento que consome, sufixado com
`Consumer` (ex: evento `PedidoCriadoEvent` → `PedidoCriadoConsumer`). Se um
módulo precisa reagir ao mesmo evento de duas formas independentes, são dois
`Consumers` distintos (duas queues, ver `MESSAGING/RULES.md` seção 8), não um
`Consumer` com dois comportamentos condicionais.

## 6. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| `Consumer` chamando `Repository`/`Entity` de outro módulo (além do `IntegrationEvent` recebido) | Só o evento é superfície pública — o resto do módulo publicador continua privado |
| `Consumer` implementando regra de negócio pesada direto em `HandleAsync` | Deveria delegar para `Handler` (reaproveitado) ou `Service` |
| `Consumer` gerenciando ack/nack ou retry manualmente, contornando a base | Reintroduz risco de processamento duplicado ou perda de mensagem — a base já resolve isso |
| Um `Consumer` cobrindo mais de um tipo de evento com `switch` interno | Quebra a convenção de um `Consumer` por evento (seção 5) |
| `Consumer` sem passar `moduloSchema` correto no construtor da base | Idempotência passaria a checar/gravar no schema errado |
| `Consumer` injetando `Repository`/`Handler`/`Service` (`Scoped`) direto no construtor | `Consumer` é `Singleton` — DI recusa a montar (ou, sem validação, prende a instância `Scoped` para sempre). Usar `IServiceScopeFactory` (seção 3.1). `IUnitOfWorkFactory` é a única exceção — é `Singleton`, injeta direto |

## 7. Enforcement

- Code review confere que todo `Consumer` novo herda de `RabbitMqConsumerBase<TEvent>` e nunca implementa consumo de fila manualmente.
- Code review bloqueia `using` de `Entities`/`Repositories`/`Services` de outro módulo dentro de `Consumers/` — só `Shared.Contracts` (incluindo `Shared.Contracts.IntegrationEvents`) é permitido, e é o único que compila sem `ProjectReference` para o módulo publicador.
