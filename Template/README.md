# Template — Governança de Arquitetura (Modular Monolith)

Esta pasta é o **livro de regras canônico** para projetar, construir e manter
qualquer projeto que siga a arquitetura Modular Monolith definida aqui. Não é
código de um projeto — é a fonte única de verdade que qualquer IA ou pessoa
consulta *antes* de tocar em código de um projeto real que implemente essa
arquitetura.

Este repositório (`Personal Governance`) é a origem centralizada. Projetos que
adotam essa arquitetura **referenciam** este rulebook — não copiam essas
pastas para dentro de si.

## Como usar esta pasta

Ordem de leitura obrigatória antes de qualquer trabalho:

1. **[`00-PRINCIPLES/ARCHITECTURE-RULES.md`](00-PRINCIPLES/ARCHITECTURE-RULES.md)** — sempre primeiro. Define as regras globais que todas as outras camadas herdam.
2. **[`00-PRINCIPLES/NODE-MAP.md`](00-PRINCIPLES/NODE-MAP.md)** — catálogo rápido de todo nó (arquivo/pasta) desta arquitetura, grafo de dependência, e métricas objetivas para conferir se um projeto real bate com o rulebook. Ponto de entrada recomendado para qualquer IA antes de tocar em um projeto real — mais rápido que ler os 20 `RULES.md` individuais.
3. O `RULES.md` da camada específica que você vai tocar (`01-HOST`, `02-INFRASTRUCTURE/...`, `03-MODULES/...`, `04-TEST/...`).
4. O arquivo do papel correspondente em `AGENTS/`, se você estiver atuando como um papel especializado (ex: construindo só a camada de Database, adote `AGENTS/database-agent.md`).
5. O checklist relevante em `CHECKLISTS/`, se a tarefa for criar algo novo do zero (ex: um módulo novo).

Nenhuma regra de camada específica pode contradizer `ARCHITECTURE-RULES.md`. Se
parecer necessário, a mudança é na regra global primeiro, com decisão
explícita — não uma exceção silenciosa numa pasta.

## Cenários de uso — qual playbook seguir

| Situação do projeto real | Playbook |
|---|---|
| Projeto novo, do zero | `AGENTS/ORCHESTRATOR.md` — "Sequência: projeto novo, do zero" |
| Feature nova em módulo já conforme | `AGENTS/ORCHESTRATOR.md` — "Sequência: feature nova em módulo existente" |
| Projeto já tem parte migrada/estruturada nesta arquitetura, mas diverge do mapa | `AGENTS/MIGRATION-AGENT.md` — "Cenário A" |
| Migrar projeto de outra stack (ex: Java) para esta arquitetura | `AGENTS/MIGRATION-AGENT.md` — "Cenário B" |

`00-PRINCIPLES/NODE-MAP.md` seção 4 detalha esta mesma tabela com contexto
adicional.

## Mapa da pasta

```
Template/
├── README.md                        # este arquivo — índice mestre
├── 00-PRINCIPLES/
│   ├── ARCHITECTURE-RULES.md        # ✅ regras globais (dependência, isolamento, comunicação)
│   └── NODE-MAP.md                  # ✅ catálogo de nós, grafo de dependência, métricas de sanidade
├── 01-HOST/
│   └── RULES.md                     # ✅ criado
├── 02-INFRASTRUCTURE/
│   ├── DATABASE/RULES.md            # ✅ criado
│   ├── MESSAGING/RULES.md           # ✅ criado
│   ├── CACHE/RULES.md               # ✅ criado
│   └── SECURITY/RULES.md            # ✅ criado — JWT, autorização por grupo, exceptions
├── 03-MODULES/
│   ├── RULES.md                     # ✅ criado
│   ├── CONTROLLER/RULES.md          # ✅ criado
│   ├── HANDLER/RULES.md             # ✅ criado
│   ├── CONTRACTS/RULES.md           # ✅ criado
│   ├── ENTITIES/RULES.md            # ✅ criado
│   ├── COMMANDS/RULES.md            # ✅ criado
│   ├── REPOSITORIES/RULES.md        # ✅ criado
│   ├── CONSUMERS/RULES.md           # ✅ criado
│   ├── SERVICES/RULES.md            # ✅ criado
│   ├── DICTIONARY/RULES.md            # ✅ criado — mensagens de usuário centralizadas (.resx, preparado pra culture)
│   └── SHARED/RULES.md              # ✅ criado
├── 04-TEST/
│   ├── UNIT/RULES.md                # ✅ criado (também define a estrutura geral de Test/)
│   ├── CONTRACT/RULES.md            # ✅ criado
│   └── INTEGRATION/RULES.md         # ✅ criado
├── AGENTS/                          # ✅ criado — papéis especializados (persona + escopo + regras)
│   ├── ORCHESTRATOR.md
│   ├── host-agent.md
│   ├── database-agent.md
│   ├── messaging-agent.md
│   ├── cache-agent.md
│   ├── module-agent.md
│   ├── test-agent.md
│   └── MIGRATION-AGENT.md           # ✅ criado — migração parcial (Cenário A) e cross-stack (Cenário B, ex: Java)
└── CHECKLISTS/                      # ✅ criado
    └── new-module-checklist.md
```

Atualize a coluna de status (✅/⏳) neste README sempre que um arquivo novo for
escrito — este mapa é a visão rápida de progresso do rulebook.

## Decisões de arquitetura já fechadas

Estas decisões foram tomadas em conversa e são vinculantes até serem revistas
explicitamente (ver `ARCHITECTURE-RULES.md` para o detalhamento de cada uma):

| Decisão | Escolha |
|---|---|
| Banco de dados entre módulos | Instância única, **schema por módulo** |
| Roteamento Command/Query → Handler | **Resolução manual via DI** (sem biblioteca mediator) |
| Granularidade de assembly por módulo | **Um único assembly por módulo** (Contracts não é projeto separado) |
| Banco de dados relacional | **SQL Server** |
| Ferramenta de migration | **DbUp** (scripts `.sql` versionados por módulo) |
| Nomenclatura no banco | **snake_case** (propriedades C# continuam PascalCase, mapeadas por type map customizado) |
| Transação/Unit of Work no Dapper | **`IUnitOfWork`** (abre/fecha conexão internamente; `Handler` nunca toca `IDbConnection`/`IDbTransaction` — atualizado em 2026-07-16, versão anterior expunha `IDbTransaction` cru ao Handler) |
| Framework de teste | **xUnit** |
| Biblioteca de mock | **NSubstitute** |
| Ambiente de Integration test | **Testcontainers** (SQL Server, RabbitMQ, Redis em Docker por suíte) |
| Registro de módulo no Host | **Descoberta via assembly** (`IModuleInstaller` + `DependencyContext`) — Host nunca lista módulo por nome em `Program.cs` (2026-07-16) |
| Nomenclatura de projeto/namespace | **Sem prefixo composto** — `Host`, `Pessoas`, `Infrastructure` (nunca `Empresa.Host`, `Empresa.Modules.Pessoas`) (2026-07-16) |
| Granularidade de projeto de Infrastructure | **Um único projeto `Infrastructure`** com `Database/`, `Messaging/`, `Cache/` como subpastas — não 3 projetos separados (2026-07-16) |
| Propriedades MSBuild comuns | **`Directory.Build.props`** na raiz (TargetFramework, Nullable, ImplicitUsings) — nenhum `.csproj` individual repete isso (2026-07-16) |
| Local de `I<NomeModulo>` e `IntegrationEvents` | **`Modules/Shared/Contracts/`** — não mais dentro do `Contracts/` do próprio módulo. Um módulo publica, outro consome, **sem `ProjectReference` direto entre os dois** (2026-07-16, substitui a decisão original de `Contracts` só dentro do módulo) |
| Visibilidade `internal` nas camadas privadas | **Parcial, por causa da decisão "sem mediator"**: `Repository`/`Service`/`Consumer` (implementação concreta) viram `internal`; `Entity`, `Repository` (interface), `Handler`, `Command` continuam `public` porque aparecem na assinatura pública do `Controller` (`ARCHITECTURE-RULES.md` seção 5 detalha o porquê) (2026-07-16) |
| Interface de `Repository` | **Decisão final:** interface em `Modules/<Nome>/Contracts/Repositories/I<Nome>Repository.cs` (subpasta dentro da `Contracts/` que o módulo já tem, junto de `Dtos/`) + implementação concreta em `Modules/<Nome>/Repository/<Nome>Repository.cs` (pasta nova, singular, irmã de `Contracts/`/`Handler/`/`Entities/`). Não vai para `Shared/Contracts` — nunca é injetado por outro módulo, visibilidade `public` é só consequência do C# (`ARCHITECTURE-RULES.md` seção 5.1, `REPOSITORIES/RULES.md` seção 2). Passou por duas tentativas intermediárias no mesmo dia (`Interface/`+`Implementation/`, depois `Repositories/Contracts/`+raiz) antes de chegar nesta (2026-07-17, forma final em 2026-07-19) |
| Handler — granularidade | **Um `Handler` por Aggregate Root/recurso** (tipicamente um por `Controller`), implementando `Shared.Kernel.IHandler<TCommand,TResult>` uma vez por `Command`/`Query` aceito — não um `Handler` por `Command`/`Query` individual. Segundo `Handler` só quando o `Controller` expõe um substantivo distinto sob outro subcaminho (critério "banana vs. tomate", ex: `GeneroHandler` + `AnimeHandler` sob `CatalogoController`) (`HANDLER/RULES.md` seções 3-4) (2026-07-19) |
| Estilo de código — LINQ vs. `foreach` | **LINQ obrigatório** para transformação/filtro/projeção de coleção sem efeito colateral; `foreach` continua correto quando há `await` sequencial ou mutação por item (`ARCHITECTURE-RULES.md` seção 7.1) (2026-07-17) |
| `public partial class Program { }` no Host | **Removido** — só necessário se algum teste precisar de `WebApplicationFactory<Program>`/`typeof(Program)`; testes de arquitetura usam `Assembly.Load("Host")` em vez disso (`01-HOST/RULES.md` seção 3) (2026-07-17) |
| Estrutura de pasta do Host/Infrastructure | **Uma pasta, um `.csproj`, sem aninhamento duplicado**: `Host/Host.csproj`, `Infrastructure/Infrastructure.csproj` (nunca `Host/Host/Host.csproj`) (`01-HOST/RULES.md` seção 2) (2026-07-17) |
| Organização interna de Database/Cache/Messaging | **Subpasta por tipo de arquivo** (`Interface/`, `Implementation/`/`Factory/`, `Map/`, `Extensions/`) na raiz de cada uma; subpasta de funcionalidade já coesa (`Migrations/`, `RabbitMq/`, `Outbox/`) mantém sua própria organização, sem fragmentar mais (`02-INFRASTRUCTURE/DATABASE/RULES.md` seção 3.1) (2026-07-17) |
| Autenticação/Autorização | **JWT** (`Infrastructure/Security`) + autorização **por Policy** (`[Authorize(Policy=...)]`, nomes centralizados em `Shared/Security/GruposDeAcesso`); dado de usuário/senha/grupo é módulo de negócio (`Auth`), nunca dentro de `Infrastructure` (`02-INFRASTRUCTURE/SECURITY/RULES.md`) — registro de policy (`AddAuthorizationPolicies()`) fica no Host, nunca dentro de `AddSecurity()`, porque `Infrastructure` não pode depender de `Shared` (2026-07-17, validado com módulo `Auth` real + smoke test HTTP em 2026-07-19) |
| Exceptions | **`AppException`** (hierarquia com `StatusCode` próprio) + `GlobalExceptionMiddleware`, separado de `DomainException` (sempre bug, sempre `500`) e de `Result.Failure` (falha de negócio esperada, continua sendo o caminho padrão) (`02-INFRASTRUCTURE/SECURITY/RULES.md` seção 6) (2026-07-17) |
| Mensagens de usuário (erro/notificação) | **`.resx` + classe acessadora escrita à mão** por módulo, na pasta `Modules/<Nome>/Dictionary/` — nomeada "Dictionary", não "Messages", para não confundir com `Infrastructure/Messaging` (RabbitMQ); `ResourceManager` resolve fallback de `culture` nativamente, mas `dotnet build` **não gera** `Designer.cs` automaticamente (isso é custom tool do Visual Studio); a classe acessadora `internal` é escrita à mão, uma vez (`03-MODULES/DICTIONARY/RULES.md` seção 2) (2026-07-17, corrigido em 2026-07-19, renomeado de Messages→Dictionary no mesmo dia) |
| Estrutura de pasta de `Test/` | **Flat, igual Host/Infrastructure**: `Test/<Nome>/Unit/Unit.csproj` (nunca `Test/<Nome>/<Nome>.UnitTests/<Nome>.UnitTests.csproj`) — só o `AssemblyName` dentro do `.csproj` mantém o nome qualificado (`<Nome>.UnitTests`), exigido pelo `InternalsVisibleTo` (`04-TEST/UNIT/RULES.md` seção 2.1) (2026-07-18) |
| Mapa de nós para consumo por IA | **`00-PRINCIPLES/NODE-MAP.md`** — catálogo único (grafo de dependência + tabela "o que cada nó pode/não pode conter" + métricas objetivas de sanidade) para qualquer agente decidir rapidamente onde um arquivo pertence, sem ler os 20 `RULES.md` um por um; e **`AGENTS/MIGRATION-AGENT.md`** — papel especializado para os dois cenários de projeto pré-existente: Cenário A (projeto já parcialmente nesta arquitetura, mas divergente — reorganizar preservando comportamento) e Cenário B (migração cross-stack, ex: Java Spring → esta arquitetura, com tabela de mapeamento de conceito e lista de anti-padrões da origem que não podem ser portados) (2026-07-19) |
| Documentação de API (Swagger) | **`Swashbuckle.AspNetCore`** (não o `Microsoft.AspNetCore.OpenApi` minimalista) — único pacote com suporte nativo ao botão "Authorize" (JWT Bearer) na UI; versão fixada em `9.0.6` (não a `10.x`), porque o Swashbuckle 10.x traz `Microsoft.OpenApi` 2.x, que removeu o namespace `Microsoft.OpenApi.Models` (`OpenApiInfo`/`OpenApiSecurityScheme`/`OpenApiReference` deixam de existir lá). Exposto só em `Development` (`UseSwaggerDocumentation()` dentro do `if (app.Environment.IsDevelopment())`), com `RoutePrefix = "swagger"`; `launchSettings.json` usa `launchBrowser: true` + `launchUrl: "swagger"` para abrir direto na página ao rodar local (`01-HOST/RULES.md` seção 3.1, pipeline atualizado na seção 5) (2026-07-19) |
| Reforço — Controller usando dispatcher genérico (`SendAsync`/MediatR) em vez do Handler | A regra "sem mediator" já existia (`ARCHITECTURE-RULES.md` §7), mas um projeto externo violou porque o texto estava só em prosa, sem exemplo ❌/✅ nem enforcement automatizado. Reforçado com: exemplo `❌ Errado` explícito (`IMediator`/`ISender`/dispatcher caseiro) ao lado do `✅ Correto` já existente, linha nova na tabela de anti-padrões, e um teste `Test/Architecture` (`NotHaveDependencyOnAny("MediatR","IMediator","ISender")`) que falha o build se acontecer de novo — vale mesmo que o `Handler` correto já exista no módulo, mas não seja injetado (`03-MODULES/CONTROLLER/RULES.md` seções 3, 7, 8) (2026-07-19) |

## Como evoluir este rulebook

- Qualquer mudança em `ARCHITECTURE-RULES.md` é uma mudança de alto impacto: revisar se ela quebra alguma regra já escrita nas camadas específicas antes de aceitar.
- `RULES.md` de camada pode evoluir de forma mais isolada, mas nunca pode contradizer `00-PRINCIPLES`.
- Toda decisão relevante nova entra na tabela "Decisões de arquitetura já fechadas" acima, com a data.
