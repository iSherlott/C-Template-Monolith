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
2. O `RULES.md` da camada específica que você vai tocar (`01-HOST`, `02-INFRASTRUCTURE/...`, `03-MODULES/...`, `04-TEST/...`).
3. O arquivo do papel correspondente em `AGENTS/`, se você estiver atuando como um papel especializado (ex: construindo só a camada de Database, adote `AGENTS/database-agent.md`).
4. O checklist relevante em `CHECKLISTS/`, se a tarefa for criar algo novo do zero (ex: um módulo novo).

Nenhuma regra de camada específica pode contradizer `ARCHITECTURE-RULES.md`. Se
parecer necessário, a mudança é na regra global primeiro, com decisão
explícita — não uma exceção silenciosa numa pasta.

## Mapa da pasta

```
Template/
├── README.md                        # este arquivo — índice mestre
├── 00-PRINCIPLES/
│   └── ARCHITECTURE-RULES.md        # ✅ regras globais (dependência, isolamento, comunicação)
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
│   ├── MESSAGES/RULES.md            # ✅ criado — mensagens de usuário centralizadas (.resx, preparado pra culture)
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
│   └── test-agent.md
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
| Interface de `Repository` | **Continua dentro de `Modules/<Nome>/Repositories/`, não em `Contracts`** — `Contracts` é reservado ao que atravessa a fronteira do módulo (`I<NomeModulo>`, `IntegrationEvents`, `Dtos`); `Repository` nunca é injetado por outro módulo, sua visibilidade `public` é só consequência do C# (`ARCHITECTURE-RULES.md` seção 5.1). Dentro de `Repositories/`, interface e implementação ficam em subpastas próprias (`Interface/`, `Implementation/`), mesmo padrão já usado em `Infrastructure` (`REPOSITORIES/RULES.md` seção 2) (2026-07-17, refinado com subpastas em 2026-07-19) |
| Estilo de código — LINQ vs. `foreach` | **LINQ obrigatório** para transformação/filtro/projeção de coleção sem efeito colateral; `foreach` continua correto quando há `await` sequencial ou mutação por item (`ARCHITECTURE-RULES.md` seção 7.1) (2026-07-17) |
| `public partial class Program { }` no Host | **Removido** — só necessário se algum teste precisar de `WebApplicationFactory<Program>`/`typeof(Program)`; testes de arquitetura usam `Assembly.Load("Host")` em vez disso (`01-HOST/RULES.md` seção 3) (2026-07-17) |
| Estrutura de pasta do Host/Infrastructure | **Uma pasta, um `.csproj`, sem aninhamento duplicado**: `Host/Host.csproj`, `Infrastructure/Infrastructure.csproj` (nunca `Host/Host/Host.csproj`) (`01-HOST/RULES.md` seção 2) (2026-07-17) |
| Organização interna de Database/Cache/Messaging | **Subpasta por tipo de arquivo** (`Interface/`, `Implementation/`/`Factory/`, `Map/`, `Extensions/`) na raiz de cada uma; subpasta de funcionalidade já coesa (`Migrations/`, `RabbitMq/`, `Outbox/`) mantém sua própria organização, sem fragmentar mais (`02-INFRASTRUCTURE/DATABASE/RULES.md` seção 3.1) (2026-07-17) |
| Autenticação/Autorização | **JWT** (`Infrastructure/Security`) + autorização **por Policy** (`[Authorize(Policy=...)]`, nomes centralizados em `Shared/Security/GruposDeAcesso`); dado de usuário/senha/grupo é módulo de negócio (`Auth`), nunca dentro de `Infrastructure` (`02-INFRASTRUCTURE/SECURITY/RULES.md`) — registro de policy (`AddAuthorizationPolicies()`) fica no Host, nunca dentro de `AddSecurity()`, porque `Infrastructure` não pode depender de `Shared` (2026-07-17, validado com módulo `Auth` real + smoke test HTTP em 2026-07-19) |
| Exceptions | **`AppException`** (hierarquia com `StatusCode` próprio) + `GlobalExceptionMiddleware`, separado de `DomainException` (sempre bug, sempre `500`) e de `Result.Failure` (falha de negócio esperada, continua sendo o caminho padrão) (`02-INFRASTRUCTURE/SECURITY/RULES.md` seção 6) (2026-07-17) |
| Mensagens de usuário (erro/notificação) | **`.resx` + classe acessadora escrita à mão** por módulo (`Modules/<Nome>/Messages/`) — `ResourceManager` resolve fallback de `culture` nativamente, mas `dotnet build` **não gera** `Messages.Designer.cs` automaticamente (isso é custom tool do Visual Studio); a classe acessadora `internal` é escrita à mão, uma vez (`03-MODULES/MESSAGES/RULES.md` seção 2) (2026-07-17, corrigido e validado em código real em 2026-07-19) |
| Estrutura de pasta de `Test/` | **Flat, igual Host/Infrastructure**: `Test/<Nome>/Unit/Unit.csproj` (nunca `Test/<Nome>/<Nome>.UnitTests/<Nome>.UnitTests.csproj`) — só o `AssemblyName` dentro do `.csproj` mantém o nome qualificado (`<Nome>.UnitTests`), exigido pelo `InternalsVisibleTo` (`04-TEST/UNIT/RULES.md` seção 2.1) (2026-07-18) |

## Como evoluir este rulebook

- Qualquer mudança em `ARCHITECTURE-RULES.md` é uma mudança de alto impacto: revisar se ela quebra alguma regra já escrita nas camadas específicas antes de aceitar.
- `RULES.md` de camada pode evoluir de forma mais isolada, mas nunca pode contradizer `00-PRINCIPLES`.
- Toda decisão relevante nova entra na tabela "Decisões de arquitetura já fechadas" acima, com a data.
