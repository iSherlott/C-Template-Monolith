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
| `dotnet test` (Integration — exige Docker/Testcontainers) | Não executado nesta verificação (Docker indisponível no ambiente); ver seção 3 sobre a lacuna conhecida |
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
| `Test/<N>/Integration/` | `Test/Pessoas/Integration/` (ver seção 3 sobre o que de fato é validado aqui) |
| `Test/Architecture/` | `Test/Architecture/` |

**Nota sobre `Infrastructure/Database`:** o `NODE-MAP.md` usa
`Implementation/` como nome de subpasta genérico no catálogo; no
`AnimeList` real essa subpasta se chama `Factory/` (`DATABASE/RULES.md`
§3.1 explica o porquê: `UnitOfWork`/`UnitOfWorkFactory` moram junto do que
os cria). Isso não é uma divergência — é o próprio `DATABASE/RULES.md`
definindo o nome específico para essa camada; `NODE-MAP.md` generaliza
porque `Cache`/`Messaging` usam `Implementation/` como nome literal.

## 3. Lacuna conhecida — Integration test de Messaging/Cache

`04-TEST/INTEGRATION/RULES.md` §5 documenta como escopo pleno testar o
fluxo `Handler` grava estado + Outbox → `OutboxPublisherService` →
RabbitMQ → `Consumer` idempotente, e `ICacheService` com TTL real
expirando, ambos contra infraestrutura real via Testcontainers.

**O `AnimeList` real não valida essa parte.** Só `Testcontainers.MsSql`
está referenciado nos projetos de `Integration` (`LIBRARIES.md` §2) — o
que é de fato testado contra infraestrutura real é:

- `Repository` executando SQL real contra o schema do módulo (`SqlServer`
  via Testcontainers) — **coberto**, ex: `PessoaRepositoryTests`.
- Que `IEventBus.PublishAsync` grava a linha certa na tabela
  `outbox_messages`, na mesma transação — **coberto**, ver
  `EventBusOutboxTests` em `Test/Watchlist/Integration/`.
- Que `OutboxPublisherService` de fato publica no RabbitMQ real, e que
  `ProgressoAtualizadoConsumer` (o único `Consumer` real do projeto)
  processa e marca idempotência corretamente contra um broker real —
  **não coberto**. Não existe `Testcontainers.RabbitMq` em nenhum
  `.csproj`.
- `ICacheService` contra um Redis real, TTL expirando de fato — **não
  coberto**. Não existe `Testcontainers.Redis` em nenhum `.csproj`.

Um projeto novo que precisa dessa cobertura completa não está "quebrando
o padrão" ao adicionar `Testcontainers.RabbitMq`/`Testcontainers.Redis` —
está fechando uma lacuna que o próprio `AnimeList` deixou aberta. Ainda
assim, a adição desses pacotes segue o processo normal de
`LIBRARIES.md` §3 (perguntar antes, não silenciosamente).

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
