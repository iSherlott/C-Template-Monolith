# RULES — Modules (geral)

> Herda todas as regras de [`00-PRINCIPLES/ARCHITECTURE-RULES.md`](../00-PRINCIPLES/ARCHITECTURE-RULES.md). Este documento especializa, nunca substitui, e é o vocabulário comum que todo `RULES.md` de subpasta (`CONTROLLER/`, `HANDLER/`, `CONTRACTS/`, etc.) assume como já lido.

## 1. Missão

Um módulo é a unidade de domínio isolada desta arquitetura — um bounded
context com dono claro, seu próprio schema de banco, seu próprio prefixo de
cache, e uma única superfície pública (`Contracts`). Tudo que um módulo
precisa para existir e funcionar mora dentro da sua própria pasta em
`Modules/<NomeModulo>/`.

## 2. Estrutura de pastas de um módulo

```
Modules/
├── Shared/                          # ver SHARED/RULES.md — não é um módulo
│   └── Contracts/                   # I<NomeModulo> + IntegrationEvents de TODOS os módulos — ver CONTRACTS/RULES.md
└── <NomeModulo>/
    ├── <NomeModulo>.csproj          # referencia só Infrastructure + Shared — nunca outro módulo
    ├── AssemblyInfo.cs              # [assembly: InternalsVisibleTo(...)] — testes + DynamicProxyGenAssembly2
    ├── GlobalUsings.cs              # usings comuns do módulo (ex: Infrastructure.Database, Shared.Kernel)
    ├── <NomeModulo>ModuleInstaller.cs # implementa IModuleInstaller — ver seção 6
    ├── Controller/                  # ver CONTROLLER/RULES.md
    ├── Handler/                     # ver HANDLER/RULES.md — um por Aggregate Root/recurso
    ├── Contracts/
    │   ├── Dtos/                    # forma de resposta HTTP — só o Controller do próprio módulo usa (CONTRACTS/RULES.md §1)
    │   └── Repositories/            # interfaces de Repository — local ao módulo, não cruza fronteira (CONTRACTS/RULES.md §1.1)
    ├── Entities/                    # ver ENTITIES/RULES.md — privado
    ├── Commands/                    # ver COMMANDS/RULES.md — privado
    ├── Repository/                  # implementação concreta de Repository — ver REPOSITORIES/RULES.md — privado
    ├── Consumers/                   # ver CONSUMERS/RULES.md — privado
    ├── Services/                    # ver SERVICES/RULES.md — privado
    └── Dictionary/                  # ver DICTIONARY/RULES.md — mensagens de usuário (.resx), privado
```

`Modules/Shared/` (Contracts, Kernel, Web) vive no mesmo nível das pastas de
módulo, mas **não é um módulo** — não tem schema de banco próprio, não tem
`ModuleInstaller`. Ver `SHARED/RULES.md`. Diferente da versão anterior desta
regra, `I<NomeModulo>` e `IntegrationEvents` **não** ficam mais dentro de
`Modules/<NomeModulo>/Contracts/` — migraram para `Modules/Shared/Contracts/`
(`CONTRACTS/RULES.md` seção 1), exatamente para que nenhum módulo de negócio
precise de `ProjectReference` para outro.

## 3. Nomenclatura — a mesma palavra em todo lugar

Um módulo tem um único nome, e esse nome se repete, sem tradução, em cada
camada técnica que ele toca:

| Conceito | Convenção | Exemplo (módulo "Vendas") |
|---|---|---|
| Nome do módulo | PascalCase | `Vendas` |
| Pasta/projeto do módulo | `Modules/<NomeModulo>/<NomeModulo>.csproj` | `Modules/Vendas/Vendas.csproj` |
| Namespace C# | `<NomeModulo>` — **sem prefixo composto** (`README.md`, tabela de decisões) | `Vendas` |
| Schema de banco | snake_case/lowercase do nome | `vendas` |
| Prefixo de chave de cache | mesmo nome do schema | `vendas:` |
| Exchange RabbitMQ | `<schema>.events` | `vendas.events` |
| Classe de composição DI | `<NomeModulo>ModuleInstaller` (implementa `IModuleInstaller`) | `VendasModuleInstaller` |
| Seção de configuração | `Modules:<NomeModulo>` | `Modules:Vendas` |

Se o nome do módulo muda, todos esses pontos mudam junto — não existe módulo
com nomes diferentes em camadas diferentes. A ausência de prefixo composto
(nunca `Empresa.Modules.Vendas`) é deliberada: simplifica o namespace e, mais
importante, elimina a possibilidade de descoberta de módulo por prefixo de
nome — o Host descobre por `lib.Type == "project"` (`01-HOST/RULES.md`
seção 3), não por convenção textual.

## 4. Fronteira pública vs. privada (recap aplicado)

De dentro de um módulo, só duas coisas são visíveis para fora
(`ARCHITECTURE-RULES.md` seção 3): a interface `I<NomeModulo>` e os
`IntegrationEvents` que ele publica — e ambos **fisicamente moram em
`Modules/Shared/Contracts/`**, não dentro da pasta do módulo (seção 2 acima).
`Dtos` (`Modules/<NomeModulo>/Contracts/Dtos/`) continuam dentro do módulo
porque só o próprio `Controller` os usa — nenhum outro módulo os importa em
código. Isso vale igualmente para `Host` e para qualquer outro módulo — não
existe "vizinho de confiança" que pode acessar `Entities`/`Repositories`
direto. A única exceção física a essa regra é o próprio arquivo
`<NomeModulo>ModuleInstaller.cs`, que naturalmente referencia tudo internamente
para registrar no DI (ver seção 6) — mas ele não expõe nada além do que já é
público (`I<NomeModulo>` em `Shared/Contracts`).

## 5. Fluxo de uma requisição dentro de um módulo

**Fluxo síncrono (HTTP):**

```
Request HTTP
  → Controller monta Command (COMMANDS/RULES.md seção 4)
  → Controller injeta e chama o Handler específico (DI manual — ARCHITECTURE-RULES seção 7)
  → Handler orquestra: cria um IUnitOfWork se necessário (DATABASE/RULES.md §5),
    chama Repository/Service, opcionalmente publica IntegrationEvent via IEventBus (MESSAGING/RULES.md §5)
  → Handler retorna um Dto (nunca uma Entity) para o Controller
  → Controller mapeia o Dto para a resposta HTTP
```

**Fluxo assíncrono (evento de outro módulo):**

```
Módulo publicador grava estado + evento no Outbox (mesma transação)
  → OutboxPublisherService entrega no RabbitMQ
  → Consumer deste módulo recebe, deserializa
  → Consumer verifica idempotência (tabela processed_messages — MESSAGING/RULES.md §9)
  → Consumer chama Service/Handler apropriado
  → Se necessário, este módulo grava seu próprio estado e pode publicar seu próprio IntegrationEvent
```

Em nenhum dos dois fluxos um Controller ou Consumer acessa `Repository`
diretamente — sempre passam por `Handler` (síncrono) ou `Service` (chamado
pelo Consumer).

## 6. `<NomeModulo>ModuleInstaller.cs` — checklist do que `Install()` deve registrar

```csharp
public class VendasModuleInstaller : IModuleInstaller
{
    public IServiceCollection Install(IServiceCollection services, IConfiguration configuration)
    {
        // checklist abaixo
        return services;
    }

    // opcional — só sobrescrever se o módulo precisa registrar algo no pipeline HTTP
    public void Use(IApplicationBuilder app) { }
}
```

`IModuleInstaller` (interface definida em `Modules/Shared` — `SHARED/RULES.md`)
é descoberto e instalado automaticamente pelo Host via assembly scanning
(`01-HOST/RULES.md` seção 3) — nenhum código do Host precisa nomear esse
módulo. `Install()` é responsável por registrar **tudo** que o módulo precisa
para funcionar:

- [ ] `Handlers` (lifetime `Scoped`) — `public`
- [ ] `Repositories` (lifetime `Scoped`) — interface `public`, implementação concreta `internal` (`REPOSITORIES/RULES.md` seção 2)
- [ ] `Services` (lifetime `Scoped` por padrão; `Singleton` só quando explicitamente sem estado e justificado em comentário) — implementação concreta `internal`, incluindo `<NomeModulo>ModuleFacade` (`SERVICES/RULES.md` seção 5)
- [ ] `Consumers` como `IHostedService`/`BackgroundService` (lifetime `Singleton`, natureza do hosted service) — `internal` (`CONSUMERS/RULES.md` seção 3)
- [ ] `Controllers` do módulo como application part (`AddControllers().AddApplicationPart(...)`) — `public`, exigido pelo ASP.NET Core
- [ ] Registro do schema do módulo no `IOutboxSchemaRegistry` (`MESSAGING/RULES.md` seção 6), se o módulo publica eventos
- [ ] Health check específico do módulo, se houver
- [ ] Leitura da própria seção de configuração (`Modules:<NomeModulo>`), nunca de outra seção

**❌ Errado — sem construtor parameterless, `internal`, sem retornar `services`:**

```csharp
internal class VendasModuleInstaller : IModuleInstaller // ❌ precisa ser public
{
    private readonly ILogger _logger;
    public VendasModuleInstaller(ILogger logger) => _logger = logger; // ❌ Activator.CreateInstance exige parameterless

    public IServiceCollection Install(IServiceCollection services, IConfiguration configuration)
    {
        services.AddScoped<PedidoHandler>();
        // ❌ esqueceu de retornar services, esqueceu de registrar Repository/Controller/Consumer
    }
}
```

**✅ Correto:**

```csharp
public class VendasModuleInstaller : IModuleInstaller // ✅ public, construtor parameterless
{
    public IServiceCollection Install(IServiceCollection services, IConfiguration configuration)
    {
        services.AddScoped<PedidoHandler>();
        services.AddScoped<IPedidoRepository, PedidoRepository>();
        services.AddScoped<IVendasModule, VendasModuleFacade>();
        services.AddHostedService<PedidoCriadoConsumer>();
        services.AddControllers().AddApplicationPart(typeof(VendasModuleInstaller).Assembly);
        return services; // ✅ sempre retorna services
    }

    public void Use(IApplicationBuilder app) { }
}
```

A classe `<NomeModulo>ModuleInstaller` precisa ser `public` com **construtor
parameterless** — o Host a instancia via `Activator.CreateInstance`
(`01-HOST/RULES.md` seção 3); sem isso, o módulo é descoberto mas falha ao
instalar, silenciosamente, na primeira subida. Isso é consistente com
`ARCHITECTURE-RULES.md` seção 5.1: mesmo `Repository`/`Service`/`Consumer`
concretos sendo `internal`, a construção via DI/hosting funciona normalmente
através de assembly boundaries — só o `ModuleInstaller` em si (que é
resolvido diretamente por `Activator.CreateInstance`, fora do container DI)
realmente precisa ser `public`.

Um módulo cujo `Install()` não cobre algum item aplicável desta lista está
incompleto — é item de checklist de code review para módulo novo (ver
`CHECKLISTS/new-module-checklist.md`).

## 7. Commands — convenção unificada (sufixo único `Command`)

A pasta `Commands/` contém tanto mutação quanto leitura, todos sob o mesmo
sufixo `Command` — não existe sufixo `Query` nem pasta `Queries/` separada
nesta arquitetura (`COMMANDS/RULES.md` seção 4 tem o detalhamento completo,
incluindo os verbos em inglês `Create`/`Update`/`Get`/`Delete`). A distinção
entre uma ação e outra é só o verbo no nome da classe (`CreatePedidoCommand`
vs. `GetPedidoByIdCommand`) e o `Handler` correspondente, não a localização
física nem um sufixo diferente. Isso mantém a estrutura de pastas do módulo
enxuta; se um módulo específico crescer a ponto de justificar a separação
física, isso é uma decisão local documentada no próprio módulo, não uma
mudança na regra geral.

## 8. Quando criar um módulo novo

Heurística: um módulo existe em torno de um **substantivo de negócio com
ciclo de vida e dono próprios** (ex: Vendas, Estoque, Pagamentos) — nunca em
torno de uma camada técnica (não existe módulo "Relatórios" genérico) nem de
uma única entidade isolada sem contexto (não existe módulo para cada tabela).

Sinal de que duas coisas pertencem ao **mesmo** módulo: mudam juntas na
maioria das vezes e uma não faz sentido de negócio sem a outra (ex:
`Pedido` e `ItemDoPedido`).

Sinal de que precisa **virar dois módulos**: times diferentes cuidam de cada
parte, ou as duas partes têm taxa de mudança e motivo de mudança
completamente diferentes (ex: `Pedido` muda por regra comercial, `NotaFiscal`
muda por regra fiscal — mesmo que hoje pareçam acoplados, são domínios
diferentes).

## 9. Anti-padrões — o que nunca pode aparecer em um módulo

| Anti-padrão | Por quê é proibido |
|---|---|
| `Controller` chamando `Repository` direto, pulando `Handler` | Perde o ponto único de orquestração/transação do fluxo |
| `Handler` retornando `Entity` em vez de `Dto` de `Contracts` | Vaza modelo interno de domínio para fora do módulo |
| `Service` registrado como `Singleton` guardando estado mutável de request | Vazamento de estado entre requisições concorrentes |
| `<NomeModulo>ModuleInstaller.cs` contendo lógica de negócio além de registro de DI | `ModuleInstaller` é composição, não comportamento |
| `ModuleInstaller` sem construtor parameterless ou não `public` | `Activator.CreateInstance` (`01-HOST/RULES.md` seção 3) exige os dois — sem isso o módulo falha ao instalar |
| Dois módulos usando o mesmo nome de schema/prefixo de cache | Colisão de namespace, quebra isolamento |

## 10. Enforcement

- Code review usa o checklist da seção 6 para todo módulo novo ou modificado.
- Teste de arquitetura (`ARCHITECTURE-RULES.md` seção 5) valida a fronteira `Contracts` também dentro de cada módulo individualmente.
- Nome de módulo, schema, prefixo de cache e exchange são conferidos como idênticos (seção 3) antes de merge.
