# RULES — Test / Integration

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md`, `03-MODULES/RULES.md` e `02-INFRASTRUCTURE/*/RULES.md`. Este documento especializa, nunca substitui.

## 1. Decisão técnica desta camada

| Decisão | Escolha |
|---|---|
| Ambiente de infraestrutura real | **Testcontainers** (SQL Server, RabbitMQ, Redis sobem em Docker por suíte de teste) |

## 2. Missão

`Integration` testa o módulo de ponta a ponta contra dependências **reais**
(não mockadas): `Repository` executando SQL de verdade contra um SQL Server
real com o schema do módulo aplicado, o fluxo completo de Outbox → RabbitMQ
→ `Consumer` com idempotência real, e `ICacheService` contra um Redis real.

## 3. Fixture — subida e derrubada dos containers

Um `Fixture` compartilhado (via `IClassFixture<T>`/`ICollectionFixture<T>`
do xUnit) sobe os containers uma única vez por suíte de teste do módulo, não
por teste individual:

```csharp
public class VendasIntegrationFixture : IAsyncLifetime
{
    public MsSqlContainer SqlContainer { get; } = new MsSqlBuilder().Build();
    public RabbitMqContainer RabbitContainer { get; } = new RabbitMqBuilder().Build();
    public RedisContainer RedisContainer { get; } = new RedisBuilder().Build();

    public async Task InitializeAsync()
    {
        await Task.WhenAll(SqlContainer.StartAsync(), RabbitContainer.StartAsync(), RedisContainer.StartAsync());
        await MigrationRunner.AplicarScriptsDoModulo("vendas", SqlContainer.GetConnectionString());
    }

    public async Task DisposeAsync() =>
        await Task.WhenAll(SqlContainer.DisposeAsync().AsTask(), RabbitContainer.DisposeAsync().AsTask(), RedisContainer.DisposeAsync().AsTask());
}
```

- As migrations (`DATABASE/RULES.md` seção 9) do **próprio módulo** são aplicadas no `InitializeAsync` do `Fixture` — os scripts reais de `Migrations/Scripts/<schema>/`, não uma versão simplificada só para teste.
- Um `Fixture` por módulo — testes de integração de módulos diferentes nunca compartilham containers, para não vazar estado entre suítes de módulos diferentes.

### 3.1 Instanciando um `Repository`/`Consumer` `internal` diretamente

Diferente do Unit test (que majoritariamente mocka interfaces `public`), o
Integration test frequentemente precisa instanciar a implementação
**concreta** de `Repository` (`internal` — `REPOSITORIES/RULES.md` seção 2)
para executar SQL de verdade, ou o `Consumer` concreto (`internal` —
`CONSUMERS/RULES.md` seção 3) para validar o fluxo completo de mensageria.
Isso só compila se o módulo sob teste declara, no seu `AssemblyInfo.cs`:

```csharp
[assembly: InternalsVisibleTo("<NomeModulo>.IntegrationTests")]
```

Sem essa linha, `new PedidoRepository(connectionFactory)` dentro do projeto
`Vendas.IntegrationTests` falha em tempo de compilação (`CS0122`) — o tipo é
`internal` e o projeto de teste é um assembly diferente. Ver
`ARCHITECTURE-RULES.md` seção 5.1 para a lista completa de camadas que ficam
`internal` e por quê.

## 4. Isolamento entre testes dentro da mesma suíte

Cada teste é responsável por não deixar estado que afete o próximo:

- Preferência: cada teste abre sua própria transação e faz `Rollback` no final (mesmo se a asserção já leu os dados dentro da transação) — mais rápido que `TRUNCATE` entre testes.
- Quando o cenário exige testar o `Commit` de fato (ex: validar que o `OutboxPublisherService` publicou depois do commit), o teste limpa explicitamente as linhas que criou ao final (`try/finally` ou `IAsyncLifetime` por teste).
- Testes de integração **não dependem de ordem de execução** — cada um monta seu próprio cenário do zero, nunca assume dado deixado por um teste anterior.

## 5. O que testar aqui (e o que não)

| Cenário | Testar em Integration? |
|---|---|
| `Repository` executando query real contra o schema do módulo | Sim |
| Fluxo `Handler` grava estado + Outbox → `OutboxPublisherService` → RabbitMQ → `Consumer` idempotente | Sim |
| `ICacheService` com TTL real expirando | Sim, quando o comportamento de expiração é relevante ao teste |
| Regra de validação de negócio isolada do `Handler` | Não — já coberta em `Unit`, reexecutar aqui é redundante e mais lento |
| Forma de `Dto`/`IntegrationEvent` | Não — coberta em `Contract` |

## 6. CI — estágio separado

Testes de `Integration` **não rodam no mesmo estágio de CI que `Unit`/`Contract`**
— sobem containers Docker, são naturalmente mais lentos. Rodam em um estágio
separado (ex: antes de merge para a branch principal, ou em pipeline
dedicado), sem bloquear o feedback rápido que `Unit`/`Contract` fornecem a
cada commit.

## 7. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| Teste de integração dependendo de dado deixado por outro teste | Quebra quando a ordem de execução muda ou quando rodado isoladamente |
| Containers compartilhados entre módulos diferentes | Vaza estado/schema entre módulos que deveriam ser isolados mesmo em teste |
| Revalidar regra de negócio já coberta em `Unit` | Redundante e mais lento sem ganho de cobertura real |
| Integration test rodando no mesmo estágio de CI que bloqueia todo commit | Deixa o ciclo de feedback do desenvolvedor lento sem necessidade |

## 8. Enforcement

- CI roda `Integration` em estágio dedicado, com timeout generoso para subida de containers.
- Falha de `Integration` bloqueia merge para a branch principal, mesmo não bloqueando todo commit intermediário.
