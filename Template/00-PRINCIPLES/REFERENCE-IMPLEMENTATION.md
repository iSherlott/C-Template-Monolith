# REFERENCE-IMPLEMENTATION — AnimeList como espelho validado

> Herda [`ARCHITECTURE-RULES.md`](ARCHITECTURE-RULES.md) e
> [`NODE-MAP.md`](NODE-MAP.md). Este arquivo não cria regra nova — mapeia
> cada nó do catálogo de `NODE-MAP.md` para um arquivo **real, compilado e
> testado** no projeto `AnimeList`, para que qualquer projeto novo (mesmo
> de domínio totalmente diferente — e-commerce, saúde, logística) tenha um
> exemplo concreto para comparar lado a lado, não só a descrição em prosa.

`AnimeList` é um projeto de exemplo (anime tracking) — o domínio em si
(`Pessoa`, `Anime`, `Watchlist`) é irrelevante para quem constrói outro
tipo de projeto. O que importa, e é o que este arquivo garante, é que a
**estrutura física** (pastas, nomes de arquivo, assinatura de contrato)
seja idêntica independente do domínio. Se um projeto novo tem módulos
chamados `Pedido`/`Estoque`/`Pagamento`, o `Handler` de `Pedido` deve ter
exatamente a mesma forma que `PedidoHandler`/`PessoaHandler` tem aqui —
só o nome e a regra de negócio mudam.

## 1. Validação — status na data abaixo

Verificado em `2026-07-19` contra `D:\My_Code\C#\AnimeList`:

| Verificação | Resultado |
|---|---|
| `dotnet build` | Limpo — 0 erros, 0 avisos |
| `dotnet test` (Unit + Contract + Architecture — não exigem Docker) | 100% verde |
| `dotnet test` (Integration — exige Docker/Testcontainers) | Código implementado e revisado (seção 3 — gap fechado), mas não executado nesta verificação: Docker indisponível no ambiente (`DockerUnavailableException` — falha idêntica à que os testes de SQL Server já existentes também têm neste ambiente, não é defeito do código novo). **Pendente: rodar `dotnet test` com Docker Desktop ativo para confirmar execução real antes de considerar 100% fechado.** |
| `grep` por `IMediator`/`ISender`/`.SendAsync(`/`IRepositoryBase`/`RepositoryBase<` em todo o código | Zero ocorrências |
| Estrutura física de módulo (`Pessoas` usado como amostra) | Idêntica ao diagrama de `03-MODULES/RULES.md` §2 e ao catálogo de `NODE-MAP.md` §2, arquivo por arquivo |
| Estrutura física do `Host` | Idêntica a `01-HOST/RULES.md` §2 |

## 2. Mapa nó → arquivo real

Mesma ordem de `NODE-MAP.md` §2. `<N>` nos exemplos abaixo foi substituído
pelo módulo real usado como amostra (`Pessoas`); qualquer outro módulo do
`AnimeList` (`Catalogo`, `Watchlist`, `Match`, `Auth`) segue exatamente o
mesmo padrão.

| Nó (`NODE-MAP.md`) | Exemplo real no `AnimeList` |
|---|---|
| `Host/Program.cs` | `Host/Program.cs` |
| `Host/DependencyInjection/*Extensions.cs` | `Host/DependencyInjection/ModuleRegistration.cs`, `Host/DependencyInjection/SwaggerExtensions.cs` |
| `Host/Properties/launchSettings.json` | `Host/Properties/launchSettings.json` |
| `Infrastructure/Database/{Interface,Implementation,Map,Migrations}` | `Infrastructure/Database/Interface/`, `Infrastructure/Database/Factory/` (implementação — ver nota abaixo), `Infrastructure/Database/Map/`, `Infrastructure/Database/Migrations/` |
| `Infrastructure/Messaging/{Interface,Implementation,RabbitMq,Outbox}` | `Infrastructure/Messaging/Interface/IEventBus.cs`, `Infrastructure/Messaging/Implementation/EventBus.cs`, `Infrastructure/Messaging/RabbitMq/`, `Infrastructure/Messaging/Outbox/` |
| `Infrastructure/Cache/{Interface,Implementation}` | `Infrastructure/Cache/Interface/ICacheService.cs`, `Infrastructure/Cache/Implementation/RedisCacheService.cs` |
| `Infrastructure/Security/{Interface,Implementation,Exceptions,Extensions}` | `Infrastructure/Security/Implementation/Pbkdf2PasswordHasher.cs`, `Infrastructure/Security/Extensions/SecurityExtensions.cs` |
| `Modules/Shared/Kernel/` | `Modules/Shared/Kernel/` (`Result.cs`, `Error.cs`, `Entity.cs`, `AggregateRoot.cs`, `IHandler.cs`) |
| `Modules/Shared/Contracts/` | `Modules/Shared/Contracts/` (`I<Modulo>.cs` por módulo + `IntegrationEvents/`) |
| `Modules/<N>/<N>.csproj` | `Modules/Pessoas/Pessoas.csproj` |
| `Modules/<N>/<N>ModuleInstaller.cs` | `Modules/Pessoas/PessoasModuleInstaller.cs` |
| `Modules/<N>/Controller/` | `Modules/Pessoas/Controller/PessoasController.cs` |
| `Modules/<N>/Handler/` | `Modules/Pessoas/Handler/PessoaHandler.cs` — exemplo de Handler consolidado cobrindo `CriarPessoaCommand` + `ObterPessoaPorIdQuery` na mesma classe; `Modules/Catalogo/Handler/{GeneroHandler,AnimeHandler}.cs` é o exemplo real do critério "banana vs. tomate" (`HANDLER/RULES.md` §4) — dois Handlers no mesmo módulo porque `CatalogoController` expõe dois substantivos distintos |
| `Modules/<N>/Contracts/Dtos/` | `Modules/Pessoas/Contracts/Dtos/PessoaDto.cs` |
| `Modules/<N>/Contracts/Repositories/I<N>Repository.cs` | `Modules/Pessoas/Contracts/Repositories/IPessoaRepository.cs` |
| `Modules/<N>/Repository/<N>Repository.cs` | `Modules/Pessoas/Repository/PessoaRepository.cs` |
| `Modules/<N>/Entities/` | `Modules/Pessoas/Entities/Pessoa.cs` |
| `Modules/<N>/Commands/` | `Modules/Pessoas/Commands/{CriarPessoaCommand,ObterPessoaPorIdQuery}.cs` |
| `Modules/<N>/Consumers/` | `Modules/Match/Consumers/ProgressoAtualizadoConsumer.cs` (único módulo com `Consumer` real no `AnimeList`) |
| `Modules/<N>/Services/` | `Modules/Pessoas/Services/PessoasModuleFacade.cs` |
| `Modules/<N>/Dictionary/<N>Dictionary.cs` + `.resx` | `Modules/Pessoas/Dictionary/PessoasDictionary.cs` + `.resx` |
| `Test/<N>/Unit/` | `Test/Pessoas/Unit/` |
| `Test/<N>/Contract/` | `Test/Pessoas/Contract/` |
| `Test/<N>/Integration/` | `Test/Pessoas/Integration/` (SQL real); `Test/Watchlist/Integration/RabbitMqOutboxPublishingTests.cs` e `Test/Match/Integration/{ProgressoAtualizadoConsumerTests,RedisCacheServiceTests}.cs` (RabbitMQ/Redis reais — ver seção 3) |
| `Test/Architecture/` | `Test/Architecture/` |

**Nota sobre `Infrastructure/Database`:** o `NODE-MAP.md` usa
`Implementation/` como nome de subpasta genérico no catálogo; no
`AnimeList` real essa subpasta se chama `Factory/` (`DATABASE/RULES.md`
§3.1 explica o porquê: `UnitOfWork`/`UnitOfWorkFactory` moram junto do que
os cria). Isso não é uma divergência — é o próprio `DATABASE/RULES.md`
definindo o nome específico para essa camada; `NODE-MAP.md` generaliza
porque `Cache`/`Messaging` usam `Implementation/` como nome literal.

## 3. Gap de Integration test de Messaging/Cache — fechado em `2026-07-19`

`04-TEST/INTEGRATION/RULES.md` §5 documenta como escopo pleno testar o
fluxo `Handler` grava estado + Outbox → `OutboxPublisherService` →
RabbitMQ → `Consumer` idempotente, e `ICacheService` com TTL real
expirando, ambos contra infraestrutura real via Testcontainers.
Originalmente o `AnimeList` só validava o lado SQL disso. Fechado a
pedido explícito do dev — `Testcontainers.RabbitMq`/`Testcontainers.Redis`
adicionados via `LIBRARIES.md` §3 (confirmação explícita, não silenciosa).

| O que é testado | Onde | Cobertura |
|---|---|---|
| `Repository` executando SQL real contra o schema do módulo | `Test/Pessoas/Integration/PessoaRepositoryTests.cs` e equivalentes | Já cobria |
| `IEventBus.PublishAsync` grava a linha certa em `outbox_messages`, mesma transação | `Test/Watchlist/Integration/EventBusOutboxTests.cs` | Já cobria |
| `OutboxPublisherService` publica de fato no RabbitMQ real (consumidor de verificação recebe a mensagem, `processed_on` é marcado) | `Test/Watchlist/Integration/RabbitMqOutboxPublishingTests.cs` | **Novo** |
| `ProgressoAtualizadoConsumer` processa mensagem recebida de um broker RabbitMQ real e grava `match.pessoa_anime_assistido` | `Test/Match/Integration/ProgressoAtualizadoConsumerTests.cs` | **Novo** |
| Idempotência do `Consumer` contra redelivery real (mesmo `event_id` entregue duas vezes → processado uma vez) | `Test/Match/Integration/ProgressoAtualizadoConsumerTests.cs` (`HandleAsync_DeveSerIdempotente_...`) | **Novo** |
| `ICacheService`/`RedisCacheService` — cache-aside real (segunda chamada não recalcula) e TTL expirando de verdade | `Test/Match/Integration/RedisCacheServiceTests.cs` | **Novo** |

**Detalhes de implementação que um projeto novo replicando este padrão
precisa saber** (não óbvios pela simples leitura do `RULES.md`):

- `WatchlistIntegrationFixture`/`MatchIntegrationFixture` agora sobem os
  três containers em paralelo (`Task.WhenAll`) — subir sequencialmente
  seria correto mas desnecessariamente mais lento por suíte.
- `RabbitMqOptions` (`HostName`/`Port`/`UserName`/`Password`) é montado a
  partir de `new Uri(rabbitContainer.GetConnectionString())` — o
  container expõe uma URI `amqp://user:pass@host:port/`, não campos
  separados.
- O teste de publish real declara sua própria fila de verificação
  (`test.verificacao-outbox-publishing`) e consome com `RabbitMQ.Client`
  puro — não reaproveita `RabbitMqConsumerBase<TEvent>` porque o objetivo
  é observar a mensagem chegando no exchange, não processar um `Command`.
- `OutboxPublisherService`/`ProgressoAtualizadoConsumer` são
  `BackgroundService` — o teste chama `StartAsync(CancellationToken.None)`
  diretamente (não precisa de `IHost` completo) e sempre `StopAsync` num
  `finally`, com polling (não `Thread.Sleep` fixo) até a condição
  esperada aparecer no banco, com timeout.
- Teste de idempotência do `Consumer` publica o **mesmo evento duas
  vezes** e confirma `COUNT(*) FROM match.processed_messages WHERE
  event_id = @Id` continua `1` — essa é a prova real de que a checagem de
  `RabbitMqConsumerBase` (`CONSUMERS/RULES.md` §3) funciona contra um
  broker de verdade, não só em teoria.

**Não executado neste ambiente** (Docker indisponível) — ver seção 1.
`dotnet build` confirma que o código compila e usa a API correta dos
pacotes `Testcontainers.RabbitMq`/`Testcontainers.Redis` `4.13.0`; rodar
`dotnet test` com Docker Desktop ativo é o passo que falta para fechar a
validação por completo.

## 4. Como usar este arquivo num projeto de domínio diferente

```
1. Identificar o nó que você está construindo (ex: "preciso do Handler do
   módulo Pedido") na tabela de NODE-MAP.md §2.

2. Abrir o arquivo real correspondente listado na seção 2 acima
   (ex: Modules/Pessoas/Handler/PessoaHandler.cs) e usar como gabarito
   físico: mesma assinatura de classe, mesmo padrão de injeção de
   dependência, mesmo formato de retorno (Result<Dto>).

3. Substituir o domínio (Pessoa → Pedido), nunca a estrutura. Se o
   arquivo real e o RULES.md correspondente parecerem divergir em algum
   detalhe, o RULES.md vence (ele é a fonte de verdade — AnimeList é a
   prova de que funciona, não a definição da regra) — mas reporte a
   divergência, pode ser um sinal de que o RULES.md ficou desatualizado.
```
