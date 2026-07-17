# RULES — Contracts (I&lt;NomeModulo&gt; e IntegrationEvents ficam em Shared)

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md` (seção 4 e 5.1) e `03-MODULES/RULES.md`. Este documento especializa, nunca substitui.

## 1. Missão

A superfície pública de um módulo tem **duas partes com localizações
diferentes**, e essa divisão é deliberada:

| O que | Onde vive | Por quê |
|---|---|---|
| `I<NomeModulo>` (fachada síncrona) + `IntegrationEvents` | `Modules/Shared/Contracts/` | É código que **outro módulo compila contra** — precisa estar num lugar que ambos os lados já referenciam, sem exigir `ProjectReference` de um módulo de negócio para outro |
| `Dtos` (forma de resposta HTTP) | `Modules/<NomeModulo>/Contracts/Dtos/` | É consumido só por HTTP (fora do processo) — nenhum outro módulo importa esse tipo em código, então não precisa sair do módulo |

Se um `Dto` um dia precisar aparecer no retorno de um método de `I<NomeModulo>`,
ele migra para `Shared/Contracts` também — a regra é "mora onde é compilado
contra", não "mora onde nasceu".

### 1.1 `IPedidoRepository` mora em `Contracts/Repositories/` — mas não é superfície pública

**Decisão final:** a interface de `Repository` fica dentro da pasta
`Contracts/` que o próprio módulo já tem (a mesma que abriga `Dtos/`), numa
subpasta dedicada: `Modules/<NomeModulo>/Contracts/Repositories/IPedidoRepository.cs`.
A implementação concreta mora numa pasta irmã, no singular:
`Modules/<NomeModulo>/Repository/PedidoRepository.cs` (`REPOSITORIES/RULES.md`
seção 2 tem o detalhamento completo).

Isso **não** significa que `Repository` passou a cruzar a fronteira do
módulo. `Repository` é `public` (`REPOSITORIES/RULES.md` seção 2) só por
consequência do C# (a interface aparece no construtor `public` de
`Handler`, `ARCHITECTURE-RULES.md` seção 5.1) — ninguém fora do módulo
jamais injeta ou referencia `IPedidoRepository`. `Contracts/` dentro de um
módulo passa a ter duas subpastas com propósitos diferentes:

| Subpasta | Cruza a fronteira do módulo? | Motivo de estar em `Contracts/` |
|---|---|---|
| `Contracts/Requests/` | Sim, via HTTP (fora do processo) — mas só entrada, nunca sai do `Controller` para dentro | Seção 1.2 abaixo |
| `Contracts/Dtos/` | Sim, via HTTP (fora do processo) | Seção 1 acima |
| `Contracts/Repositories/` | **Não** — nenhum outro módulo o referencia | Só organização: centraliza toda interface/contrato do módulo (o que é "tipo que algo compila contra", mesmo que esse "algo" seja só o próprio módulo) num único lugar, em vez de espalhar uma pasta `Contracts/` própria dentro de `Repository/` |

Code review não deve inferir "está em `Contracts/`, logo é público para
outros módulos" — a pergunta certa continua sendo "existe `ProjectReference`
de outro módulo para este?" (nunca existe, `CONTRACTS/RULES.md` seção 2), e
"esse tipo específico é injetado fora do módulo?" (só `Dtos/` é, e mesmo
assim só via serialização HTTP, nunca em código).

### 1.2 Nomenclatura — `Request`, `Dto` (Response), `Entity`: cada um só no seu escopo ideal

Quatro nomes aparecem o tempo todo nesta arquitetura e é fácil misturar o
escopo de cada um. A regra é: cada um vive **só** onde a tabela abaixo diz,
nunca aparece fora dali, e nunca é o mesmo tipo C# que outro.

| Sufixo/nome | Onde vive | Papel | Nunca aparece em |
|---|---|---|---|
| `<Recurso>Request` | `Modules/<NomeModulo>/Contracts/Requests/` — parâmetro de ação do `Controller` (`[FromBody]`/query string) | Forma de **entrada** HTTP — o `Controller` traduz `Request` → `Command` (`COMMANDS/RULES.md`) | `Handler`, `Repository`, `Entity`, retorno de qualquer método — só existe como parâmetro de ação |
| `<Recurso>Dto` | `Modules/<NomeModulo>/Contracts/Dtos/` — retorno do `Handler`, servido como corpo de resposta pelo `Controller` (`ToActionResult()`) | Forma de **saída** HTTP — "Response" é só como chamamos esse papel do `Dto` na borda; **não existe uma terceira classe `<Recurso>Response`** nesta arquitetura | `Repository` (retorna `Entity`), qualquer campo do tipo `Entity` |
| `<Recurso>` (sem sufixo "Entity" no nome da classe, ex: `Pedido`) | `Modules/<NomeModulo>/Entities/` | Modelo de domínio — nome do substantivo puro, nunca `PedidoEntity` | `Contracts/` (nem `Requests/`, nem `Dtos/`), `Command`, corpo de resposta HTTP |

- Endpoints de leitura simples (`GET` por rota/query string) normalmente
  **não precisam de `Request`** — o `Controller` recebe o parâmetro
  primitivo direto (`Get(Guid id)`, `CONTROLLER/RULES.md` seção 3) e monta
  o `Command` com ele. `Request` só entra quando o corpo HTTP tem forma
  própria a desserializar (`POST`/`PUT`/`PATCH`).
- Verbo do nome do `Request` acompanha o verbo do `Command` que ele
  alimenta (`COMMANDS/RULES.md` seção 4) — ex: `CreatePedidoRequest` →
  `CreatePedidoCommand`, nunca um verbo diferente entre os dois.

**❌ Errado — `Request` vazando pro Handler, `Entity` com sufixo redundante, sufixo todo maiúsculo:**

```csharp
public record CreatePedidoRequest(Guid ClienteId, List<ItemPedidoInput> Itens);

public class PedidoEntity : AggregateRoot { /* ... */ } // ❌ sufixo "Entity" redundante — já mora em Entities/

public class PedidoHandler
{
    // ❌ Handler recebendo Request direto — deveria receber CreatePedidoCommand
    public async Task<Result<PedidoDTO>> Handle(CreatePedidoRequest request) // ❌ "DTO" todo maiúsculo
    {
        // ...
    }
}
```

**✅ Correto — cada nome só no seu escopo, Command como fronteira entre Request e Handler:**

```csharp
// Contracts/Requests/CreatePedidoRequest.cs — só existe como parâmetro de ação do Controller
public record CreatePedidoRequest(Guid ClienteId, List<ItemPedidoInput> Itens);

// Entities/Pedido.cs — nome do substantivo puro, sem sufixo
public class Pedido : AggregateRoot { /* ... */ }

// Handler/PedidoHandler.cs — recebe Command, nunca Request; devolve Dto, nunca DTO
public class PedidoHandler
{
    public async Task<Result<PedidoDto>> Handle(CreatePedidoCommand command) { /* ... */ }
}
```

**Nota sobre SonarQube (regra de nomenclatura de tipo, ex: `S101`):** o
analisador rejeita nome de tipo terminando em sufixo **todo maiúsculo**
(`PedidoDTO`, `CriarPedidoREQUEST`) — para o Sonar isso não é PascalCase
válido. A convenção desta arquitetura já evita isso por padrão: sempre
`Dto`, `Request`, `Entity` (primeira letra maiúscula, resto minúsculo),
nunca `DTO`, `REQUEST`, `ENTITY`. Se um caso excepcional realmente exigir
manter um sufixo todo maiúsculo (ex: interoperar com um contrato externo
que já usa esse nome), suprimir explicitamente em vez de deixar o
analisador falhar silenciosamente ignorado:

```csharp
[System.Diagnostics.CodeAnalysis.SuppressMessage("Major Code Smell", "S101:Types should be named in PascalCase")]
public class PedidoDTO { } // exceção documentada, não o padrão
```

ou, fora do C#/quando o Sonar roda como scanner externo: comentário
`// NOSONAR` na linha da declaração. Isso é exceção pontual e justificada
— nunca a forma padrão de nomear `Dto`/`Request`/`Entity` neste rulebook.

## 2. Estrutura de pastas

```
Modules/
├── Shared/
│   └── Contracts/
│       ├── I<NomeModulo>.cs              # ex: IVendasModule.cs — fachada síncrona
│       └── IntegrationEvents/
│           └── <EventoNoPassado>Event.cs # ex: PedidoCriadoEvent.cs
└── <NomeModulo>/
    ├── Contracts/
    │   ├── Requests/
    │   │   └── <Verbo><Recurso>Request.cs # ex: CreatePedidoRequest.cs — só parâmetro de ação (seção 1.2)
    │   ├── Dtos/
    │   │   └── <Recurso>Dto.cs          # ex: PedidoDto.cs — só usado pelo próprio Controller
    │   └── Repositories/
    │       └── I<Recurso>Repository.cs  # interface — ver seção 1.1 e REPOSITORIES/RULES.md §2
    └── Repository/
        └── <Recurso>Repository.cs       # implementação concreta — ver REPOSITORIES/RULES.md §2
```

**Consequência prática:** nenhum módulo de negócio tem `ProjectReference`
para outro módulo de negócio — todos referenciam `Shared` (e `Infrastructure`),
nunca uns aos outros diretamente. Isso não é só convenção: sem essa
referência no `.csproj`, o `using` de outro módulo **não compila**
(`ARCHITECTURE-RULES.md` seção 5).

## 3. `I<NomeModulo>` — a fachada síncrona

```csharp
// Modules/Shared/Contracts/IVendasModule.cs
namespace Shared.Contracts;

public interface IVendasModule
{
    Task<bool> ClienteTemPedidoAbertoAsync(Guid clienteId);
}
```

- Expõe só o que outro módulo genuinamente precisa consumir de forma síncrona — não é um espelho de todos os `Handlers` do módulo. Se nenhum outro módulo precisa de uma operação, ela não entra aqui, mesmo que exista um `Handler` equivalente para uso via `Controller`.
- Assinatura de método nunca retorna `Entity` — sempre tipo primitivo, coleção de primitivo, ou `bool`/`void`. (Não retorna `Dto` do módulo publicador — `Dto` fica no próprio módulo, seção 1; se um método de fachada precisar devolver algo estruturado, o tipo de retorno também é definido aqui em `Shared/Contracts`, não importado do módulo.)

**❌ Errado:**

```csharp
public interface IVendasModule
{
    Task<Pedido> ObterPedidoAsync(Guid clienteId); // ❌ Entity de domínio cruzando módulo
}
```

**✅ Correto:**

```csharp
public interface IVendasModule
{
    Task<bool> ClienteTemPedidoAbertoAsync(Guid clienteId); // ✅ primitivo, resposta mínima que o consumidor precisa
}
```
- **Quem implementa:** uma classe concreta dentro de `Services/` do módulo publicador (ex: `Services/VendasModuleFacade.cs`, ver `SERVICES/RULES.md`), registrada no `Install()` do `<NomeModulo>ModuleInstaller` como a implementação de `I<NomeModulo>`. Essa classe concreta é `internal` (`ARCHITECTURE-RULES.md` seção 5.1) — só a interface, pública e em `Shared`, é visível para quem consome.
- Um módulo consumidor **nunca** referencia a classe concreta (nem conseguiria — está em `internal` num assembly que ele nem referencia) — só injeta `I<NomeModulo>`.

## 4. `Dtos` — a forma de resposta do próprio módulo

- Continuam dentro do módulo (`Modules/<NomeModulo>/Contracts/Dtos/`), porque só o `Controller` do próprio módulo os usa — nenhum outro módulo importa esse tipo em código.
- Sempre `record` imutável (nunca classe com setters públicos). Ex: `public record PedidoDto(Guid Id, Guid ClienteId, decimal Total, string Status);`
- Nunca contém um campo do tipo `Entity` de domínio.
- Convenção de mapeamento: método `FromEntity(Entity entity)` na própria classe do `Dto`, chamado pelo `Handler` (`HANDLER/RULES.md` seção 8) — pode ser `internal` (só o `Handler` do próprio módulo chama), enquanto o `Dto` em si continua `public` (é serializado como resposta HTTP). O `Dto` nunca faz o caminho inverso (não tem `ToEntity()`).

**❌ Errado — `Dto` embrulhando a `Entity` em vez de espelhar os campos:**

```csharp
public record PedidoDto(Pedido Pedido); // ❌ serializa o modelo de domínio inteiro pra fora

public class PedidoDtoMutavel // ❌ classe, não record — permite setter público
{
    public Guid Id { get; set; }
    public decimal Total { get; set; }
}
```

**✅ Correto — campos primitivos espelhados, `record` imutável, mapeamento explícito:**

```csharp
public record PedidoDto(Guid Id, Guid ClienteId, decimal Total, string Status)
{
    internal static PedidoDto FromEntity(Pedido pedido) =>
        new(pedido.Id, pedido.ClienteId, pedido.Total, pedido.Status.ToString());
}
```

## 5. `IntegrationEvents` — o que é publicado

```csharp
// Modules/Shared/Contracts/IntegrationEvents/PedidoCriadoEvent.cs
namespace Shared.Contracts.IntegrationEvents;

public record PedidoCriadoEvent(Guid PedidoId, Guid ClienteId, decimal Total) : IntegrationEvent;
```

- Toda classe aqui herda de `IntegrationEvent` (`record` base definido em `Infrastructure/Messaging` — ver `02-INFRASTRUCTURE/MESSAGING/RULES.md` seção 4, motivo de viver lá e não em `Shared/Kernel`). Precisa ser `record` herdando de `record` — `class` não compila (`error CS8864`).
- Nome sempre no particípio passado, descrevendo um fato já ocorrido (`PedidoCriadoEvent`, não `CriarPedidoEvent`) — evento é notificação de algo que aconteceu, nunca um comando disfarçado.
- Sempre `record` imutável, mesmos critérios de tipos permitidos do `Dto` (seção 4).
- Vive em `Shared/Contracts/IntegrationEvents/`, não no módulo publicador: o módulo consumidor (`Consumer`, ver `CONSUMERS/RULES.md`) referencia o tipo do evento diretamente em código para deserializar a mensagem — precisa estar em `Shared` pelo mesmo motivo de `I<NomeModulo>` (seção 1).
- Se o evento carrega dado que o consumidor precisaria buscar de volta no publicador (ex: gêneros de um anime), esse dado entra como campo do próprio evento (event-carried state) — evita o consumidor precisar de uma segunda chamada síncrona de volta ao publicador.

## 6. Compatibilidade — `Dto`, `I<NomeModulo>` e `IntegrationEvent` são contrato, não implementação

Uma vez que um `Dto`, método de `I<NomeModulo>` ou `IntegrationEvent` tem
consumidores, mudar sua forma é uma **mudança de contrato**, não um detalhe
interno:

- Nunca remover ou renomear um campo/método existente sem processo de depreciação (adicionar o novo, manter o antigo por um período, migrar consumidores, só então remover).
- Adicionar um campo novo opcional é seguro e não exige depreciação.
- `Contract tests` (`04-TEST/CONTRACT/RULES.md`) existem exatamente para pegar essas quebras antes de chegarem em produção — todo `Dto`/`I<NomeModulo>`/`IntegrationEvent` publicado precisa ter um teste de contrato cobrindo sua forma.

## 7. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| `Dto`/`IntegrationEvent` com campo do tipo `Entity` | Vaza modelo interno de domínio |
| `I<NomeModulo>` com método retornando `Entity` | Mesma razão — a fachada é uma fronteira, não uma janela para dentro |
| `Dto`/`IntegrationEvent` mutável (classe com setter público) | Consumidor externo poderia alterar um objeto que deveria ser um retrato imutável de um fato/estado |
| Renomear/remover campo de contrato já publicado sem depreciação | Quebra consumidor(es) sem aviso |
| `I<NomeModulo>` ou `IntegrationEvent` dentro do módulo publicador em vez de `Shared/Contracts` | Reintroduz a necessidade de `ProjectReference` direto entre módulos de negócio — anula o isolamento da seção 1 |
| `Dto` (de resposta HTTP) movido para `Shared` sem necessidade | `Dto` só sai do módulo se outro módulo realmente precisar dele em código — mover por padrão infla `Shared` à toa |
| Outro módulo instanciando `<NomeModulo>ModuleFacade` diretamente em vez de injetar `I<NomeModulo>` | Impossível de qualquer forma — a classe concreta é `internal` num assembly não referenciado, mas documentado aqui como reforço da intenção |

## 8. Enforcement

- Contract tests bloqueiam merge se a forma de um `Dto`/`I<NomeModulo>`/`IntegrationEvent` já publicado mudar de forma incompatível.
- Code review bloqueia qualquer tipo `Entity` aparecendo como parâmetro ou retorno dentro de `Contracts/` (módulo ou `Shared`).
- Code review bloqueia qualquer `ProjectReference` de um módulo de negócio para outro módulo de negócio — só `Shared`/`Infrastructure` são permitidos (`ARCHITECTURE-RULES.md` seção 5.2).
