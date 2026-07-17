# NODE-MAP — Catálogo rápido de nós, grafo de dependência e métricas

> Herda [`ARCHITECTURE-RULES.md`](ARCHITECTURE-RULES.md). Este arquivo não cria
> regra nova — é o **índice de consulta rápida** para qualquer IA que precise
> decidir, em segundos, "que tipo de nó é este arquivo, o que pode ir dentro
> dele, e qual `RULES.md` eu preciso abrir para o detalhe completo". Leia este
> arquivo **inteiro antes de tocar em qualquer projeto real** — ele é mais
> rápido de carregar do que os 20 `RULES.md` individuais e evita decisão por
> suposição.

Se este arquivo e um `RULES.md` de camada específica divergirem, o `RULES.md`
específico vence — este mapa é resumo, não fonte de verdade.

## 1. Grafo de dependência (direção nunca pode inverter)

```
                        ┌───────────────┐
                        │     Host      │  composition root — conhece todo mundo
                        └───────┬───────┘
                 ┌──────────────┼──────────────┐
                 ▼              ▼              ▼
        ┌────────────────┐ ┌──────────┐ ┌─────────────┐
        │ Infrastructure │ │ Modules/ │ │ Modules/    │
        │ (Database/     │ │ Shared   │ │ <Nome> (N×) │
        │  Messaging/    │ │          │ │             │
        │  Cache/        │ └────┬─────┘ └──────┬──────┘
        │  Security)     │      │              │
        └───────▲────────┘      │              │
                │                └──────┬───────┘
                │                       ▼
                │              Modules/<Nome> pode referenciar
                │              Infrastructure + Modules/Shared,
                │              NUNCA outro Modules/<Outro> direto
                │              (ProjectReference proibido — só via
                │               Modules/Shared/Contracts, seção 4 abaixo)
                │
     Infrastructure NUNCA referencia Modules/Shared nem Modules/<Nome>
     (regra de ouro — ARCHITECTURE-RULES.md seção 2)

        ┌─────────────────────────────────────────────┐
        │ Test/<Nome>/{Unit,Contract,Integration}      │  referencia o módulo
        │ Test/Architecture                            │  (NetArchTest) valida
        └─────────────────────────────────────────────┘  todas as setas acima
```

**As duas únicas exceções que atravessam módulo:**

1. `I<NomeModulo>` (interface pública do módulo) — vive em `Modules/Shared/Contracts/`.
2. `IntegrationEvents` (evento publicado/consumido entre módulos) — também em `Modules/Shared/Contracts/`.

Nenhuma outra pasta de um módulo (`Entities`, `Repository`, `Handler`,
`Services`, `Commands`, `Consumers`, `Dictionary`, `Contracts/Dtos`,
`Contracts/Repositories`) é visível fora do próprio módulo — nem por leitura,
nem por `ProjectReference`.

## 2. Catálogo de nós — o que cada pasta/arquivo é, pode e não pode ter

> Convenção da tabela: `<N>` = nome do módulo (PascalCase, ex: `Vendas`).
> "Visibilidade" é a decisão já fechada em `README.md` ("Visibilidade
> `internal` nas camadas privadas").

| Nó (caminho) | Camada | O que é | Pode conter | Não pode conter | Visibilidade | Fonte de verdade |
|---|---|---|---|---|---|---|
| `Host/Program.cs` | Host | Composition root único | Chamadas a `Add*`/`Use*` já existentes, `if (IsDevelopment())` | Regra de negócio, `if` de domínio, `new AlgumRepository()` direto, módulo listado por nome | — | `01-HOST/RULES.md` §3, §5, §7 |
| `Host/DependencyInjection/*Extensions.cs` | Host | Agrupador de `IServiceCollection`/`IApplicationBuilder` por preocupação (Infra, Módulos, Swagger) | `AddInfrastructure()`, `AddModules()` (assembly scan), `AddSwaggerDocumentation()` | Lógica condicional de negócio | — | `01-HOST/RULES.md` §3, §3.1 |
| `Host/appsettings.json` | Host | Configuração por seção (`Modules:<N>`, `Infrastructure:*`) | Valores default, connection string de dev | Segredo de produção versionado | — | `01-HOST/RULES.md` §4 |
| `Host/Properties/launchSettings.json` | Host | Perfil de execução local | `launchUrl`/`launchBrowser` apontando pro Swagger | — | — | `01-HOST/RULES.md` §3.1 |
| `Infrastructure/Database/{Interface,Implementation,Map,Migrations}` | Infra | Acesso a dado genérico (`IUnitOfWork`, DbUp) | Conexão, transação, type map customizado | Regra de negócio, schema de módulo específico | pública (contrato) / interna (impl) | `02-INFRASTRUCTURE/DATABASE/RULES.md` |
| `Infrastructure/Messaging/{Interface,Implementation,RabbitMq,Outbox}` | Infra | Publish/consume genérico (`IEventBus`, Outbox) | Serialização, retry, DLQ, idempotência genérica | `IntegrationEvent` de módulo específico | pública/interna | `02-INFRASTRUCTURE/MESSAGING/RULES.md` |
| `Infrastructure/Cache/{Interface,Implementation}` | Infra | Abstração de cache (`ICacheService`) | Get/Set/Invalidate genérico | Chave hardcoded de módulo específico | pública/interna | `02-INFRASTRUCTURE/CACHE/RULES.md` |
| `Infrastructure/Security/{Interface,Implementation,Exceptions,Extensions}` | Infra | JWT (`ITokenService`), hashing, `AppException`, `GlobalExceptionMiddleware` | Emissão/validação de token, hashing de senha, middleware de exceção | `GruposDeAcesso` (mora em `Modules/Shared/Security` — é vocabulário de negócio), `AddAuthorizationPolicies()` (mora no Host) | pública/interna | `02-INFRASTRUCTURE/SECURITY/RULES.md` |
| `Modules/Shared/Kernel/` | Módulo (não-módulo) | `IHandler<TCommand,TResult>`, `Result<T>` | Contratos genéricos sem dependência de negócio | Qualquer tipo específico de um módulo | pública | `03-MODULES/SHARED/RULES.md` |
| `Modules/Shared/Contracts/` | Módulo (não-módulo) | `I<NomeModulo>` + `IntegrationEvents` de TODOS os módulos | Interface pública do módulo, classe de evento | `Entity`, `Dto` de request/response, lógica | pública | `03-MODULES/CONTRACTS/RULES.md` §1 |
| `Modules/Shared/Security/GruposDeAcesso.cs` | Módulo (não-módulo) | Constantes de nome de policy/grupo | `const string Administrador = "..."` | Lógica de autorização em si (isso é `Infrastructure/Security`) | pública | `02-INFRASTRUCTURE/SECURITY/RULES.md` §7 |
| `Modules/<N>/<N>.csproj` | Módulo | Projeto do módulo | `ProjectReference` a `Infrastructure` + `Modules/Shared` | `ProjectReference` a outro `Modules/<Outro>` | — | `03-MODULES/RULES.md` §3 |
| `Modules/<N>/<N>ModuleInstaller.cs` | Módulo | Composition local — implementa `IModuleInstaller` | Registro DI de tudo do módulo (checklist §6 abaixo) | Regra de negócio | pública, construtor parameterless | `03-MODULES/RULES.md` §6 |
| `Modules/<N>/Controller/` | Módulo | Superfície HTTP — 1 Controller por Aggregate Root/recurso | `[Authorize]`/`[AllowAnonymous]`, mapeamento request→Command, injeção do Handler | Regra de negócio, acesso a `Repository` direto, `IMediator`/`ISender`/dispatcher genérico (`SendAsync`/`Dispatch`) no lugar do `Handler` concreto | pública (exigido ASP.NET Core) | `03-MODULES/CONTROLLER/RULES.md` §3, §7, §8 |
| `Modules/<N>/Handler/` | Módulo | Orquestração — 1 Handler por recurso, implementa `IHandler<TCmd,TResult>` por Command/Query aceito | Chamada a `Repository`/`Service`, `IUnitOfWork`, publicação de evento | Acesso a `IDbConnection` cru, captura de exceção de infra pra virar HTTP | pública | `03-MODULES/HANDLER/RULES.md` |
| `Modules/<N>/Contracts/Dtos/` | Módulo | Forma de request/response HTTP | Records/classes imutáveis, sem lógica | Referência a `Entity` | pública | `03-MODULES/CONTRACTS/RULES.md` §1 |
| `Modules/<N>/Contracts/Repositories/I<N>Repository.cs` | Módulo | Interface do Repository — local, não cruza módulo | Assinatura de métodos de persistência do Aggregate Root | Repository genérico (`IRepository<T>`) | pública (consequência do C#, não contrato cruzado) | `03-MODULES/REPOSITORIES/RULES.md` §2, `03-MODULES/CONTRACTS/RULES.md` §1.1 |
| `Modules/<N>/Repository/<N>Repository.cs` | Módulo | Implementação concreta — SQL explícito via Dapper | Query/comando SQL explícito por Aggregate Root | `Repository<T>` genérico, LINQ-to-SQL dinâmico sem controle | interna | `03-MODULES/REPOSITORIES/RULES.md` |
| `Modules/<N>/Entities/` | Módulo | Modelo de domínio rico | Invariante, comportamento, validação de estado | Atributo de ORM, referência a `Dto`/`Infrastructure` | pública (aparece na assinatura do `Handler`) | `03-MODULES/ENTITIES/RULES.md` |
| `Modules/<N>/Commands/` | Módulo | Command + Query (mesma pasta, sem separação física) | Records imutáveis de entrada do `Handler` | Lógica, acesso a dado | pública | `03-MODULES/COMMANDS/RULES.md`, `03-MODULES/RULES.md` §7 |
| `Modules/<N>/Services/` | Módulo | Lógica de domínio reutilizável entre Handlers, `<N>ModuleFacade` (implementa `I<N>`) | Regra de negócio compartilhada, chamada síncrona vinda de outro módulo via `I<N>` | Estado mutável em `Singleton` | interna (exceto Facade que implementa interface pública) | `03-MODULES/SERVICES/RULES.md` |
| `Modules/<N>/Consumers/` | Módulo | Reação a `IntegrationEvent` de outro módulo | Deserialização, checagem de idempotência, chamada a `Service` | Acesso direto a `Repository` de outro módulo | interna | `03-MODULES/CONSUMERS/RULES.md` |
| `Modules/<N>/Dictionary/<N>Dictionary.cs` + `.resx` | Módulo | Mensagens de usuário centralizadas (i18n-ready) | Chave/valor de texto, classe acessadora via `ResourceManager` | Lógica, string hardcoded espalhada pelo módulo | interna | `03-MODULES/DICTIONARY/RULES.md` |
| `Test/<N>/Unit/` | Test | Testa `Handler`/`Entity`/`Service` isolado (mock de `Repository`) | Mock via NSubstitute, xUnit | Banco/fila real | — | `04-TEST/UNIT/RULES.md` |
| `Test/<N>/Contract/` | Test | Testa fronteira pública (`I<N>`, `Dto`, `IntegrationEvent`) do módulo | Serialização, forma do contrato | Detalhe de implementação interna | — | `04-TEST/CONTRACT/RULES.md` |
| `Test/<N>/Integration/` | Test | Testa `Repository`/`Consumer` contra infra real | Testcontainers (SQL Server/RabbitMQ/Redis) | Mock de infra | — | `04-TEST/INTEGRATION/RULES.md` |
| `Test/Architecture/` | Test | Valida as setas da seção 1 via NetArchTest | `Assembly.Load(nome)`, regra de dependência proibida | Teste de comportamento de negócio | — | `ARCHITECTURE-RULES.md` §9 |

### 2.1 Erros comuns de leitura deste catálogo

Duas colunas da tabela acima são fonte frequente de inferência errada — vale
o par ❌/✅ explícito:

**❌ Errado — "a coluna Visibilidade diz `pública`, logo outro módulo pode injetar/referenciar":**

```
Modules/<N>/Contracts/Repositories/I<N>Repository.cs → Visibilidade: pública
❌ inferência errada: "então outro módulo pode injetar IPedidoRepository"
```

**✅ Correto — `public` aqui é só cascata de acessibilidade do C# (a interface aparece no
construtor `public` do `Handler`), não uma declaração de fronteira cruzável.
A pergunta certa é sempre "existe `ProjectReference` de outro módulo para
este?" (nunca existe — `CONTRACTS/RULES.md` §1.1) — nunca "a visibilidade
C# permite?":**

```
✅ inferência correta: nenhum outro módulo tem ProjectReference para este
   módulo, então IPedidoRepository fisicamente não compila fora dele,
   mesmo sendo `public`
```

**❌ Errado — "o nó `Handler/` existe e está correto, logo o Controller está usando ele":**

```
Modules/Vendas/Handler/PedidoHandler.cs → implementado certinho, testado
❌ inferência errada: "logo a rota HTTP /api/vendas/pedidos está correta"
```

**✅ Correto — um `Handler` correto não garante que o `Controller` o injeta;
são nós separados na tabela acima, cada um com sua própria métrica de
sanidade (seção 3) — confira as duas independentemente:**

```
✅ verificar separadamente: Controller/PedidosController.cs injeta
   PedidoHandler no construtor e chama .Handle(...)? (métrica dedicada,
   seção 3 abaixo — este foi exatamente o caso real que motivou a métrica)
```

## 3. Métricas de sanidade — como uma IA confere se um projeto real bate com o mapa

Estes são checks objetivos, executáveis (por grep, build, ou pelo próprio
`Test/Architecture`), não opinião:

| Métrica | Como verificar | Resultado esperado |
|---|---|---|
| Nenhum módulo referencia outro módulo direto | `grep ProjectReference` em cada `.csproj` de `Modules/<N>/` | Só `Infrastructure` e `Modules/Shared` aparecem |
| `Infrastructure` não referencia módulo nenhum | `grep ProjectReference` em `Infrastructure.csproj` | Vazio (nenhum `Modules/*`) |
| Todo `Controller` declara intenção de acesso | `grep -L "\[Authorize\|\[AllowAnonymous\]"` em `Controller/*.cs` | Lista vazia — todo arquivo tem um dos dois |
| Todo `Handler` retorna `Dto`, nunca `Entity` | Assinatura pública de `IHandler<TCmd,TResult>` — `TResult` é sempre tipo de `Contracts/Dtos` | 100% |
| Nenhum `Controller` usa dispatcher genérico em vez do `Handler` concreto | `grep -rn "IMediator\|ISender\|_mediator\.\|\.SendAsync(\|\.Dispatch("` em `Modules/*/Controller/*.cs` | Zero ocorrências — todo `Controller` injeta `<Recurso>Handler` no construtor e chama `.Handle(...)` direto (`CONTROLLER/RULES.md` §3, §7, §8) |
| `Repository` implementação é sempre `internal` | `grep "internal class.*Repository"` em `Repository/*.cs` | 100% dos arquivos |
| Nenhum `Repository<T>`/`IRepositoryBase<T>` genérico | `grep "Repository<T>\|RepositoryBase"` no repo inteiro | Zero ocorrências (anti-padrão banido — `README.md`) |
| 1 `Handler` por Aggregate Root, não por Command | Contar classes em `Handler/` vs. contar `Command`/`Query` em `Commands/` | Nº de Handlers ≤ nº de Controllers; cada Handler implementa `IHandler<>` 1x por Command/Query que atende |
| `Test/Architecture` passa | `dotnet test Test/Architecture` | 100% verde — se falhar, o código real diverge do mapa, não o contrário |
| Todo módulo tem as 3 camadas de teste | `ls Test/<N>/{Unit,Contract,Integration}` | 3 pastas presentes por módulo com lógica não-trivial |
| Nomenclatura consistente (seção 3 de `03-MODULES/RULES.md`) | Nome do módulo == nome do schema == prefixo de cache == nome do exchange | Idêntico em todos os 4 lugares |
| Swagger nativo (`01-HOST/RULES.md` §3.1) | `grep AddSwaggerDocumentation` em `Program.cs` + `launchUrl` em `launchSettings.json` | Presente, condicionado a `Development` |

Uma métrica falhando não significa "editar o mapa para o código passar" —
significa que o projeto real tem uma divergência que precisa ser corrigida ou
explicitamente escalada como exceção documentada (`README.md`, tabela de
decisões).

## 4. Cenários de uso — qual playbook seguir

| Cenário | Playbook | Quando usar |
|---|---|---|
| Projeto novo, do zero | [`AGENTS/ORCHESTRATOR.md`](../AGENTS/ORCHESTRATOR.md) — "Sequência: projeto novo, do zero" | Nenhum código ainda existe |
| Feature nova em módulo já existente e já conforme | [`AGENTS/ORCHESTRATOR.md`](../AGENTS/ORCHESTRATOR.md) — "Sequência: feature nova em módulo existente" | O módulo já segue o mapa, só falta uma funcionalidade |
| Projeto já tem parte migrada/estruturada nesta arquitetura | [`AGENTS/MIGRATION-AGENT.md`](../AGENTS/MIGRATION-AGENT.md) — "Cenário A" | Existe código C# real, mas pastas/arquivos não batem 100% com a seção 2 deste mapa |
| Migrar projeto de outra stack/estrutura (ex: Java) para esta arquitetura | [`AGENTS/MIGRATION-AGENT.md`](../AGENTS/MIGRATION-AGENT.md) — "Cenário B" | Código-fonte de origem não é C#, ou é C# mas em arquitetura totalmente diferente (ex: N-layer clássico, hexagonal) |

Em todos os quatro cenários, a leitura mínima antes de qualquer edição é:
este `NODE-MAP.md` + `ARCHITECTURE-RULES.md`. O playbook específico diz o que
ler a seguir.
