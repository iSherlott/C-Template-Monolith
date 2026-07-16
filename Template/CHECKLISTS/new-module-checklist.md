# CHECKLIST — Criar um módulo novo

Checklist operacional, passo a passo, para nascer um módulo do zero até
pronto para produção. Cada item referencia o `RULES.md` onde a regra
completa está detalhada — este documento não substitui aqueles, só
consolida a ordem de execução. Usado principalmente pelo
[`module-agent`](../AGENTS/module-agent.md), coordenado pelo
[`ORCHESTRATOR`](../AGENTS/ORCHESTRATOR.md).

## Antes de começar

- [ ] O domínio passa a heurística de "isto merece ser um módulo?" ([`03-MODULES/RULES.md`](../03-MODULES/RULES.md) §8)
- [ ] O nome escolhido não colide com schema, prefixo de cache ou exchange de nenhum módulo já existente ([`03-MODULES/RULES.md`](../03-MODULES/RULES.md) §3)
- [ ] Está claro se o módulo depende de outro já existente (via `Contracts`) e se publica/consome eventos

## 1. Banco de dados — `database-agent`

- [ ] Pasta `Migrations/Scripts/<schema>/` criada
- [ ] `0001_create_schema_<modulo>.sql` — sempre o primeiro script ([`DATABASE/RULES.md`](../02-INFRASTRUCTURE/DATABASE/RULES.md) §9)
- [ ] Se o módulo publica eventos: tabela `<schema>.outbox_messages` ([`MESSAGING/RULES.md`](../02-INFRASTRUCTURE/MESSAGING/RULES.md) §6)
- [ ] Se o módulo consome eventos: tabela `<schema>.processed_messages` ([`MESSAGING/RULES.md`](../02-INFRASTRUCTURE/MESSAGING/RULES.md) §9)

## 2. Estrutura de pastas — `module-agent`

- [ ] `Modules/<NomeModulo>/<NomeModulo>.csproj` criado, referenciando só `Infrastructure` e `Shared` (nunca outro módulo)
- [ ] `Modules/<NomeModulo>/` criado com as 8 subpastas: `Controller`, `Handler`, `Contracts` (só `Dtos/` — `I<NomeModulo>`/`IntegrationEvents` vão em `Shared/Contracts`, ver §7 abaixo), `Entities`, `Commands`, `Repositories`, `Consumers`, `Services` ([`03-MODULES/RULES.md`](../03-MODULES/RULES.md) §2)
- [ ] `GlobalUsings.cs` com os `using` comuns do módulo (`Infrastructure.Database`, `Shared.Kernel`, etc.)
- [ ] `AssemblyInfo.cs` com `[assembly: InternalsVisibleTo(...)]` para `<NomeModulo>.UnitTests`, `<NomeModulo>.IntegrationTests` e `DynamicProxyGenAssembly2` (necessário para testar/mockar os tipos `internal` do módulo — ver §15)

## 3. Entities

- [ ] Aggregate Root(s) definida(s), herdando `Entity`/`AggregateRoot` de `Shared/Kernel`
- [ ] Construtor privado + factory method estático (`Criar(...)`)
- [ ] Invariante de domínio lança `DomainException`, nunca `Result.Failure` dentro da própria `Entity`
- [ ] ([`ENTITIES/RULES.md`](../03-MODULES/ENTITIES/RULES.md))

## 4. Commands

- [ ] Um `Command`/`Query` por caso de uso, `record` imutável
- [ ] Nomenclatura: `<Verbo><Recurso>Command` / `<Verbo><Recurso>Query`
- [ ] ([`COMMANDS/RULES.md`](../03-MODULES/COMMANDS/RULES.md))

## 5. Repositories

- [ ] Interface em `Repositories/Contracts/` (`public`, pasta local — não confundir com `Shared/Contracts`) + implementação concreta na raiz de `Repositories/` (`internal`), ambas privadas ao módulo em termos de quem referencia (`ARCHITECTURE-RULES.md` §5.1, `REPOSITORIES/RULES.md` §2)
- [ ] Métodos de escrita recebem `IUnitOfWork` explícito (nunca `IDbConnection`/`IDbTransaction` crus)
- [ ] Toda query qualifica `<schema-do-modulo>.<tabela>` — nunca cross-schema
- [ ] `Repository` só da Aggregate Root, nunca de entidade filha
- [ ] ([`REPOSITORIES/RULES.md`](../03-MODULES/REPOSITORIES/RULES.md))

## 6. Handlers

- [ ] Um `Handler` por Aggregate Root/recurso, implementando `IHandler<TCommand,TResult>` uma vez por `Command`/`Query` aceito — nunca um `Handler` por `Command`/`Query` individual
- [ ] Segundo `Handler` só criado se o módulo expõe outro substantivo distinto sob o mesmo `Controller` (critério "banana vs. tomate" — `HANDLER/RULES.md` seção 4)
- [ ] Retorno sempre `Task<Result<TDto>>`
- [ ] Nunca instancia `IDbConnection`/`IDbTransaction` diretamente — só pede `IUnitOfWork` a `IUnitOfWorkFactory`
- [ ] Falha de negócio esperada vira `Result.Failure(Error)`; erro de infraestrutura propaga (não é capturado como `Result.Failure`)
- [ ] ([`HANDLER/RULES.md`](../03-MODULES/HANDLER/RULES.md))

## 7. Contracts

- [ ] `I<NomeModulo>` criado em `Modules/Shared/Contracts/I<NomeModulo>.cs` (não dentro do módulo) — expõe só o que outro módulo genuinamente precisa
- [ ] `Dtos` (em `Modules/<NomeModulo>/Contracts/Dtos/`) imutáveis (`record`), sem nenhum campo do tipo `Entity`
- [ ] `IntegrationEvents` criados em `Modules/Shared/Contracts/IntegrationEvents/` (não dentro do módulo), herdando `IntegrationEvent` (de `Infrastructure/Messaging`), nome no particípio passado
- [ ] ([`CONTRACTS/RULES.md`](../03-MODULES/CONTRACTS/RULES.md))

## 8. Controller

- [ ] Rota `api/<schema>/<recurso>`
- [ ] `Handler` injetado direto no construtor — sem service locator
- [ ] Resposta sempre via `Result.ToActionResult()`
- [ ] ([`CONTROLLER/RULES.md`](../03-MODULES/CONTROLLER/RULES.md))

## 9. Consumers — se o módulo consome eventos

- [ ] Cada `Consumer` (classe `internal`) herda `RabbitMqConsumerBase<TEvent>`
- [ ] Importa o `IntegrationEvent` de `Shared.Contracts.IntegrationEvents`, não do módulo publicador
- [ ] `moduloSchema` correto passado no construtor (idempotência aponta para o schema certo)
- [ ] `HandleAsync` delega para `Handler` reaproveitado ou `Service`, nunca implementa regra de negócio direto
- [ ] ([`CONSUMERS/RULES.md`](../03-MODULES/CONSUMERS/RULES.md))

## 10. Services — se aplicável

- [ ] `<NomeModulo>ModuleFacade` (classe `internal`) implementando `I<NomeModulo>` (de `Shared.Contracts`), se o módulo expõe fachada síncrona
- [ ] Lógica extraída para `Service` só quando existe um segundo uso real (não antecipado)
- [ ] ([`SERVICES/RULES.md`](../03-MODULES/SERVICES/RULES.md))

## 11. Shared — só consumo

- [ ] Nenhuma adição a `Modules/Shared/Kernel` ou `Helpers` específica deste módulo — se parecer necessário, a lógica fica dentro do próprio módulo
- [ ] ([`SHARED/RULES.md`](../03-MODULES/SHARED/RULES.md))

## 11.1 Messages

- [ ] `Modules/<NomeModulo>/Messages/<NomeModulo>Messages.resx` criado com toda mensagem de `Error.Validation`/`Error.NotFound`/`Error.Conflict`/`DomainException` do módulo — nenhuma string literal de mensagem de usuário direto no `Handler`/`Entity`
- [ ] `Modules/<NomeModulo>/Messages/<NomeModulo>Messages.cs` (classe acessadora `internal`, escrita à mão) criado junto — o `.resx` sozinho **não** gera classe fortemente tipada com `dotnet build`
- [ ] Chave semântica (`PedidoSemItens`), nunca posicional/numérica ou o texto inteiro como chave
- [ ] ([`MESSAGES/RULES.md`](../03-MODULES/MESSAGES/RULES.md))

## 12. `<NomeModulo>ModuleInstaller.cs` — composição

- [ ] Implementa `IModuleInstaller` (`Modules/Shared`), classe `public` com construtor parameterless
- [ ] Registra `Handlers`, `Repositories`, `Services` (`Scoped`)
- [ ] Registra `Consumers` como `IHostedService` (`Singleton`)
- [ ] Registra `Controllers` como application part
- [ ] Registra o schema em `IOutboxSchemaRegistry`, se publica eventos
- [ ] Lê apenas sua própria seção `Modules:<NomeModulo>`
- [ ] `Use(IApplicationBuilder)` só sobrescrito se o módulo precisa de algo no pipeline HTTP — senão, herda o default vazio da interface
- [ ] ([`03-MODULES/RULES.md`](../03-MODULES/RULES.md) §6)

## 13. Messaging — `messaging-agent`, se aplicável

- [ ] Exchange `<schema>.events` existe
- [ ] Toda queue de consumo tem DLQ correspondente (`<queue>.dlq`)
- [ ] ([`MESSAGING/RULES.md`](../02-INFRASTRUCTURE/MESSAGING/RULES.md) §8)

## 14. Host — `host-agent`

- [ ] Nada a tocar em `Program.cs` — o módulo é descoberto sozinho por ter `IModuleInstaller` e estar referenciado no `.csproj` do Host (`01-HOST/RULES.md` §3)
- [ ] Seção `Modules:<NomeModulo>` presente em `appsettings`
- [ ] Confirmar que o módulo aparece registrado ao subir a aplicação (log/health check) — se não aparecer, checar se `ModuleInstaller` é `public` com construtor parameterless

## 15. Testes — `test-agent`

- [ ] `Unit`: um teste de sucesso + um por `ErrorType` de falha esperada, por `Handler`; invariantes testados direto na `Entity`
- [ ] `Contract`: teste de forma para todo `Dto`/`IntegrationEvent` publicado
- [ ] `Integration`: `Fixture` com Testcontainers, migrations reais do módulo aplicadas, `Repository` e fluxo de Outbox testados contra infraestrutura real
- [ ] ([`04-TEST/`](../04-TEST/))

## 16. Validação final

- [ ] `Test/Architecture/` passa — nenhuma referência indevida a `Entity`/`Repository` de outro módulo
- [ ] Nomenclatura consistente em todo lugar: nome do módulo = schema = prefixo de cache = exchange = seção de configuração = classe `<Nome>ModuleInstaller` ([`03-MODULES/RULES.md`](../03-MODULES/RULES.md) §3)
- [ ] Nenhum segredo versionado em `appsettings.json`

Só depois de todos os itens marcados o módulo está pronto para revisão final
antes do merge.
