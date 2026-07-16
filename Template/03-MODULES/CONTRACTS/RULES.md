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

### 1.1 Por que `IPedidoRepository` (interface) **não** entra aqui

`Repository` é `public` (`REPOSITORIES/RULES.md` seção 2) — mas isso é uma
consequência do C# (a interface aparece no construtor `public` de `Handler`,
`ARCHITECTURE-RULES.md` seção 5.1), não uma declaração de que ele é parte da
superfície pública do módulo. Ninguém fora do módulo — nenhum outro módulo,
nenhum `Controller` de outro lugar — jamais injeta ou referencia
`IPedidoRepository`. "Contracts" nesta arquitetura significa especificamente
"o que outro módulo/HTTP compila contra" (seção 1 acima); `Repository` nunca
se encaixa nisso, então continua dentro de `Modules/<NomeModulo>/Repositories/`,
não aqui — mesmo com a interface fisicamente separada da implementação em
`Repositories/Contracts/` (pasta **local**, sem relação com este documento
ou com `Shared/Contracts` — seção 1.1 do próprio título já avisa: é o
terceiro lugar chamado "Contracts" nesta arquitetura, cada um com escopo de
visibilidade diferente) enquanto a implementação fica na raiz de
`Repositories/` (`REPOSITORIES/RULES.md` seção 2), essa divisão é só
organização de arquivo, não uma mudança de fronteira arquitetural —
`Repositories/` inteiro continua tão privado ao módulo quanto `Entities/`.

## 2. Estrutura de pastas

```
Modules/
├── Shared/
│   └── Contracts/
│       ├── I<NomeModulo>.cs              # ex: IVendasModule.cs — fachada síncrona
│       └── IntegrationEvents/
│           └── <EventoNoPassado>Event.cs # ex: PedidoCriadoEvent.cs
└── <NomeModulo>/
    └── Contracts/
        └── Dtos/
            └── <Recurso>Dto.cs           # ex: PedidoDto.cs — só usado pelo próprio Controller
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
- **Quem implementa:** uma classe concreta dentro de `Services/` do módulo publicador (ex: `Services/VendasModuleFacade.cs`, ver `SERVICES/RULES.md`), registrada no `Install()` do `<NomeModulo>ModuleInstaller` como a implementação de `I<NomeModulo>`. Essa classe concreta é `internal` (`ARCHITECTURE-RULES.md` seção 5.1) — só a interface, pública e em `Shared`, é visível para quem consome.
- Um módulo consumidor **nunca** referencia a classe concreta (nem conseguiria — está em `internal` num assembly que ele nem referencia) — só injeta `I<NomeModulo>`.

## 4. `Dtos` — a forma de resposta do próprio módulo

- Continuam dentro do módulo (`Modules/<NomeModulo>/Contracts/Dtos/`), porque só o `Controller` do próprio módulo os usa — nenhum outro módulo importa esse tipo em código.
- Sempre `record` imutável (nunca classe com setters públicos). Ex: `public record PedidoDto(Guid Id, Guid ClienteId, decimal Total, string Status);`
- Nunca contém um campo do tipo `Entity` de domínio.
- Convenção de mapeamento: método `FromEntity(Entity entity)` na própria classe do `Dto`, chamado pelo `Handler` (`HANDLER/RULES.md` seção 8) — pode ser `internal` (só o `Handler` do próprio módulo chama), enquanto o `Dto` em si continua `public` (é serializado como resposta HTTP). O `Dto` nunca faz o caminho inverso (não tem `ToEntity()`).

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
