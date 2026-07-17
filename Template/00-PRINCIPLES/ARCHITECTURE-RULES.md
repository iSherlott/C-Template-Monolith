# Regras Globais de Arquitetura — Modular Monolith

Estas são as regras que **nenhuma camada, módulo ou agente pode violar**. Todo
`RULES.md` específico (Host, Infrastructure, Modules, Test) é uma
especialização destas regras, nunca uma exceção a elas.

---

## 1. As quatro soluções

| Solução | Responsabilidade | Nunca contém |
|---|---|---|
| **Host** | Composição do ecossistema: DI, `appsettings`, pipeline HTTP, startup | Regra de negócio |
| **Infrastructure** | Recursos técnicos compartilhados: Database, Messaging, Cache | Entidades ou regra de domínio de qualquer módulo |
| **Modules** | Domínio de negócio, um módulo por pasta, isolado | Referência direta a outro módulo (exceto via `Contracts`) |
| **Test** | Unit, Contract, Integration — um conjunto por módulo | Lógica de produção |

## 2. Direção de dependência (regra de ouro)

```
Host  ──depende de──▶  Modules  ──depende de──▶  Infrastructure
```

- `Host` conhece todos os módulos (para compor DI e pipeline), mas nenhum módulo conhece o `Host`.
- `Modules` depende de abstrações expostas por `Infrastructure` (ex: `IDbConnectionFactory`, `IEventBus`, `ICacheService`). `Infrastructure` nunca depende de nada dentro de `Modules`.
- **Módulo A nunca referencia código de Módulo B diretamente.** O único caminho é através da pasta `Contracts` do módulo B (ver seção 4).

Violação desta regra é o critério nº 1 de reprovação em qualquer revisão de
código nesta arquitetura.

## 3. Isolamento de módulo

Dentro de cada módulo, apenas a pasta `Contracts` é superfície pública. Todo o
resto (`Entities`, `Repositories`, `Services`, `Handlers`, `Commands`,
`Consumers`, `Controller`) é privado ao módulo.

- `Entities` nunca são expostas fora do módulo — o que atravessa a fronteira são `Dtos`.
- Um módulo não pode injetar `Repository`, `Service` ou `Handler` de outro módulo. Só pode injetar a interface pública `I<NomeModulo>` — declarada em `Modules/Shared/Contracts/`, não dentro do módulo publicador (ver seção 4).

**❌ Errado — módulo Estoque acessando código privado do módulo Vendas:**

```csharp
// dentro de Modules/Estoque/Handler/ReservarEstoqueHandler.cs
using Vendas.Repositories; // ❌ nem compila — Estoque.csproj não tem ProjectReference pra Vendas (seção 5)
using Vendas.Entities;     // ❌ mesma coisa

public class ReservarEstoqueHandler
{
    private readonly IPedidoRepository _pedidoRepository; // ❌ Repository de outro módulo injetado direto
}
```

**✅ Correto — só a interface pública do módulo publicador, vinda de `Shared`:**

```csharp
// dentro de Modules/Estoque/Handler/ReservarEstoqueHandler.cs
using Shared.Contracts; // ✅ único using que cruza módulo

public class ReservarEstoqueHandler
{
    private readonly IVendasModule _vendasModule; // ✅ interface pública, Modules/Shared/Contracts/
}
```

## 4. Os dois canais de comunicação entre módulos

| Canal | Localização | Natureza | Quando usar |
|---|---|---|---|
| Interface pública `I<NomeModulo>` | `Modules/Shared/Contracts/` | Síncrona, mesmo processo, resposta imediata | O módulo chamador precisa do resultado agora para continuar seu próprio fluxo |
| `IntegrationEvent` | `Modules/Shared/Contracts/IntegrationEvents/` + RabbitMQ (via Outbox) | Assíncrona, desacoplada, eventual consistency | O módulo publicador só precisa notificar um fato; não importa quando/se todos os interessados reagem |

**Os dois vivem em `Shared`, não no módulo publicador.** Isso é deliberado: um
módulo implementa `I<NomeModulo>` e publica `IntegrationEvent`s definidos em
`Shared/Contracts`, e o módulo consumidor só referencia `Shared` — nunca o
`.csproj` do módulo publicador diretamente. Nenhum módulo de negócio tem
`ProjectReference` para outro módulo de negócio; a única aresta permitida é
`Módulo → Shared`. Isso reforça o isolamento além do que code review sozinho
garantiria: fisicamente não é possível `Watchlist` referenciar
`Pessoas.Entities` por engano, porque `Watchlist.csproj` nunca declara essa
referência — o `using` nem compila. Detalhe completo em `CONTRACTS/RULES.md`.

Não existe um terceiro canal. Se uma necessidade de comunicação não se encaixa
em nenhum dos dois, o desenho do módulo está errado — não se cria um atalho.

## 5. Granularidade de assembly — decisão e implicação de enforcement

**Decisão:** um único assembly (`.csproj`) por módulo, nomeado sem prefixo
composto (`Pessoas`, não `Empresa.Modules.Pessoas` — ver `03-MODULES/RULES.md`
seção 3). `I<NomeModulo>` e `IntegrationEvents` não são uma pasta dentro
desse assembly — vivem em `Modules/Shared/Contracts/` (seção 4). O restante
(`Entities`, `Repositories`, `Handler`, `Commands`, `Services`, `Consumers`)
continua dentro do assembly do próprio módulo.

**Isso já elimina metade do problema de enforcement sozinho**: como nenhum
módulo de negócio tem `ProjectReference` para outro módulo de negócio (só
para `Shared` e `Infrastructure`), é **fisicamente impossível compilar** um
`using Pessoas.Entities` de dentro de `Watchlist` — o assembly nem está no
grafo de referências. Isso é mais forte que "code review pega" — o próprio
build falha.

**O que ainda não é garantido pelo compilador**: dentro do *mesmo* módulo,
nada impede uma classe de `Controller/` de acessar `Repositories/` sem passar
por `Handler/`, ou um `Handler` de vazar uma `Entity` em vez de mapear pra
`Dto` — isso continua dependendo de `internal` (seção 5.1) + code review +
teste de arquitetura.

### 5.1 Visibilidade `internal` — até onde dá pra ir sem mediator

Tentamos marcar `Entity`, `Repository` (interface e implementação), `Handler`
e `Command` como `internal` para reforçar ainda mais a fronteira. Esbarramos
numa regra do próprio C#: **um membro `public` não pode ter parâmetro ou
retorno de um tipo menos acessível (`CS0050`/`CS0051`)**. Como esta
arquitetura decidiu **não** usar um dispatcher genérico (`ARCHITECTURE-RULES.md`
seção 7 — `Controller` injeta o `Handler` concreto direto no construtor), a
cadeia de tipos que aparece na assinatura pública do `Controller` é obrigada
a ser pública também:

```
Controller (public, exigido pelo ASP.NET Core para descoberta de rota)
  → construtor injeta Handler          ⇒ Handler precisa ser public
  → action recebe Command no corpo     ⇒ Command precisa ser public
Handler (public)
  → construtor injeta IRepository      ⇒ IRepository (interface) precisa ser public
IRepository (public)
  → método expõe Entity no parâmetro/retorno ⇒ Entity precisa ser public
```

**O que sobra para `internal`, então, é só o que nunca aparece nessa cadeia**:

| Camada | Visibilidade | Por quê |
|---|---|---|
| `Entity` | `public` | Aparece na assinatura de `IRepository`, que é `public` |
| `Repository` (interface) | `public` | Aparece no construtor de `Handler`, que é `public` |
| `Repository` (implementação concreta) | **`internal`** | Só é nomeada dentro do próprio `Install()` do módulo (`services.AddScoped<IRepo, RepoImpl>()`) — nunca em assinatura pública |
| `Handler` | `public` | Injetado direto no construtor do `Controller` |
| `Command`/`Query` | `public` | Recebido como parâmetro de ação do `Controller` |
| `Service`/`<Nome>ModuleFacade` (implementação concreta) | **`internal`** | Só nomeada dentro do próprio `Install()` (`services.AddScoped<IContrato, Facade>()`) |
| `Consumer` | **`internal`** | Só nomeado dentro do próprio `Install()` (`services.AddHostedService<Consumer>()`) |
| `Controller` | `public` | Exigido pelo ASP.NET Core |
| `<NomeModulo>ModuleInstaller` | `public` | Exigido por `Activator.CreateInstance` a partir do assembly do Host (`01-HOST/RULES.md` seção 3) |

Isso funciona porque `Microsoft.Extensions.DependencyInjection` e o hosting
do ASP.NET Core constroem instâncias via reflection respeitando construtores
`public` de tipos `internal` normalmente — o container resolve `internal
PessoaRepository` a partir de `Install()` (mesmo assembly) sem problema, e
quem injeta `IPessoaRepository` em outro lugar nunca precisa nomear o tipo
concreto. Validado em runtime nesta arquitetura (Host sobe, resolve
`Facade`/`Consumer` internos de outro assembly sem erro).

**Se no futuro esta arquitetura adotar um dispatcher genérico** (reabrindo a
decisão da seção 7), `Handler` e `Command` deixam de aparecer na assinatura
pública do `Controller` (que passaria a só conhecer `ICommand<TResult>`) e
**poderiam** virar `internal` também — mas isso não foi adotado aqui.

### 5.2 Enforcement

1. **Code review**: toda PR que adiciona `ProjectReference` de um módulo de negócio para outro módulo de negócio é bloqueada — a única referência permitida é para `Shared` e `Infrastructure`.
2. **Testes de arquitetura obrigatórios** (`04-TEST/CONTRACT/RULES.md`): um teste automatizado (via `NetArchTest` ou equivalente) que falha o build se qualquer módulo referenciar tipo de outro módulo fora de `Shared.Contracts.*`.
3. Sem esse teste de arquitetura no CI, esta decisão de granularidade é considerada **não implementada corretamente** — o teste não é opcional, é parte da própria definição da regra.

## 6. Banco de dados — schema por módulo

**Decisão:** instância única de banco, **um schema por módulo** (ex:
`vendas.pedidos`, `estoque.produtos`).

- Um módulo só pode executar queries (via Dapper) contra o seu próprio schema.
- Nenhum módulo faz `JOIN` ou query direta contra tabelas de outro módulo — mesmo estando no mesmo servidor de banco, isso violaria o isolamento tanto quanto acessar a `Entity` de outro módulo em código.
- Se um módulo precisa de dado que "pertence" a outro, ele pede pela interface pública (`I<NomeModulo>` em `Shared/Contracts`) ou mantém uma cópia local sincronizada via `IntegrationEvent` — nunca por acesso direto ao schema alheio.
- Migrations são versionadas por módulo, dentro do próprio schema.

## 7. Roteamento de Command/Query — resolução manual via DI

**Decisão:** sem biblioteca mediator (ex: MediatR). `Controller` injeta o
`Handler` específico diretamente via construtor e chama o método explicitamente.

- **Um `Handler` por recurso** (tipicamente um por `Controller`, com exceção
  de subcaminhos de Aggregate Root distinta — `HANDLER/RULES.md` seção 4),
  não um `Handler` por `Command`/`Query` individual. O `Handler` implementa
  `IHandler<TCommand, TResult>` (`SHARED/RULES.md` seção 3) uma vez por
  `Command`/`Query` que aceita, com método de entrada explícito por
  sobrecarga (ex: `Handle(CriarPedidoCommand command)`, `Handle(ObterPedidoPorIdQuery query)`
  na mesma classe) — `HANDLER/RULES.md` seção 3 tem o detalhamento completo.
- Nenhum dispatcher genérico por convenção — nem `IMediator.Send(...)` (MediatR), nem um bus/dispatcher caseiro com um método `Send`/`SendAsync`/`Dispatch` que resolve o `Handler` certo por reflection/nome de tipo em runtime. A proibição é do **padrão**, não só da biblioteca MediatR especificamente — um `IRequestDispatcher`/`ICommandBus` escrito à mão que faça a mesma coisa é igualmente proibido. A dependência entre Controller e Handler é explícita e rastreável por "ir para definição", sem indireção de convenção/reflection — o overload correto é resolvido pelo compilador C#, não por reflection em runtime. Isso vale mesmo que o `Handler` correto já exista e esteja correto no módulo: se o `Controller` não o injeta e chama `Handle()` diretamente, a violação existe independente da qualidade do `Handler` (exemplo completo e teste de arquitetura de enforcement em `03-MODULES/CONTROLLER/RULES.md` seções 3 e 8).
- Cross-cutting concerns (validação, logging, transação, exceção não tratada) que normalmente viveriam num pipeline de mediator são resolvidos por decoration explícita do `Handler` ou middleware do `Host`: exceção não tratada → `GlobalExceptionMiddleware` (`02-INFRASTRUCTURE/SECURITY/RULES.md` seção 6); transação → `IUnitOfWork` explícito no `Handler` (`HANDLER/RULES.md` seção 5); validação de forma → `[ApiController]`/data annotation na borda (`CONTROLLER/RULES.md` seção 6).
- **Este foi exatamente o incidente que originou `LIBRARIES.md`**: um projeto adicionou `MediatR` sem perguntar, reabrindo esta decisão silenciosamente. Nenhum pacote NuGet novo — mediator ou qualquer outro — entra num projeto sem confirmação explícita do dev responsável (`LIBRARIES.md` seção 3).

## 7.1 Estilo de código — LINQ obrigatório sobre loop imperativo, quando aplicável

**Decisão:** toda a manutenção deste código é humana (nenhuma geração/edição
automatizada em massa) — legibilidade para quem lê depois é a prioridade
número um em qualquer empate de estilo. Nessa base, transformação/filtro/
projeção de coleção usa LINQ (`Where`, `Select`, `Any`, `FirstOrDefault`,
`OrderBy`, ...) em vez de `foreach` com lista acumuladora manual, sempre que
o resultado for igualmente ou mais legível.

```csharp
// Preferido — a intenção ("iguais compartilhados") está na própria expressão
var animesEmComum = pessoaA.Assistidos.Select(a => a.AnimeId)
    .Intersect(pessoaB.Assistidos.Select(a => a.AnimeId));

// Evitar — mesma coisa, mas exige ler o corpo inteiro do loop pra entender o objetivo
var animesEmComum = new List<Guid>();
foreach (var assistido in pessoaA.Assistidos)
{
    if (pessoaB.Assistidos.Any(a => a.AnimeId == assistido.AnimeId))
        animesEmComum.Add(assistido.AnimeId);
}
```

- **Não é absoluto.** Quando o `foreach` tem efeito colateral por iteração (ex: `await` sequencial por item, mutação de estado externo, múltiplas linhas de lógica por item), `foreach` continua sendo a escolha certa — LINQ existe para transformação de dados sem efeito colateral, forçar isso dentro de `.Select()`/`.ForEach()` com efeito colateral piora a legibilidade, não melhora (regra prática: se o lambda passa de uma linha ou tem `await`/mutação, é sinal de que deveria ser `foreach`).
- Aplica-se em qualquer camada (`Repository`, `Handler`, `Entity`, `Service`), não só `Repository` — a camada mais visada é `Repository` porque é onde mais aparecem transformações de coleção (linha de resultado de query → `Entity`, agrupar itens de agregado), mas a regra é geral de estilo de código desta arquitetura, não específica de uma camada.
- Query LINQ (sintaxe `from x in y select ...`) não é preferida sobre sintaxe de método (`y.Select(x => ...)`) — a arquitetura usa sintaxe de método por consistência, já que ela compõe melhor com `Where`/`Select` encadeados e é o que já aparece no resto do código-base.

## 8. Regra do Shared/Kernel

`Shared` (dentro de `Modules`) é código cruzado entre módulos, composto por
`Helpers`, `Kernel`, `Web` e `Contracts` (seção 4).

- **Kernel** contém apenas primitivos de domínio genéricos e sem significado de negócio próprio: `Result<T>`, `Error`, classes base (`Entity`, `AggregateRoot`), `Guard` clauses, exceptions base.
- **Contracts** contém as interfaces `I<NomeModulo>` e os `IntegrationEvents` publicados — cada tipo aqui *pertence* a um módulo específico (é a superfície pública dele), mesmo vivendo fisicamente em `Shared`. Diferente de `Kernel`, onde nada pode ser específico de um módulo.
- Se um tipo em `Kernel` só faz sentido para um módulo específico, ele não pertence lá — é erro de design, deve voltar para dentro do módulo (isso não se aplica a `Contracts`, que é o lugar certo para tipo específico de módulo que precisa ser público).
- `Shared` nunca depende de nenhum módulo específico (dependência é sempre módulo → Shared, nunca o contrário). `Shared` pode depender de `Infrastructure` (ex: `Contracts.IntegrationEvents` herda `IntegrationEvent`, definido em `Infrastructure/Messaging` — `MESSAGING/RULES.md` seção 4).

**❌ Errado — tipo específico de um módulo colocado em `Kernel` "porque parecia genérico":**

```csharp
// Modules/Shared/Kernel/StatusPedido.cs — ❌ StatusPedido é conceito do módulo Vendas, não genérico
public enum StatusPedido { Aberto, Confirmado, Cancelado }
```

**✅ Correto — fica dentro do próprio módulo; só o que é genuinamente genérico vai para `Kernel`:**

```csharp
// Modules/Vendas/Entities/StatusPedido.cs — ✅ privado ao módulo, nunca aparece em Contracts
public enum StatusPedido { Aberto, Confirmado, Cancelado }

// Modules/Shared/Kernel/Result.cs — ✅ isso sim é genérico, sem significado de negócio próprio
public readonly record struct Result<T>(bool IsSuccess, T? Value, Error? Error);
```

## 9. Enforcement — como estas regras são validadas

- **Testes de arquitetura** (obrigatórios, ver seção 5) rodando no CI a cada build.
- **Contract tests** por módulo garantindo que `I<NomeModulo>` e `IntegrationEvents` publicados não quebram consumidores.
- **Checklist de code review** (a consolidar em `CHECKLISTS/`) cobrindo: direção de dependência, uso correto dos dois canais de comunicação, ausência de query cross-schema.
- Nenhuma regra desta arquitetura é "boa prática sugerida" — todas são critério de bloqueio de PR até prova em contrário documentada aqui.

## 10. Glossário rápido

| Termo | Significado |
|---|---|
| Módulo | Unidade de domínio isolada dentro de `Modules/`, com seu próprio schema de banco, um único `.csproj` sem `ProjectReference` para outro módulo de negócio |
| Contracts | `Modules/Shared/Contracts/` — onde vivem `I<NomeModulo>` e `IntegrationEvents` de todos os módulos; é a única forma de um módulo ser visível para outro |
| IntegrationEvent | Evento publicado por um módulo via Outbox/RabbitMQ para consumo assíncrono por outros módulos |
| Outbox | Padrão que garante que escrita de estado e publicação de evento aconteçam atomicamente |
| Handler | Executor de um `Command` ou `Query` específico, resolvido via DI direta |
